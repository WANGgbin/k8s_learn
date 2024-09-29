daemon controller 保证了 node 上总是有对应的 pod 在运行。<br>

关于 daemon controller，我们重点关注两个问题：

- daemon controller 创建的 pod 如何绑定到固定的 node
    
    daemon controller 发现某个 node 上没有对应的 pod 后，就会创建 pod 在该 node 上运行。可是，创建的 pod 完全有可能被调度到其他 node
上，如何让 pod 只能运行在目标 node 上呢？<br>

    这就是 nodeAffinity 的作用，通过在 pod 中设置 nodeAffinity。scheduler 在调度 pod 的时候，就只会将 pod 调度到满足 nodeAffinity
的 node 上。

- 如何在 unscheduled node 上创建 daemon pod 呢？
    
    当节点为 unscheduled 的时候，表示不能在该 node 上调度 pod，可是我们就想在此 node 上创建 daemon pod 怎么办呢？比如 node 的网络插件
就是通过 daemon pod 的方式部署的，但是在成功部署网络插件之前，node 是 unscheduled 的。<br>

    通过 toleration 实现，node 可能会被打上各种各样的 taint(污点)，为了能够让 pod 调度到这些 node 上，pod 需要设置 toleration 字段，表示
能够容忍这些 taint。

# pod

我们看看 daemon controller 创建的 pod 有哪些特殊之处。

```go
// 基于 daemonSet 中的 template 创建 pod
func CreatePodTemplate(template v1.PodTemplateSpec, generation *int64, hash string) v1.PodTemplateSpec {
	newTemplate := *template.DeepCopy()
    
    // 这一步会给 pod 加上各种 
	AddOrUpdateDaemonPodTolerations(&newTemplate.Spec)

	return newTemplate
}
```
```go
// 加入各种各样的 tolerations
func AddOrUpdateDaemonPodTolerations(spec *v1.PodSpec) {
	// DaemonSet pods shouldn't be deleted by NodeController in case of node problems.
	// Add infinite toleration for taint notReady:NoExecute here
	// to survive taint-based eviction enforced by NodeController
	// when node turns not ready.
	v1helper.AddOrUpdateTolerationInPodSpec(spec, &v1.Toleration{
		Key:      v1.TaintNodeNotReady,
		Operator: v1.TolerationOpExists,
		Effect:   v1.TaintEffectNoExecute,
	})

	// DaemonSet pods shouldn't be deleted by NodeController in case of node problems.
	// Add infinite toleration for taint unreachable:NoExecute here
	// to survive taint-based eviction enforced by NodeController
	// when node turns unreachable.
	v1helper.AddOrUpdateTolerationInPodSpec(spec, &v1.Toleration{
		Key:      v1.TaintNodeUnreachable,
		Operator: v1.TolerationOpExists,
		Effect:   v1.TaintEffectNoExecute,
	})

	// According to TaintNodesByCondition feature, all DaemonSet pods should tolerate
	// MemoryPressure, DiskPressure, PIDPressure, Unschedulable and NetworkUnavailable taints.
	v1helper.AddOrUpdateTolerationInPodSpec(spec, &v1.Toleration{
		Key:      v1.TaintNodeDiskPressure,
		Operator: v1.TolerationOpExists,
		Effect:   v1.TaintEffectNoSchedule,
	})

	v1helper.AddOrUpdateTolerationInPodSpec(spec, &v1.Toleration{
		Key:      v1.TaintNodeMemoryPressure,
		Operator: v1.TolerationOpExists,
		Effect:   v1.TaintEffectNoSchedule,
	})

	v1helper.AddOrUpdateTolerationInPodSpec(spec, &v1.Toleration{
		Key:      v1.TaintNodePIDPressure,
		Operator: v1.TolerationOpExists,
		Effect:   v1.TaintEffectNoSchedule,
	})

	v1helper.AddOrUpdateTolerationInPodSpec(spec, &v1.Toleration{
		Key:      v1.TaintNodeUnschedulable,
		Operator: v1.TolerationOpExists,
		Effect:   v1.TaintEffectNoSchedule,
	})

	if spec.HostNetwork {
		v1helper.AddOrUpdateTolerationInPodSpec(spec, &v1.Toleration{
			Key:      v1.TaintNodeNetworkUnavailable,
			Operator: v1.TolerationOpExists,
			Effect:   v1.TaintEffectNoSchedule,
		})
	}
}
```

我们接着看看如何设置 nodeAffinity 的：
```go
	podTemplate.Spec.Affinity = util.ReplaceDaemonSetPodNodeNameNodeAffinity(
					podTemplate.Spec.Affinity, nodesNeedingDaemonPods[ix])
```
```go
func ReplaceDaemonSetPodNodeNameNodeAffinity(affinity *v1.Affinity, nodename string) *v1.Affinity {
	// pod 只能调度到 node.name == nodename 的 node 上
	nodeSelReq := v1.NodeSelectorRequirement{
		Key:      metav1.ObjectNameField,
		Operator: v1.NodeSelectorOpIn,
		Values:   []string{nodename},
	}

	nodeSelector := &v1.NodeSelector{
		NodeSelectorTerms: []v1.NodeSelectorTerm{
			{
				MatchFields: []v1.NodeSelectorRequirement{nodeSelReq},
			},
		},
	}

    return &v1.Affinity{
        NodeAffinity: &v1.NodeAffinity{
            RequiredDuringSchedulingIgnoredDuringExecution: nodeSelector,
    },

	return affinity
}
```