描述 k8s 相关知识。

- 整体架构
- 如何搭建一个 k8s 集群
- k8s 与 docker 关系
- 声明式理念的理解
- 插件化理念的理解
- 经常说的分配几G几Core，是如何实现的
- 控制器是什么
- 如何自定义 api 对象 和 控制器
- 调度器如何保证性能的？
- APIService api 对象的实现原理
- K8s 使用 开源实现 CoreDNS 作为自己的 DNS 实现，需要了解 CoreDNS 的原理
- Service 的工作原理
- 集群外部外部如何访问 Service ?
- iptables、ipvs、netfilter 关系？

    - netfilter 内核框架，在数据包处理流程中，设置了 5 个 hook。
    - iptables 在 netfilter 的 hook 中插入了各种规则。table、chain 都是 iptables 的概念
    - ipvs 同样在 netfilter 的 input 和 output hook，注册了一些 dnat 处理逻辑。

- service external 的使用场景？

    一句话：将 k8s 外部的服务暴露给 k8s 集群内部的服务
    - 参考：https://developer.aliyun.com/article/1309553

- 有了容器为什么还要有 pod
- 有了 pv 为什么还要有 pvc? pv 与 pvc 的生命周期是什么样的？
- 竟然通过 kubeadm 可以直接部署 k8s，原理是什么
- 如何容器化分布式应用 ？
- 学习下 k8s 中的 shell 脚本
- kublet 是如何识别 cri 的？
- 127.0.0.1 工作原理？
- 这么复杂的项目，单测是怎么做的？

    - 接口的使用