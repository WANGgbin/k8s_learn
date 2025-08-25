为了提高系统整体性能以及降低对 api-server 压力，controller-manager 中通过 List + Watch 的方式，将关心的 api 对象存储在缓存中，
这样就可以直接从 cache 中获取 api 对象。

但是有个问题，查 api 对象的时候，可能根据不同的条件去查，怎么提高查询的性能呢？答案是通过**二级索引**.

k8s store 维护了一个 indices 的对象，本身就是个所有二级索引的集合，每个 value 就是个具体的索引 index，而index 又是个 map[string]sets[string]
对象，可以理解是索引值到主键列表的映射，而主键就是每个对象的 id。