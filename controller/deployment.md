有了 replica_set 为什么还要有 deployment 对象呢？

replica_set 只解决了水平伸缩的问题，没有解决版本变更(升级/降级)的问题。比如，我们要变更应用的版本，通过 replica_set 是没法实现的。

那 deployment 又是如何实现版本变更的特性的呢？**使用新版本创建一个新的 replica_set，将旧的 replica_set 缩容为 0，将新的 replica_set
扩容为目标数量即可**，因此，deployment 是 **replica_set 的 controller**

