描述 k8s 中的 watch 是如何实现的。

我们这里以 kubelet watch pod 的变化，来描述 k8s 中 watch 的实现原理。<br>

整个调用链路是：kubelet -> api-server -> etcd。其中 kubelet 与 api-server 是 http 请求，
api-sever -> etcd 是 grpc 请求。<br>

# kubelet -> api-server

http 如何实现 watch 呢？我们知道 watch response 是需要连续不断的发送给客户端的，可是 http 不是
获取到整个 resp 后再将 resp 序列化后发送给客户端的吗？<br>

实际上这里用到的是 http chunk 特性，http chunk 允许服务端将响应通过一个个 chunk 的方式发送给客户端，
http chunk 的大概格式如下：

```text
length\r\n
data\r\n
```

那怎么标记 resp 的结束呢？发送一个长度为 0 的 chunk 即可。<br>

另一个问题是服务端什么时候结束 watch 的处理呢？假如很长一段时间都没有 event 发生，如果不结束 watch 的处理，
就会一直占用底层的 tcp 连接。<br>

k8s 的解决办法是，通过超时的方式实现。对于这种长请求，服务端有默认的超时时间，当然客户端也可以通过 param 的
方式指定超时时间。一旦超时，api-server 就会 cancel 到 etcd 的 watcher，同时结束 handle 并发送一个空
chunk 标记本次请求处理完成。

# api-server -> etcd

我们知道 etcd 是通过 grpc 的方式对外提供服务的，而 grpc 的 stream 特性本身就是支持 watch 的。client 
stream 可以不断的从客户端接受请求，server stream 可以不断给服务端发送响应。<br>

watcher 在 etcd 中是通过一个单独的 service 实现的，对应一个 Watch 方法，该方法既是 client stream 又
是 server stream。client stream 可以不断的接受 create/cancel Watch Request，server stream 不断
发送 Watch Events 发送给 api-server。<br>

同样，这个 watcher 什么时候结束呢？前面已经说过，当 api-server 中的超时时间到达的时候，api-server 就会
发送 cancel watch request 给 etcd 从而删除对应的 watcher。


# 其他场景

k8s 中有很多的 watch 场景，需要 watch 各种 resource 的变化，从而触发各种行为。整体流程给 kubelet 监控
pod 的变化是类似的。都是通过 http chunk 跟 api-server 通信，api-server 又跟 etcd 通信。