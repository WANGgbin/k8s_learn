我们知道在 k8s 集群当中，我们可以通过 `kubeadmin join` 的方式加入很多的 worker node，那么 k8s 是如何监控这些节点的呢？比如当某个节点<br>
异常退出后，k8s 如何感知这个信息？<br>

这实际上依赖两部分：

- kubelet

每个 worker node 上有一个 kubelet，kubelet 会定时的给 api-server 上报 node 信息，每次信息的上报都会刷新 node 中的 heartbeat timestamp。


- nodelifecycle controller

此外，在 controller plane 中还运行这一个 node-life-cycle controller，该 controller 会查看 node 中 heartbeat timestamp 的更新时间，<br>
当超过某个时间后，就标记该 node 的 ready condition 为 unknown，同时将 node 上的 pod 标记为 not ready。