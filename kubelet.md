描述 k8s 中的 kubelet 组件。<br>

kubelet 的主要职责就是负责从 api-server、local file dir、http 接受操作(创建/更新/删除) Pod 的请求，然后调用 CRI 完成具体的操作。同时需要
不断给 api-server 上报 Pod status。<br>


# 如何从 api-server、local file dir 监控 pod 事件

- api-server

当用户执行 kubectl apply 一个 pod 的更新后，api-server 会将 pod 信息更新到 etcd，kubelet 通过 api-server 在 etcd 注册的 watcher 会监听
到对应的 pod 事件，然后将 pod 事件发送给 api-server，api-server 又将此事件通过 http chunk 发送给 kubelet。具体 watch 的实现可以参考 watch.md<br>

- local file dir

获取本地目录种文件的变更有两种方式：

 - 主动轮询
 - 操作系统主动推送

   一些 linux 操作系统提供了 `SYS_INOTIFY_INIT1` 系统调用，可以监控特定目录的变化，该系统调用的返回是一个 fd，当监控的目录发生变化的时候，就会给此 fd 发送信息。<br>

   SYS_INOTIFY_INIT1 是一个系统调用，用于在 Linux 系统中初始化 inotify 实例。inotify 是 Linux 内核提供的一种文件系统监控机制，它允许应用程序监控文件和目录的变化，例如文件的创建、修改、删除等。
   通过使用 SYS_INOTIFY_INIT1 系统调用，应用程序可以创建一个 inotify 实例，并使用该实例来添加监控项、接收监控事件等。

# 如何及时监控 node 上 pod/container 的变化

即使我们通过 CRI 成功执行了某种 Pod 的操作，比如成功创建了一个 Pod，但这之后，Pod 的状态仍然有可能发生变化，比如某个业务 container 突然 panic 了。
因此，当 pod/container 的状态发生了变化，我们仍然需要重新执行 Sync。<br>

那么，我们该如何及时感知 node 上的 pod/container 变更事件呢？<br>
    
CRI 是基于 GRPC 的，定义了一个 PodEvents 的 Stream API，通过此 API， kubelet 可以源源不断的监听 node 上所有 Pod/Container 的
更新事件。接口定义如下：

```go
	// GetContainerEvents gets container events from the CRI runtime
	// 事件会通过 containerEventsCh 发送给 pleg(pod lifecycle events generator)
	GetContainerEvents(containerEventsCh chan *runtimeapi.ContainerEventResponse, connectionEstablishedCallback func(runtimeapi.RuntimeService_GetContainerEventsClient)) error
```

# 如何给 api-server 上报 pod status

在 podWorker 每次执行 Sync*(SyncPod、SyncTerminatingPod、SyncTerminatedPod) 的时候，都会计算 pod 的 api status，然后通过 http 发送 patch 请求更新 pod status。

# pod 中的 restartPolicy 实现原理

在执行 syncPod 的时候，会基于目标状态、当前状态以及 restartPolicy 计算应该 restart 哪些 container. 代码如下：
```go
// pod 为目标状态， podStatus 为实际状态
func ShouldContainerBeRestarted(container *v1.Container, pod *v1.Pod, podStatus *PodStatus) bool {
	// Once a pod has been marked deleted, it should not be restarted
	if pod.DeletionTimestamp != nil {
		return false
	}
	// 从 podStatus 获取 container 的状态
	status := podStatus.FindContainerStatusByName(container.Name)
	// If the container was never started before, we should start it.
	// NOTE(random-liu): If all historical containers were GC'd, we'll also return true here.
	if status == nil {
		return true
	}
	// Check whether container is running
	if status.State == ContainerStateRunning {
		return false
	}
	// Always restart container in the unknown, or in the created state.
	if status.State == ContainerStateUnknown || status.State == ContainerStateCreated {
		return true
	}
	// Check RestartPolicy for dead container
	// 永不重启
	if pod.Spec.RestartPolicy == v1.RestartPolicyNever {
		return false
	}
	// 只有失败的时候才重启
	if pod.Spec.RestartPolicy == v1.RestartPolicyOnFailure {
		// Check the exit code.
		// 退出码 == 0 表示成功，否则未失败
		if status.ExitCode == 0 {
			return false
		}
	}
	return true
}
```

# 容器活性探测机制

容器是否健康运行，我们不能依赖 CRI 返回的容器状态，而应该由容器本身定义是否健康运行的标准。在 pod 中，我们可以为每个容器定义 livenessProbe
来监测容器的状态。

容器 probe 的方式包括：

 - exec cmd

    在目标容器中执行一个命令。

 - http get

    默认往 host = pod id 发送一个 http 请求。用户可以指定 host、目标端口、path、headers。

 - tcp
 - grpc request

    发送一个 grpc 请求。


那么在 kubelet 中，是如何消费 liveness probe 的结果的呢？

在执行 SyncPod() 的时候，我们需要判断应该拉起哪些容器，删除哪些容器。一些容器状态虽然是 running，但是 liveness probe 检测结果不一定是成功的，
对于这种 container，就应该 kill 掉。代码路径为：pkg/kubelet/kuberuntime/kuberuntime_manager.go:993

其实在 k8s 中，有各种各样的 probe，liveness probe 只是其中一种。

# csi

几个问题：
- kubelet 如何跟 csi 通信的？


# plugin 机制

k8s 为了扩展性考虑，会将一些某方面的能力抽象为插件。比如，对于 volume 的管理抽象为 CSI，将资源分配(GPU 等)能力抽象为 DRA(Dynamic Resource Allocation)。
而 kubelet 会与这些插件交互，这里就涉及到几个问题：

- 怎么知道当前节点有哪些插件
- 怎么注册这些插件

## 插件发现机制

为了能够让 kubelet 发现这些插件，kubelet 规定这些插件需要将自己的 uds(kubelet 与 插件通过 uds 通信)，注册到特定的目录下。

然后 kubelet 会通过 inotify 系统调用监听该目录的变更，当有新的 uds 创建的时候(通过 path 的 stat 即可判断是不是 uds)，便知道有新的插件注册,
随后与该 uds 建立连接，之后进行 grpc 通信。

接下来的问题是，kubelet 怎么知道这个新 plugin 的类型、名字等信息呢？这就要求 plugin 需要实现一个 `GetInfo` 的接口，该接口返回 GetInfo
的相关信息，据此 kubelet 能够知道插件的类型，kubelet 为不同类型的插件实现了不同的 Validate、Register、DeRegister 逻辑。

## 调谐

插件管理也涉及到一个调谐机制，我们要达到的目标就是实际注册的插件要与指定目录下的插件信息一致。


# csi

csi 插件是通过 daemonset 的方式部署在每一个节点上的。通过前面介绍的 plugin 机制，当 csi 插件启动后，kubelet 就能发现并建立 uds 连接。

当 kubelet 接收到一个 pod 时，便会调用 csi 的能力，进行 mount 操作。

