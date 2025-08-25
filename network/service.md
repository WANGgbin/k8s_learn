- 研究 nginx-ingress-controller 的实现

# service

service 有什么用？

如果 client 通过 pod ip 来访问一组 pod 这有两个问题：1、pod 的 ip 是会发生变化的；2、如何实现多 pod 访问的负载均衡呢？

service 本质上就是一组 pod 的反向代理，能够将请求自动的代理到对应的 pod，同时还能实现负载均衡。client 通过 service 访问后端服务即可。


# 类别

service 有很多类，包括：
- clusterIP
- nodePort
- externalName

还有其他类型，我们重点关注上面3中类型。其他类型有时间可以再研究。

## clusterIP

k8s 会为 clusterIP 类型的 service 分配一个 vip，具体的分配逻辑如下：
  配置一个 ip range，然后通过 bitmap 的方式从这个 ip range 中分配 ip。这个分配信息也是需要持久化的，k8s 将分配信息存储到 etcd 中，
这样即使 api-server crash 了，分配信息还在。另外还需要注意的是：如果有多个执行流同时访问 etcd 获取分配信息再进行分配，就可能存在并发问题。
k8s 通过乐观锁的方式解决此问题。从 etcd 获取一个版本，然后进行分配，最后写回的时候判断一下版本是否还是之前的版本，如果是更新成功否则进入下一轮重新分配。
而这个版本监测的逻辑，就是通过 etcd 的 transaction 实现的。

有了 vip 后，我们可以在 k8s 集群的任意一台宿主机上都可以访问 service 对应的服务。那么原理又是什么呢？

k8s 集群中每一个 worker 节点都会拉起一个 kube-proxy process，该进程负责监听集群中 service 与 endpoint(简单理解为 pod) 的变化，然后在宿主机上
创建对应的 iptables rule 或者 创建 ipvs virtual server 以及 real server.

这样无论是从宿主机发出的请求还是发送到宿主机的请求，只要目标ip 和 port 跟 vip 以及 service 定义的 port 匹配，就会进行 dnat，同时在 postRoute hook，通过
masquerade 的方式进行 snat(让响应从 pod 路由到当前宿主机，否则会直接发送给 client)，从而将请求转发给实际的 pod。

关于 kube-proxy，我们在 kube-proxy.md 中单独介绍。

## nodePort

有个问题：我们如何在 k8s 集群外部访问 service 呢？

clusterIP 模式是不太行的，因为无法用 vip 直接路由到某个 k8s 宿主机器。而 nodePort 类型的 service 便是用来解决此问题的。

nodePort 指的是使用宿主机的ip+指定端口来代理后端 pod，而不再分配一个 vip。具体的代理方式还是通过 iptables or ipvs。这样只要 client 访问
任意一台宿主机的 ip 即可访问后端服务。

## externalName

有个问题，如何在 k8s 集群 **内部** 访问 k8s 集群外部的服务呢？

首先为什么会有这样的场景。一个例子是，若干个服务开始往 k8s 迁移，如果上游已经迁移到 k8s 但是下游还未迁移，就会涉及到从 k8s 内部访问外部的问题。

这就是 externalName 类型的 service 要解决的问题。externalName 其实就是个外部服务的域名，该 service 创建后，kube-dns 就会为 service 对应的
k8s 内部域名创建一个 cname 到外部域名。这样当 client 通过 k8s内部域名访问 service 的时候，就会解析到外部服务的 ip，进而完成对外部服务的访问。

## load balancer

## 解决什么问题

我们可能会有这样的需求，通过一个固定的 ip 来访问 k8s 中的某个 service。应该怎么实现呢？

## 怎么解决的

通常一些公有云厂商的 k8s 支持此类型的 service。当在云厂商的 k8s 中创建一个类型为 load balancer 的 service 时，内部的控制器会创建一个
外部的四层负载均衡器，并为该负载均衡器分配一个固定的 ip。

那么流量如何从该负载均衡器打到后段 pod 呢？实际上负载均衡器是在 k8s 外部的，所以没法通过 service 的 vip 访问的，访问 k8s 服务的方式与 nodePort
类似，即负载均衡器内部代理的地址是 k8s 集群节点的 ip，流量打到 k8s 某个节点后，后续的流程就与 nodePort 类似。

load balancer 本质上是在 nodePort 基础上封装了一层负载均衡器。

# Ingress

## 解决什么问题

如果用户想要从外部访问 k8s service，之前的方式有以下问题：

- nodePort 类型

  每个 service 在所有节点上都要占用一个端口，资源浪费。而且用户访问某个 service 还需要知道具体的端口号，用户体验差。

- loadBalancer 类型

  除了有着与 nodePort 类似问题，每个 service 都需要建立一个外部的负载均衡器，资源浪费严重。

此外，前面介绍的各种 pod 代理都是 4 层代理，虽然性能好，但是灵活性比较差，我们还是需要一个 7 层代理。

ingress 要做的就是：

- 代理 k8s 内部的 service
- 可以工作在 7 层，提供更丰富的负载均衡功能

## 怎么解决的

k8s 引入 ingress 对象以及 ingress controller 的概念。

ingress 就是一个 api 对象，ingress 描述，对哪些 service 代理，以及代理的 url 以及端口号是什么。

ingress controller 本身就是个 deployment(可以有多种实现，比如 nginx、haproxy)，每个 pod 根据实现会拉起一个实际的 7 层负载均衡进程，比如
nginx controller 就会拉起一个 nginx 进程，然后监听 ingress 对象的变化，发生变化时，更改 nginx 的配置文件。

为了用户能够在 k8s 外部访问 ingress controller pod，ingress controller 对应的 pod 也被一个 service 代理。外部访问的方式，可以选择 nodePort 或者 loadBalancer.
仔细想想，ingress 以及 ingress controller 不就是自定义的 api 对象以及自定义的控制器吗？在 k8s 中，很多问题都可以通过自定义 api 对象以及自定义控制器的方式解决，
本质就是个调谐的过程，之前这些工作由运维人员完成，现在由 k8s 的 controller 自动完成。


最后，我们以下面的 ingress 举例，看看用户在浏览器输入 http://cafe.example.com/tea 后，请求的流动路径：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com # rule 中的 host 必须是一个完整的 fqdn 不能是一个 ip 地址
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

假设 ingress controller 对应的 service 通过 loadBalancer 方式部署，对外暴露一个稳定的 ip。

- 用户请求 dns 解析 cafe.example.com 获取对应的 ip 地址，实际上这里域名解析后的地址就是 ingress controller service 外部负载均衡器暴露的 ip
- 请求到外部负载均衡器
- 负载均衡器将流量转发到 k8s 集群的某个节点
- 内部节点通过 iptables 将请求转发给 ingress controller 的某个 pod
- pod 收到信息后，根据配置发现 /tea 匹配的 service 是 tea-svc，通过内部 dns 获取 tea-svc 对应的 vip
- 然后根据 kube-proxy 配置的 iptables 规则将 vip 转化为某个实际的业务 pod ip
- 流量打到后段 pod
- 之后响应的流程类似，不再赘述。
