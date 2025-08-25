描述 k8s 中 dns 相关内容。

这里先说明几个关键点，详细内容，后续可以慢慢补充。

1. dns 是通过 deployment 部署的，其对应的实现为 core-dns，其大体实现思路为：向 api-server 发送请求获取对应的 service/pod 的 ip 地址。

2. dns 可以通过 service 访问。service 是 clusterIP 模式，其 ip 是固定的。

3. kubelet 在创建 pod 的时候，会更改 pod 的 dnsIP，从而使得 pod 能够访问 k8s 集群内部的 dns server。

4. k8s 中的 fqdn 格式如下：

    - clusterIP service

      - service 对应的 fqdn 为：serviceName.namespaceName.svc.cluster.local，对应的 ip 地址为 vip
      - pod 对应的 fqdn 为：podName.namespaceName.pod.cluster.local，对应的 ip 就是 pod 的 ip

    - headless service

        - service 对应的 fqdn 为：serviceName.namespaceName.svc.cluster.local，对应的 ip 为代理的所有 pod 的 ip 列表
        - pod 对应的 fqdn 为：podName.serviceName.namespaceName.svc.cluster.local，对应的 ip 就是 pod 的 ip


# 服务发现

我们来聊聊 k8s 中的服务发现机制。

k8s 中的服务发现是通过 `coreDNS` 实现的。coreDNS 也是通过 service、endpointSlice 等对象的 informer 来监听对象的变更，进而更新 service、
pod 等对应域名对应的 A 记录。业务 pod 中就可以直接根据 fqdn 来访问某个 service，前面已经说过，kubelet 在创建 pod 的时候其 dnsIP 已经被设置为 dns
service 对应的 vip，从而所有的域名解析全部交给 k8s 集群内部的 coreDNS。

# coreDNS 如何访问 api-server

我们借助外部工具部署高可用的 api-server 集群，所有 client 通过一个 stable 的 ip 访问 api-server 集群。关于如何构建高可用的 api-server，
可以参考 api-server.md 中相关内容。

还有一种思路是：通过 k8s 的 service 机制访问 api server。k8s 自动为 api-server 创建了一个 clusterIP service，这个 clusterIP 是固定的，
kubelet 在创建 pod 的时候，通过环境变量，将这个 vip 以及端口注入到环境变量中。这样 pod 就可以直接通过这个 vip:port 来与 api-server 通信了。

