描述 k8s 中 replica_set 相关内容。<br>

replica_set 能够保证集群中有指定数量的 pod 运行。我们大概看看其实现原理。<br>

首先，replica_set listen 了两个 informer: pod_informer、replica_set_informer。当集群中 pod 和 replica_set 对象变更
的时候，replica_set 会将对应的 replica_set 对象扔到一个 queue 中。<br>

replica_set 在启动的时候，会拉起若干个异步 goroutine，每个 goroutine 负责从 queue 获取 replica_set 然后进行 control 操作。
大体逻辑如下：
- 获取当前集群中所有与 replica_set 匹配的处于 active 中的 pod
- 基于这些 pod 以及 replica_set 期望的 pod 数量，来计算需要创建/删除多少 pod
- 根据匹配到的 active 的 pod 计算 replica_set 的 status 并更新

这里有个问题，假设 replica_set 期望 3 个 pod，集群中一个 pod 都没有，那么经过一轮计算后，replica_set 就会创建 3 个 pod。但是，这里
创建 pod 仅仅指的是在 etcd 中创建了 pod 对象，并不代表 pod 已经成功运行在某个 node 上了。<br>

如果在 informer 收到 pod 创建成功事件前，我们又从 queue 中获取到该 replica_set 对象(比如：informer 的 sync 操作将 replica_set 对象回灌到 queue)，这一轮
计算过程中，集群中还是没有一个 active(创建成功即使还没真正运行也算 active) 的 pod，那岂不是又会创建 3 个 pod。<br>

那么 k8s 如何解决此问题的呢？<br>

这就是 `Expectations` 要解决的问题，比如要创建 3 个 pod，那 Expectations.Add = 3。当 replica_set listen 到对应的 pod 的创建成功事件时，就会将 Expectations.Add - 1
，直到 Expectations.Add == 0，即表示上一轮期望的目标已经完成。这样在下一轮处理的时候，才会重新判断是否创建/删除 pod。<br>

当然，有可能我们期望的 pod 创建事件可能很久都收不到，怎么办？ 给 Expectations 设置个过期时间就可以了。当 Expectations 过期的时候，也会重新判断创建/删除 pod，也即新建 Expectation。
那如果在新创建了 pod 后，原来的 pod 创建事件收到了，那岂不是集群中的 pod 数量多了？那没关系，下次同步的过程中，会把多余的 pod 删除掉。<br>

我们看看 replicaSet 的 sync 流程：
```go
// replicaSet 的真正同步逻辑。
func (rsc *ReplicaSetController) syncReplicaSet(ctx context.Context, key string) error {
	// 通过 lister 从本地 replica_set store 中获取对象
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
    
    // 这一点很重要，避免重复创建/删除。
	// 只有 expectation fulfilled 或者上次 expectation 超时，才需要再次同步。
	rsNeedsSync := rsc.expectations.SatisfiedExpectations(logger, key)
	selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
	
    // 获取集群中所有的 pod
	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
	
	// 过滤所有 active 的 pod
	filteredPods := controller.FilterActivePods(logger, allPods)
    
    // 收养匹配的孤儿 pod 或者 释放不再匹配的 pod
	filteredPods, err = rsc.claimPods(ctx, rs, selector, filteredPods)

	// 执行真正的 sync 操作，创建/删除 pod
	if rsNeedsSync && rs.DeletionTimestamp == nil {
		manageReplicasErr = rsc.manageReplicas(ctx, filteredPods, rs)
	}

	// 因为 rs 是指向 cache 的，是只读的，如果要修改，就必须先 deepCopy()
	rs = rs.DeepCopy()
	newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)

	// 更新 replicaSet 的 status 信息
    // k8s 中对象都有一个 status 参数，用来描述对象的运行时信息
	updatedRS, err := updateReplicaSetStatus(logger, rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
	
    return
}
```

我们接着看看 claimPods 的流程，其底层调用 ``.
```go
// ClaimObject tries to take ownership of an object for this controller.
//
// It will reconcile the following:
//   - Adopt orphans if the match function returns true.
//   - Release owned objects if the match function returns false.
func (m *BaseControllerRefManager) ClaimObject(ctx context.Context, obj metav1.Object, match func(metav1.Object) bool, adopt, release func(context.Context, metav1.Object) error) (bool, error) {
	controllerRef := metav1.GetControllerOfNoCopy(obj)
	if controllerRef != nil {
		// pod 的 controller 是其他对象，直接返回
		if controllerRef.UID != m.Controller.GetUID() {
			// Owned by someone else. Ignore.
			return false, nil
		}
		if match(obj) {
			// We already own it and the selector matches.
			// Return true (successfully claimed) before checking deletion timestamp.
			// We're still allowed to claim things we already own while being deleted
			// because doing so requires taking no actions.
			return true, nil
		}
		// Owned by us but selector doesn't match.
		// 虽然 pod 的 owner 还是当前 replicaSet 对象，但是 label 不再匹配，则释放 pod 对象，
        // 所谓的释放实际上就是删除 pod 的 ownReference 属性。
		release(ctx, obj)
		// Successfully released.
		return false, nil
	}

	// pod 没有 controllerRef，即是个 orphan(孤儿), 如果 match，则需要收养，即给 pod 的 ownReference 添加当前 replicaSet 对象
	// Selector matches. Try to adopt.
	 adopt(ctx, obj)
	// Successfully adopted.
	return true, nil
}
```

最后我们看看 replicaSet 是如何判断添加/删除 pod 的。

```go
func (rsc *ReplicaSetController) manageReplicas(ctx context.Context, filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
	diff := len(filteredPods) - int(*(rs.Spec.Replicas))
	// 小于 0，表示需要创建
	if diff < 0 {
		diff *= -1
		
		// 这里很关键，加入 diff 个 Creation(Add) 到 Expectation 中
		rsc.expectations.ExpectCreations(logger, rsKey, diff)
		
        // 给 api-server 发送请求创建 pod 对象，注意成功创建并不意味者 pod 真正运行
		successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
			// 这里的创建实际上只是在 etcd 中创建了一个 pod 对象
			err := rsc.podControl.CreatePods(ctx, rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
			return err
		})

		// Any skipped pods that we never attempted to start shouldn't be expected.
		// The skipped pods will be retried later. The next controller resync will
		// retry the slow start process.
	    // 上面英文注释已经说的很明白了，不再赘述。	
		if skippedPods := diff - successfulCreations; skippedPods > 0 {
			for i := 0; i < skippedPods; i++ {
				// Decrement the expected number of creates because the informer won't observe this pod
				rsc.expectations.CreationObserved(logger, rsKey)
			}
		}
		return err
		
    // 多了，需要删除多余的 pod
	} else if diff > 0 {


		podsToDelete := getPodsToDelete(filteredPods, relatedPods, diff)
        
        // Expectations 中加入 len(podsToDelete) 个 delete
		rsc.expectations.ExpectDeletions(logger, rsKey, getPodKeys(podsToDelete))
        
        // 并发场景下的错误处理，倒是可以学习学习。通过 channel 收集错误。
		errCh := make(chan error, diff)
		var wg sync.WaitGroup
		wg.Add(diff)
		for _, pod := range podsToDelete {
			go func(targetPod *v1.Pod) {
				defer wg.Done()
				if err := rsc.podControl.DeletePod(ctx, rs.Namespace, targetPod.Name, rs); err != nil {
					// Decrement the expected number of deletes because the informer won't observe this deletion
					podKey := controller.PodKey(targetPod)
					rsc.expectations.DeletionObserved(logger, rsKey, podKey)
					if !apierrors.IsNotFound(err) {
						errCh <- err
					}
				}
			}(pod)
		}
		wg.Wait()

		// 可以学习下并发场景下，通过 chan 传递错误的方式。
		// 如果指向获取第一个错误，就可以使用这种方式。
		select {
		case err := <-errCh:
			// all errors have been reported before and they're likely to be the same, so we'll only return the first one we hit.
			if err != nil {
				return err
			}
		default:
		}
	}

	return nil
}
```