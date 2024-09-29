# 什么是有状态服务
    
k8s 将有状态服务抽象为两种类型：

- 服务多个实例之间有依赖关系

    deployment 中 pod 之间是对等的关系，在遇到 pod 存在依赖/先后关系的时候，deployment 就不适用。stateful_set 如何解决服务实例之间
依赖关系的呢？<br>

    stateful_set 严格按照固定的顺序启动 pod，每个 pod 的 name 记录了该 pod 在 stateful_set 中的顺序，格式为`statefule_set.name + ordinal`。
只有序号较小的 pod 成功运行后，才会继续拉起后面的 pod。<br>

    此外，为了能够让 client 以固定的网络标识(fqdn)访问 stateful_set 中的实例，stateful_set 中的所有 pod 都被一个 service 代理，同时每个 pod
的 hostname 固定为 pod.name。这样 pod 的 fqdn 固定为 hostname.service_name.namespace.svc.cluster。即使 pod 发生变更，client 仍然可以
以固定的网络标识访问 stateful_set 中的 pod。

- 服务跟特定的存储绑定

  有些服务实例运行过程中会持久化一些信息，显然，当 pod 重启的时候，pod 仍然能够继续访问之前存储的实例。典型的应用就是 mysql。这也算是有状态服务。那么
stateful_set 如何实现的呢？<br>

  在 stateful_set 的 spec 中，可以通过设置 pvcTemplate 的方式设置该 stateful_set 下 pod 需要的 pvc 信息，在 stateful_set 创建 pod 的时候，
就会根据 pvcTemplate 给每个 pod 创建对应的 pvc 对应，并将 pvc 跟 pod 绑定(添加到 pod 的 volumes 声明中)。需要注意的是，即使 pod 退出，其关联
的 pvc 对象仍然存在，当 stateful_set 重新创建 pod 的时候，就会沿用之前的 pvc 对象，从而能够访问之前存储的数据。

# pod

我们来看看 stateful_set 创建的 pod 有哪些特殊点：
```go
func newVersionedStatefulSetPod(currentSet, updateSet *apps.StatefulSet, currentRevision, updateRevision string, ordinal int) *v1.Pod {
	// 创建一个 pod，ordinal 表示此 pod 在 stateful_set 中的序号
	pod := newStatefulSetPod(updateSet, ordinal)
	// 通过设置 pod 的 label，追踪 pod 关联的 revision，
    // 这里的 revision 就是 stateful_set 对应的 controllerRevision 对象的 name，格式为 stateful_set.name-hash
	setPodRevision(pod, updateRevision)
	return pod
}
```

newStatefulSetPod 流程如下：
```go
func newStatefulSetPod(set *apps.StatefulSet, ordinal int) *v1.Pod {
	// 根据 stateful_set.spec.template 初始化一个 pod 对象并设置 pod 的 controllerRef
	pod, _ := controller.GetPodFromTemplate(&set.Spec.Template, set, metav1.NewControllerRef(set, controllerKind))
	// 名字的格式就是 set.name-ordinal
	pod.Name = getPodName(set, ordinal)
	// 设置网络标识相关内容
	initIdentity(set, pod)
	// 设置 volumes 相关内容
	updateStorage(set, pod)
	return pod
}
```

podName 格式如下：
```go
func getPodName(set *apps.StatefulSet, ordinal int) string {
	return fmt.Sprintf("%s-%d", set.Name, ordinal)
}
```

我们接着看看如何设置网络标识的：
```go
func updateIdentity(set *apps.StatefulSet, pod *v1.Pod) {
	ordinal := getOrdinal(pod)
	pod.Name = getPodName(set, ordinal)
	pod.Namespace = set.Namespace
	if pod.Labels == nil {
		pod.Labels = make(map[string]string)
	}
	// 设置 pod 相关 label
	pod.Labels[apps.StatefulSetPodNameLabel] = pod.Name
	if utilfeature.DefaultFeatureGate.Enabled(features.PodIndexLabel) {
		pod.Labels[apps.PodIndexLabel] = strconv.Itoa(ordinal)
	}
}

func initIdentity(set *apps.StatefulSet, pod *v1.Pod) {
    updateIdentity(set, pod)
    // Set these immutable fields only on initial Pod creation, not updates.
	// 这一步很重要，正式因为设置了 hostName 以及 subdomain，才可以通过固定的网络标识访问此 pod
    pod.Spec.Hostname = pod.Name
    pod.Spec.Subdomain = set.Spec.ServiceName
}
```

我们接着看看如何设置 pod 的 volumes 字段的：
```go
func updateStorage(set *apps.StatefulSet, pod *v1.Pod) {
	// pod 已有的 volumes
	currentVolumes := pod.Spec.Volumes
	// 从 stateful_set 获取的 pvcs
	claims := getPersistentVolumeClaims(set, pod)
	newVolumes := make([]v1.Volume, 0, len(claims))
	// 根据每个 pvcs 生成对应的 volume
	for name, claim := range claims {
		newVolumes = append(newVolumes, v1.Volume{
			Name: name,
			VolumeSource: v1.VolumeSource{
				PersistentVolumeClaim: &v1.PersistentVolumeClaimVolumeSource{
					ClaimName: claim.Name,
					ReadOnly: false,
				},
			},
		})
	}
	for i := range currentVolumes {
		if _, ok := claims[currentVolumes[i].Name]; !ok {
			newVolumes = append(newVolumes, currentVolumes[i])
		}
	}
	// 将 volumes 添加到 pod 中
	pod.Spec.Volumes = newVolumes
}
```

在设置了 pod 的 volumes 之后，在创建 pod 的时候，也是先创建对应的 pvc，最后再创建 pod 对象。至于 pvc 相关的内容，我们后续单独介绍，简单来说
pvc 需要跟 pv 对象绑定，具体绑定的流程也是通过 controller 的方式实现的。

# 扩/缩容

扩容按照序号由小到达的方式创建 pod，缩容按照序号由大到小的方式删除 pod。