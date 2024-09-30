configMap 以及 secret 是类似的，不过 secret 是加密的。类似应用的配置信息，我们就可以搞个 configMap 对象，然后在 pod 的 volumes 中<br>
声明该 configMap，然后在 container mount 该 configMap。那么在 container 中即可访问 configMap 内容。<br>

那么 configMap 的工作原理是什么呢？<br>

实际上，kubelet 会 informer 当前节点上 pod 关注的 configMap 对象，然后为每个 configMap 创建若干个文件，每个文件名以及内容跟 configMap<br>
中的 data(key/value 结构) 对应。然后容器启动前，mount 对应的 path，容器即可访问到对应的内容。<br>

secret 对象与 configMap 对象工作方式是很类似的，这里不再赘述。
