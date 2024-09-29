描述 k8s 的整体架构以及各个组件。

# 整体架构


# 组件

## control plane components

控制面组件。

### kube-apiserver

- apiserver 多实例如何部署，负载均衡如何实现


### etcd

k8s 使用 etcd 来存储集群中的数据。

### kube-scheduler

负责 pod 的调度。


### kube-controller-manager


## node

- 新加入节点时，如何向 k8s 注册
- 节点的健康状态如何同步给 k8s，两种方式：api server 轮询、节点主动上报，这两种方式的区别是什么？

    - 节点主动上报优点：
      - 实时性更好
      - master 节点只需要被动接受数据，不需要定时批量主动发送请求，master 节点负载更低
      - master 节点逻辑更简单，只需要被动接受数据即可

    - api server 轮询：
      - 实时性差，在下游节点状态发生变化时，不能及时感知
      - 当集群中节点很多的时候，需要批量给每一个节点发送健康检查请求，server 节点工作负载大
      - server 需要管理什么时候、如何(串行/并行) 发送查询，逻辑更加复杂 


### kubelet

每个节点一个 kubelet 组件，负责接受 podSpec，然后保证 PodSpec 中描述的容器处于运行状态。

### kube-proxy

service 实现的一部分，用来维护节点上的一系列规则，比如：iptables、ipvs。保证集群内/外 可以访问到一组 pod 通过 service 提供的服务。

### cri

即 container runtime interface，用来管理容器。
