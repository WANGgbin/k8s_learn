# metric


# log

两种思路：

- node 上以 daemon set 方式部署一个收集日志的 log agent(fluentd)

关键点在于 log agent 怎么知道容器输出的日志路径呢？

 - 对于 stdout/stderr 输出的日志，kubelet 会 捕获这些输出，并以固定的路径格式写入到本地磁盘中，格式为：`/var/log/pods/<pod-uid>/<container-name>/<日志文件>`
   这算是一种约定，log agent 只需要从该路径读取日志即可。

 - 如果容器自定义了日志路径，则需要通过挂载暴露到宿主机，然后再让 log agent 读取日志。

- 每个 pod 通过 sidecar 方式部署 log agent 收集日志。

基本上用第一种方式，好处在于对于 pod 没有侵入性，同时只需要一个 log agent，资源占用低。