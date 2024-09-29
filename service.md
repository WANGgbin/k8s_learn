- 研究 nginx-ingress-controller 的实现

# service

service 有什么用？<br>

如果 client 通过 pod ip 来访问一组 pod 这有两个问题：1、pod 的 ip 是会发生变化的；2、如何实现多 pod 访问的负载均衡呢？<br>

service 本质上就是一组 pod 的反向代理，能够将请求自动的代理到对应的 pod，同时还能实现负载均衡。client 通过 service 访问后端服务即可。


# 类别

service 有很多类，包括：
- clusterIP
- nodePort
- externalName

还有其他类型，我们重点关注上面3中类型。其他类型有时间可以再研究。

## clusterIP

k8s 会为 clusterIP 类型的 service 分配一个 vip，具体的分配逻辑如下：<br>
  配置一个 ip range，然后通过 bitmap 的方式从这个 ip range 中分配 ip。这个分配信息也是需要持久化的，k8s 将分配信息存储到 etcd 中，<br>
这样即使 api-server crash 了，分配信息还在。另外还需要注意的是：如果有多个执行流同时访问 etcd 获取分配信息再进行分配，就可能存在并发问题。<br>
k8s 通过乐观锁的方式解决此问题。从 etcd 获取一个版本，然后进行分配，最后写回的时候判断一下版本是否还是之前的版本，如果是更新成功否则进入下一轮重新分配。<br>
而这个版本监测的逻辑，就是通过 etcd 的 transaction 实现的。<br>

有了 vip 后，我们可以在 k8s 集群的任意一台宿主机上都可以访问 service 对应的服务。那么原理又是什么呢？<br>

k8s 集群中每一个 worker 节点都会拉起一个 kube-proxy process，该进程负责监听集群中 service 与 endpoint(简单理解为 pod) 的变化，然后在宿主机上<br>
创建对应的 iptables rule 或者 创建 ipvs virtual server 以及 real server.<br>

这样无论是从宿主机发出的请求还是发送到宿主机的请求，这样目标ip 和 port 跟 vip 以及 service 定义的 port 匹配，就会进行 dnat，同时在 postRoute hook，通过<br>
masquerade 的方式进行 snat(让响应从 pod 路由到当前宿主机，否则会直接发送给 client)，从而将请求转发给实际的 pod。<br>

关于 kube-proxy，我们在 kube-proxy.md 中单独介绍。

## nodePort

有个问题：我们如何在 k8s 集群外部访问 service 呢？<br>

clusterIP 模式是不太行的，因为无法用 vip 直接路由到某个 k8s 宿主机器。而 nodePort 类型的 service 便是用来解决此问题的。<br>

nodePort 指的是使用宿主机的ip+指定端口来代理后端 pod，而不再分配一个 vip。具体的代理方式还是通过 iptables or ipvs。这样只要 client 访问<br>
任意一台宿主机的 ip 即可访问后端服务。

## externalName

有个问题，如何在 k8s 集群 **内部** 访问 k8s 集群外部的服务呢？<br>

首先为什么会有这样的场景。一个例子是，若干个服务开始往 k8s 迁移，如果上游已经迁移到 k8s 但是下游还未迁移，就会涉及到从 k8s 内部访问外部的问题。<br>

这就是 externalName 类型的 service 要解决的问题。externalName 其实就是个外部服务的域名，该 service 创建后，kube-dns 就会为 service 对应的<br>
k8s 内部域名创建一个 cname 到外部域名。这样当 client 通过 k8s内部域名访问 service 的时候，就会解析到外部服务的 ip，进而完成对外部服务的访问。

# 7 层代理

前面介绍的 service 实现，本质是个 4 层代理。一个问题是我们需要为每个 service 对象都要创建对应的 iptables/ipvs。有没有什么方法可以不用为每个 service
都配置负载均衡。<br>

有，这就是 ingress 对象的概念。ingress 对象用来描述将什么样的请求(fqdn + path) 发送给那个 service.这里我们取极客时间中的一个例子：

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

ingress 的思路是创建一个 7 层的反向代理，根据 fqdn + path 将请求代理到 pod。具体的实现原理为：

- 实现一个 ingress controller

  该 controller 负责监听 ingress 对象、service、pod 对象的变化，创建对应的 pod 实例并更改 pod 的配置.比如 nginx ingress controller 就需要<br>
创建 nginx 实例并更改对应的配置，从而可以将请求发送给对应的后端 pod。

- 如何访问 ingress controller 创建的 pod

  这些 pod 还是通过 nodeport service 的方式由 client 访问。这里有个问题，比如 client 通过 http://cafe.example.com/tea 发起请求，这些请求怎么发送
到这个 nodeport service 呢？可以通过覆盖 dns 解析结果的方式，强行让 cafe.example.com 指向 nodeport service. 比如，curl --resolve cafe.example.com:targetPort:targetIP 