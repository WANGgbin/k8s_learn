# 解决了什么问题

k8s 持久化存储的实现，依赖调用外部的存储服务完成存储的分配、mount、umount 能力。怎么在 k8s 中接入各种不同的存储分配服务呢？

这就是 csi 解决的问题，将底层的细节抽象为一个 csi 实现，k8s 核心逻辑只需要根 csi 交互即可，这样后续如果需要更改 pv 的实现，
只需要介入另一种 csi 实现即可，k8s 不需要该代码、也不需要重新部署，实现所谓热插拔的能力，扩展性更好、更加灵活。

# 如何解决的

csi(container storage interface) 为 k8s 提供存储分配、attach、deattach、mount、umount 能力。为了更 csi 交互，k8s 将存储相关的核心
逻辑封装为若干个的独立的模块(进程)，因为这些进程与 csi 由频繁的调用，为了降低开销，通过 uds 通信，因此通常部署在一个 pod 中，一个 k8s 集群中
只有这一个 pod。

这些独立的模块，主要包括：

- external provisioner

    本质是 pvc 控制器的逻辑，当监听到 pvc 声明后，调用 storage class 指定的 csi 插件，完成 pv 创建。

- external attacher

    有的 存储创建后，还需要将存储 attach 到 pod 对应的节点上。当然这一步不要求必须在 pod 所在节点上执行，所以抽象为一个独立组件部署。


除此之外，对于分配的 pv，还需要挂载到 pod 所在的节点，**而这一步必须在 pod 所在节点进行**，所以对于 csi 还需要通过 daemon set 方式部署
到每一个节点上，这样每个节点上的 kubelet 就可以与 csi 通信完成 mount 操作。

综上，csi 部署包括：
- stateful set

    整个集群中通过 stateful set 部署一个 pod，该 pod 中包括 csi 以及若干个 k8s 存储相关的组件。
    之所以使用 stateful set，是因为 stateful set 能保证只有旧的 pod 退出，才会创建新的 pod，这种机制能保证，任何时候，集群有且只有这一个 pod。

- daemon set

    为每个节点部署一个 csi 组件。


# k8s 持久化存储总结

最后，我们总结下 k8s 中持久化存储创建的流程：

- 用户创建 storage class 指定 csi 插件
- 用户创建 pvc 对象并指定 storage class
- external provisioner 控制器监听到 pvc 对象后，与同一个 pod 中的 csi 进行 grpc 调用，分配存储并创建 pv 对象。
- k8s 内部控制器 volumeController 将 pvc 与 pv 绑定。
- 用户 pod 中指定 pvc 对象。
- volumeController 判断 pv 是否与 pod 待运行的节点 attach，如果没有则创建一个 VolumeAttachment 对象。
- external attacher 控制器监听到 VolumeAttachment 对象，与 csi 通信，进行 attach 操作。
- 当 kubelet 接收到 pod 时，与同节点上的 csi 进行通信，将 volume 挂载到宿主机目录。

通过上述步骤，一个持久化存储就完成了创建和挂载。