描述 k8s 中调度相关的内容。

# 整体架构

schedule 的作用就是将 pod 调度到某个 node 上。整个架构包括的模块如下：

- informer

    为了能够监听到集群中 pod/node 信息的变化，scheduler 会创建 podInformer 和 nodeInformer。关于 informer 的内容，可以参考 controller/controller.md。

- cache

    scheduler 会维护一个调度的上下文信息，这就是 cache。其定义如下：

```go
type cacheImpl struct {
	// 后面会介绍是 scheduler 提高调度效率的一种方式。
	assumedPods sets.Set[string]
	// 所有的 pod 上下文
	podStates map[string]*podState
	
    // node 调度相关信息
	nodes     map[string]*nodeInfoListItem
	// headNode points to the most recently updated NodeInfo in "nodes". It is the
	// head of the linked list.
	
    // 之所以维护一个列表，主要是方便获取自上次调度以来变更的 node 信息。
    // 每个 nodeInfo 都有一个 generation 的字段，即逻辑时钟, 每次
    // 执行调度的时候，都会基于当前 cache 生成一个 snapshot，后续的调度
    // 都是基于此 snapshot 的。
	headNode *nodeInfoListItem
}

type nodeInfoListItem struct {
    info *framework.NodeInfo
    next *nodeInfoListItem
    prev *nodeInfoListItem
}

// NodeInfo is node level aggregated information.
// NodeInfo 聚合了 node 维度跟调度相关的上下文
type NodeInfo struct {
    // Overall node information.
    node *v1.Node

    // Pods running on the node.
    Pods []*PodInfo

    // The subset of pods with affinity.
    PodsWithAffinity []*PodInfo

    // The subset of pods with required anti-affinity.
    PodsWithRequiredAntiAffinity []*PodInfo

    // Ports allocated on the node.
    UsedPorts HostPortInfo

    // Total requested resources of all pods on this node. This includes assumed
    // pods, which scheduler has sent for binding, but may not be scheduled yet.
    Requested *Resource
    // Total requested resources of all pods on this node with a minimum value
    // applied to each container's CPU and memory requests. This does not reflect
    // the actual resource requests for this node, but is used to avoid scheduling
    // many zero-request pods onto one node.
    NonZeroRequested *Resource
    // We store allocatedResources (which is Node.Status.Allocatable.*) explicitly
    // as int64, to avoid conversions and accessing map.
    Allocatable *Resource


    // Whenever NodeInfo changes, generation is bumped.
    // This is used to avoid cloning it if the object didn't change.
    Generation int64
}
```

信息 cache 化也是提高调度性能的方法之一。这样相关的信息无须通过网络获取，直接从本地获取即可。既然使用了 cache，一个要考虑的问题就是 cache 信息
与 etcd 中信息的一致性。实际上，通过 informer 即可实现最终一致性。

既然 informer 已经 cache 了对应的 api 对象，为什么还要单独搞一个 cache 呢？主要是因为 informer cache 的 api 对象无法直接用于调度，比如我们想知道
某个 node 上已经有哪些 pod 了，这个信息是无法从 informer 获取的。因此单独搞了一个 scheduler 专用的 cache。

- schedule queue

  另一个重要的模块就是 schedule queue。实际上 schedule queue 不是一个队列，而是包括好几个调度相关的数据结构，其定义如下：

```go
// PriorityQueue implements a scheduling queue.
// The head of PriorityQueue is the highest priority pending pod. This structure
// has two sub queues and a additional data structure, namely: activeQ,
// backoffQ and unschedulablePods.
//   - activeQ holds pods that are being considered for scheduling.
//   - backoffQ holds pods that moved from unschedulablePods and will move to
//     activeQ when their backoff periods complete.
//   - unschedulablePods holds pods that were already attempted for scheduling and
//     are currently determined to be unschedulable.
type PriorityQueue struct {
	// pod initial backoff duration.
	podInitialBackoffDuration time.Duration
	// pod maximum backoff duration.
	podMaxBackoffDuration time.Duration
	// the maximum time a pod can stay in the unschedulablePods.
	podMaxInUnschedulablePodsDuration time.Duration

	// inFlightPods holds the UID of all pods which have been popped out for which Done
	// hasn't been called yet - in other words, all pods that are currently being
	// processed (being scheduled, in permit, or in the binding cycle).
	// 表示正在调度中的 pod 列表，value 对应 inFlightEvents 中的 entry
	inFlightPods map[types.UID]*list.Element

	// 该字段存在的意义是为了追踪某个 pod 调度过程中都有哪些 clusterEvent 发生，
    // 从而在 pod 不可调度的时候，重放这些 clusterEvent，进而判断将 pod 放入
    // 到哪个队列中。
	// 那么 inFlightEvents 是如何追踪某个 pod 开始调度之后发生的事件呢？在 pod
    // 开始调度的时候插入一各 pod 类型的 entry，之后发生 clusterEvent 的时候
    // 插入 clusterEvent 类型的 entry。需要注意的是，当 pod 不可调度时，会重放
    // pod entry 之后所有的 clusterEvent entry，直到决定 pod 的归属。
	inFlightEvents *list.List

	// activeQ is heap structure that scheduler actively looks at to find pods to
	// schedule. Head of heap is the highest priority pod.
	// 需要注意的是：activeQ 是个 heap，heap 的堆顶元素是最高优先级的 pod。
	activeQ *heap.Heap
	// podBackoffQ 是对 pod 的一个惩罚，当 pod 变得可能可调度的时候，我们并不会立即将 pod 添加到
    // activeQ 中，而是会先放入 podBackoffQ 中，pod 的 attempt 次数越多，backoff 的时间就越久，
    // 直到过期，才会重新放入 activeQ 中。
	podBackoffQ *heap.Heap
	// unschedulablePods holds pods that have been tried and determined unschedulable.
	unschedulablePods *UnschedulablePods

}
```

可以看到最重要的就是三个集合：activeQ、podBackoffQ、unschedulablePods。我们来看看 pod 是如何在这三个容器中流动的。

- activeQ

  - 当 scheduler 通过 informer 接收到一个 unassigned pod 的 add、update 事件的时候，就会将 pod 扔到 activeQ 中，等待被调度。
  - schedule queue 有两个兜底的定时任务，一个负责将 backoffQ 中过期的 pod 仍到 activeQ 中重新调度，另一个会扫描 unschedulablePods
    中过期的 pod，然后生成一个 UnschedulableTimeout clusterEvent，由各个插件决定将 pod 插入到 backOffQ 或者 activeQ 中。
  - 当一些 clusterEvent 发生的时候，都会尝试将 unschedulablePods 中相关的 pod 放入到 activeQ 或者 backoffQ 中。

- podBackoffQ

  - 当某个 clusterEvent(包括过期事件) 发生的时候，就会根据插件注册的逻辑把 pod 从 unschedulablePods 中添加到 activeQ 或者 backoffQ 中。

- unschedulablePods

  - 当某个 pod 不可调度(包括调度失败)的时候，就会存放到 unschedulablePods 中。


## clusterEvent

怎么理解 clusterEvent 呢？我们知道一个 pod 能否成功调度依赖各种各样的条件，比如如果待调度 pod 设置了 nodeAffinity 属性 或者 antiNodeAffinity 属性，
那么当新增一个 node 的时候，该 pod 就可能变得可调度，又或者 pod 可能跟其他 pod 有 affinity 关系，这样当一个 pod 成功调度后，当前 pod 也可能变得可调度。

scheduler 将这些事件抽象为 clusterEvent，每个 plugin 都可以给自己关心的 clusterEvent 设置 hintFunc，这样当某个 event 发生的时候，就可以调用 pod.UnschedulablePlugins
注册的 hintFunc 来决定将 pod 添加到 activeQ 中还是 backoffQ 中，又或者继续待在 unschedulablePods 容器中。

我们看看这个决定 pod 走向的函数：
```go
func (p *PriorityQueue) movePodsToActiveOrBackoffQueue(logger klog.Logger, podInfoList []*framework.QueuedPodInfo, event framework.ClusterEvent, oldObj, newObj interface{}) {
	// 当前 framework 中是否有插件关心此事件
	if !p.isEventOfInterest(logger, event) {
		return
	}

	activated := false
	// 处理每一个 pod
	for _, pInfo := range podInfoList {
		// schedulingHint 决定了 pod 的归宿
		schedulingHint := p.isPodWorthRequeuing(logger, pInfo, event, oldObj, newObj)
		if schedulingHint == queueSkip {
			// QueueingHintFn determined that this Pod isn't worth putting to activeQ or backoffQ by this event.
			continue
		}
        // 将 pod 从 unschedulablePods 中移除 
		p.unschedulablePods.delete(pInfo.Pod, pInfo.Gated)
		// 添加到 activeQ 或者 backoffQ 中
		queue := p.requeuePodViaQueueingHint(logger, pInfo, schedulingHint, event.Label)
		if queue == activeQ {
			activated = true
		}
	}
    
    // 发生 clusterEvent 后，如果有存在的 inFlightPods，还需要将 clusterEvent 添加到 inFlightEvents 中
	if p.isSchedulingQueueHintEnabled && len(p.inFlightPods) != 0 {
		p.inFlightEvents.PushBack(&clusterEvent{
			event:  event,
			oldObj: oldObj,
			newObj: newObj,
		})
	}

	// 如果将某个 pod 添加到 activeQ 中了，通过条件变量的方式通知调度协程开始调度。
	if activated {
		p.cond.Broadcast()
	}
}
```

可以看到核心的逻辑在 `isPodWorthRequeuing` 中，我们看看该函数的实现：
```go
func (p *PriorityQueue) isPodWorthRequeuing(logger klog.Logger, pInfo *framework.QueuedPodInfo, event framework.ClusterEvent, oldObj, newObj interface{}) queueingStrategy {
	// pod 的 unschedulablePlugins 表明了 pod 不可调度的原因，因此只有这些 plugin 表明在 event 下，pod 变得可能调度，才会将 pod 从 unscheduablePods 中移除。
	// 可以暂时不用关  pendingPlugins.
	rejectorPlugins := pInfo.UnschedulablePlugins.Union(pInfo.PendingPlugins)
    
    // 通配事件，无论如何重新调度，比如 unscheduablePods 中的 timeout 事件就是个 wildCard 类型的事件。
	if event.IsWildCard() {
		return queueAfterBackoff
	}
    
    // 获取 schedulerName 对应的 hintFuncs，如果没有，重新调度
	hintMap, ok := p.queueingHintMap[pInfo.Pod.Spec.SchedulerName]
	if !ok {
		return queueAfterBackoff
	}

	pod := pInfo.Pod
	queueStrategy := queueSkip
	for eventToMatch, hintfns := range hintMap {
		// 事件不匹配，continue
		if !eventToMatch.Match(event) {
			continue
		}

		for _, hintfn := range hintfns {
			// hintfn 如果不是 rejectorPlugins 中某个 plugin 关心的，直接过滤。
			if !rejectorPlugins.Has(hintfn.PluginName) {
				// skip if it's not hintfn from rejectorPlugins.
				continue
			}
            
            // 执行 hint
			hint, err := hintfn.QueueingHintFn(logger, pod, oldObj, newObj)
			if err != nil {
				// 出错了，兜底继续调度
				hint = framework.Queue
			}
			if hint == framework.QueueSkip {
				continue
			}

			if pInfo.PendingPlugins.Len() == 0 {
				// We can return immediately because no Pending plugins, which only can make queueImmediately, registered in this Pod,
				// and queueAfterBackoff is the second highest priority.
				return queueAfterBackoff
			}

			// We can't return immediately because there are some Pending plugins registered in this Pod.
			// We need to check if those plugins return Queue or not and if they do, we return queueImmediately.
			queueStrategy = queueAfterBackoff
		}
	}

	return queueStrategy
}
```

# 调度流程

这一部分是整个 scheduler 最核心的部分，我们看看具体的调度流程是啥样的。

scheduler 的调度协程会不断的从 activeQ 中获取 pod 进行调度，如果 activeQ 中无可调度的 pod 的时候，调度协程就会阻塞在一个条件变量上，当有可调度
的 pod 的时候，其他协程通过此条件变量激活调度线程。调度每个 pod 的逻辑如下：

```go
func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	// 每次调度之前，scheduler 都会基于当前的 cache 信息生成一份 nodeInfo 的快照信息，后续的调度都是基于这个快照信息进行的。
	if err := sched.Cache.UpdateSnapshot(klog.FromContext(ctx), sched.nodeInfoSnapshot); err != nil {
		return result, err
	}

	if sched.nodeInfoSnapshot.NumNodes() == 0 {
		return result, ErrNoNodesAvailable
	}

	// 这一步寻找所有可能的 node，称为：predicate 阶段
	// feasible 可能的;可行的
	// 为了尽可能提高这一步的效率，通过并发的方式将 nodes 分发到若干个 goroutine(默认 20)
	feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
	if err != nil {
		return result, err
	}

	if len(feasibleNodes) == 0 {
		return result, &framework.FitError{
			Pod:         pod,
			NumAllNodes: sched.nodeInfoSnapshot.NumNodes(),
			Diagnosis:   diagnosis,
		}
	}

	// When only one node after predicate, just use it.
	if len(feasibleNodes) == 1 {
		return ScheduleResult{
			SuggestedHost:  feasibleNodes[0].Node().Name,
			EvaluatedNodes: diagnosis.EvaluatedNodes,
			FeasibleNodes:  1,
		}, nil
	}

	// 这一步给每个 node 进行打分，为了尽可能提高打分的效率，这一步也是通过并发的方式执行的，将待打分的 nodes 
    // 分发到若干个(默认 20) goroutine 中。
	priorityList, err := prioritizeNodes(ctx, sched.Extenders, fwk, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}
    
    // 根据一定算法从所有最高分的 nodes 中选择一个 node
	host, _, err := selectHost(priorityList, numberOfHighestScoredNodesToReport)

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: diagnosis.EvaluatedNodes,
		FeasibleNodes:  len(feasibleNodes),
	}, err
}
```

## 调度快照信息

在每调度一个 pod 之前，scheduler 都会生成一个 nodeInfoSnapshot，后续所有的调度都是基于这个上下文进行的。有几个问题：

- 为什么要生成快照

  如果调度过程中直接访问 cache，因为还会有 cache 的更新操作，这就导致需要加锁，从而导致调度的效率较慢。而通过快照的方式，调度逻辑只需要从快照
获取信息即可，无须从 cache 获取信息，从而避免了调度逻辑的加锁，让调度更快。

  其实很多场景我们都可以看到这种通过快照的方式解耦读写的问题。比如 mysql 的 mvcc 就是通过快照的方式解耦读写，从而提高读写并发度。

- 如何生成快照

  要更新快照，有两种方式。一种就是每次都基于 cache 信息重建快照信息，另一种就是每次将 cache 中自上一次快照依赖的更新信息应用到快照信息。

  显然第一种方式性能比较差，scheduler 使用了第二种方式。要使用第二种方式就需要一个逻辑时钟用来判断新/旧。scheduler 给 cache 中的每个 node
和快照引入了一个 generation 的概念，每当 node 有更新的时候，就会将 node 信息置于 cache node 链表的头部，同时 node.generation += 1。当
更新快照的时候，就会从 cache node 链表头部开始应用所有 generation > snapshot.generation 的 node 信息，同时更新 snapshot 的 generation，
从而实现快照的增量更新。

- 调度期间集群发生的事件如何处理

  使用了快照后，整个调度过程看到的是一个静态的信息，在调度期间可能发生一些影像调度结果的集群事件。scheduler 又是如何处理这些事件的呢？

  这实际上就是前面讨论过的 inFlightEvents 的作用，将调度某个 pod 期间所有的集群事件都存放到一个单链表中，当基于快照信息不可调度的时候，会重放
这些事件，如果这些事件会影响调度的结果，就将 pod 重新入队到 backoffQ 或者 activeQ 中。

# 如何优化调度性能

调度性能是调度器的核心指标之一。我们看看 scheduler 都做了哪些努力来提高调度性能：

- 集群信息 cache 化

  通过 cache 集群调度相关的信息，避免从 api-server 查询，大大提高集群性能。通过 informer 机制保证了 cache 信息跟集群实际信息的最终一致性。

- cache 信息快照化

  每次调度的时候，生成一个快照，基于快照读取调度上下文信息。如果从 cache 读取就必然涉及加锁，从而影响调度性能。

- 并发 & 无锁

  无论是 predicate 阶段还是 priority 阶段，scheduler 都会搞好几个协程并发的执行，从而提高调度性能。此外，还通过无锁的方式进一步提高调度的
性能。

  关于无锁部分，我们来看看 score 阶段，是怎么实现协程之间的无锁化的:

```go
func (f *frameworkImpl) RunScorePlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodes []*framework.NodeInfo) (ns []framework.NodePluginScores, status *framework.Status) {
    // 先初始化 map，后续协程所有对该 map 的更新操作，更新的只是 map 的 value，这样就没有并发问题。	
	pluginToNodeScores := make(map[string]framework.NodeScoreList, numPlugins)
	for _, pl := range f.scorePlugins {
		pluginToNodeScores[pl.Name()] = make(framework.NodeScoreList, len(nodes))
	}

	if len(plugins) > 0 {
		// Run Score method for each node in parallel.
		f.Parallelizer().Until(ctx, len(nodes), func(index int) {
			// 每个协程直接根据 index 从 nodes 读取 nodeInfo，避免了并发问题
			nodeName := nodes[index].Node().Name

			for _, pl := range plugins {
				s, status := f.runScorePlugin(ctx, pl, state, pod, nodeName)
				// 错误处理也是，通过非阻塞写 channel 方式，避免加锁
                if !status.IsSuccess() {
                    err := fmt.Errorf("plugin %q failed with: %w", pl.Name(), status.AsError())
                    errCh.SendErrorWithCancel(err, cancel)
                    return
                }
				// 看这里，每个协程最终通过 index 来更新 map value 中的元素，避免了并发问题。
				pluginToNodeScores[pl.Name()][index] = framework.NodeScore{
					Name:  nodeName,
					Score: s,
				}
			}
		}, metrics.Score)
	}
	
  if err := errCh.ReceiveError(); err != nil {
      return nil, framework.AsStatus(fmt.Errorf("running Score plugins: %w", err))
  }

	return allNodePluginScores, nil
}
```

从上面的实现，我们可以学习到，通过 index 方式读/写 slice 可以实现无锁化，同时，对于错误处理，可以通过非阻塞写 channle 实现无锁化。

# 插件化设计

最后我们看一看，scheduler 中整个调度框架以及插件化是如何设计的。

## framework

为了能够让用户自定义调度逻辑，整个 framework 是基于 plugin 构建的。类似于操作系统内核的 netfilter，用户可以在协议栈的特定 point 定义 hook，
framework 也在各个 point 定义了 plugin，用户只需要实现特定的 plugin 即可自定义调度逻辑。

在 k8s 中，每个 pod 可以指定不同的调度 framework，即系统中可以同时生效多个 framework。framework 定义如下：

```go
type frameworkImpl struct {
	// 整个调度流程中需要使用的各种各样的 plugin
	preEnqueuePlugins    []framework.PreEnqueuePlugin
	enqueueExtensions    []framework.EnqueueExtensions
	queueSortPlugins     []framework.QueueSortPlugin
	preFilterPlugins     []framework.PreFilterPlugin
	filterPlugins        []framework.FilterPlugin
	postFilterPlugins    []framework.PostFilterPlugin
	preScorePlugins      []framework.PreScorePlugin
	scorePlugins         []framework.ScorePlugin
	reservePlugins       []framework.ReservePlugin
	preBindPlugins       []framework.PreBindPlugin
	bindPlugins          []framework.BindPlugin
	postBindPlugins      []framework.PostBindPlugin
	permitPlugins        []framework.PermitPlugin
    
    // 通过并行的方式尽可能提高调度的性能
	parallelizer parallelize.Parallelizer
}
```

## plugin

framework 中不同 point 的 plugin 是不尽相同的，包括 preFilter、filter、postFilter 等。

- 注册

  一般我们都是在系统启动的时候注册 plugin。之所以注册 plugin，是因为我们需要知道系统中都有哪些生效的 plugin，进而可以通过用户配置
获取对应的 plugin。

  但是，在启动的时候，初始化 plugin 的一些必要的信息我们可能是无法获取到的，只有在运行的时候才可以确定 这些信息，进而才能初始化 plugin。
那我们应该如何注册这些 plugin 呢？

  scheduler 的解决办法就是启动的时候不注册 plugin，而是注册 plugin 对应的 new func，然后在运行过程中调用 new func 生成对应的 plugin。
具体可以参考下面 framework 初始化源码介绍。

- 上下文传递

  在整个调度允许过程中，后续的 plugin 可能需要使用到前面 plugin 产生的数据，那么 plugin 之间如何进行数据传递呢？

  为此，scheduler 引入了 `CycleState` 对象，可以理解为整个 framework 的数据总线，各个 plugin 都可以读/写这个总线，我们看看这个对象的定义：

```go
// CycleState provides a mechanism for plugins to store and retrieve arbitrary data.
// StateData stored by one plugin can be read, altered, or deleted by another plugin.
// CycleState does not provide any data protection, as all plugins are assumed to be
// trusted.
// CycleState 类似一个数据总线，可以学习下数据总线的设计。
type CycleState struct {
	// 存储 plugin 维度的数据, value 是个 interface
	storage sync.Map
	
	// 其他一些 framework 维度的数据
	// SkipFilterPlugins are plugins that will be skipped in the Filter extension point.
	SkipFilterPlugins sets.Set[string]
	// SkipScorePlugins are plugins that will be skipped in the Score extension point.
	SkipScorePlugins sets.Set[string]
}
```

我们在业务开发中也可以学习这种设计，如果某个业务场景处理流程比较长，我们就可以抽象成多个阶段，各个阶段之间就可以通过 cycleState 的方式进行数据交换。
一些公用的数据，可以置位单独的字段，而一些特定的字段就可以通过 map 的方式传递，这样可以将这个特定的字段跟不关心它的业务场景解耦。

## 如何指定 plugin

用户在启动 scheduler 的时候，只需要在配置文件中指定各个 point 的 plugin name 即可使 plugin 生效，同时还可以指定特定 plugin 的参数。如下：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: '%s'
profiles: # 定义各个阶段的 plugin
- plugins:
    preEnqueue:
      enabled:
      - name: foo
    reserve:
      enabled:
      - name: foo
      - name: bar
      disabled:
      - name: VolumeBinding
    preBind:
      enabled:
      - name: foo
      disabled:
      - name: VolumeBinding
  pluginConfig: # 定义各个 plugin 用到的参数
  - name: InterPodAffinity
    args:
      hardPodAffinityWeight: 2
  - name: foo
    args:
      bar: baz
```

最后我们通过源代码看看每个 framework 的初始化流程：

- scheduler 启动

```go
// New returns a Scheduler
func New(ctx context.Context,
	client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	opts ...Option) (*Scheduler, error) {
    
    // 这里的 registry 就是每个 plugin 对应的 new func 或者 factory
	registry := frameworkplugins.NewInTreeRegistry()
	if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
		return nil, err
	}


    // 基于 registry 和用户指定的 cfg 初始化 plugin
	profiles, err := profile.NewMap(ctx, options.profiles, registry, recorderFactory,
		frameworkruntime.WithComponentConfigVersion(options.componentConfigVersion),
		frameworkruntime.WithClientSet(client),
		frameworkruntime.WithKubeConfig(options.kubeConfig),
		frameworkruntime.WithInformerFactory(informerFactory),
		frameworkruntime.WithSnapshotSharedLister(snapshot),
		frameworkruntime.WithCaptureProfile(frameworkruntime.CaptureProfile(options.frameworkCapturer)),
		frameworkruntime.WithParallelism(int(options.parallelism)),
		frameworkruntime.WithExtenders(extenders),
		frameworkruntime.WithMetricsRecorder(metricsRecorder),
	)

	return sched, nil
}
```

- 注册 plugin factory

```go
func NewInTreeRegistry() runtime.Registry {
	// 注册 plugin factory，因为在启动的时候，只有 factory 是可以确定的。
	registry := runtime.Registry{
		dynamicresources.Name:                runtime.FactoryAdapter(fts, dynamicresources.New),
		imagelocality.Name:                   imagelocality.New,
		tainttoleration.Name:                 tainttoleration.New,
		nodename.Name:                        nodename.New,
		nodeports.Name:                       nodeports.New,
		nodeaffinity.Name:                    nodeaffinity.New,
		podtopologyspread.Name:               runtime.FactoryAdapter(fts, podtopologyspread.New),
		nodeunschedulable.Name:               nodeunschedulable.New,
		noderesources.Name:                   runtime.FactoryAdapter(fts, noderesources.NewFit),
		noderesources.BalancedAllocationName: runtime.FactoryAdapter(fts, noderesources.NewBalancedAllocation),
		volumebinding.Name:                   runtime.FactoryAdapter(fts, volumebinding.New),
		volumerestrictions.Name:              runtime.FactoryAdapter(fts, volumerestrictions.New),
		volumezone.Name:                      volumezone.New,
		nodevolumelimits.CSIName:             runtime.FactoryAdapter(fts, nodevolumelimits.NewCSI),
		nodevolumelimits.EBSName:             runtime.FactoryAdapter(fts, nodevolumelimits.NewEBS),
		nodevolumelimits.GCEPDName:           runtime.FactoryAdapter(fts, nodevolumelimits.NewGCEPD),
		nodevolumelimits.AzureDiskName:       runtime.FactoryAdapter(fts, nodevolumelimits.NewAzureDisk),
		nodevolumelimits.CinderName:          runtime.FactoryAdapter(fts, nodevolumelimits.NewCinder),
		interpodaffinity.Name:                interpodaffinity.New,
		queuesort.Name:                       queuesort.New,
		defaultbinder.Name:                   defaultbinder.New,
		defaultpreemption.Name:               runtime.FactoryAdapter(fts, defaultpreemption.New),
		schedulinggates.Name:                 schedulinggates.New,
	}

	return registry
}
```

- 构造并注册 plugin

profile.NewMap 中最终调用 NewFramework 完成 Framework 的初始化。

```go
// NewFramework 基于 Registry 和 用户指定的 profile config 来初始化 framework
func NewFramework(ctx context.Context, r Registry, profile *config.KubeSchedulerProfile, opts ...Option) (framework.Framework, error) {
	f := &frameworkImpl{
		registry:             r,
		snapshotSharedLister: options.snapshotSharedLister,
		scorePluginWeight:    make(map[string]int),
		waitingPods:          newWaitingPodsMap(),
		clientSet:            options.clientSet,
		kubeConfig:           options.kubeConfig,
		eventRecorder:        options.eventRecorder,
		informerFactory:      options.informerFactory,
		metricsRecorder:      options.metricsRecorder,
		extenders:            options.extenders,
		PodNominator:         options.podNominator,
		parallelizer:         options.parallelizer,
		logger:               logger,
	}

	// get needed plugins from config
	pg := f.pluginsNeeded(profile.Plugins)


	f.pluginsMap = make(map[string]framework.Plugin)
	// 最核心的逻辑在这里，通过遍历每个 factory，构造对应的 plugin 并注册到 framework 中
	for name, factory := range r {
		// initialize only needed plugins.
		if !pg.Has(name) {
			continue
		}

		// 从配置文件中获取 args
		args := pluginConfig[name]
		if args != nil {
			outputProfile.PluginConfig = append(outputProfile.PluginConfig, config.PluginConfig{
				Name: name,
				Args: args,
			})
		}
		// 这里获取对应的 plugin
		p, err := factory(ctx, args, f)
		if err != nil {
			return nil, fmt.Errorf("initializing plugin %q: %w", name, err)
		}
		// 注册到 framework 中
		f.pluginsMap[name] = p

	}
	return f, nil
}
```