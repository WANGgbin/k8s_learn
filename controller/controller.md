描述 k8s 中关于控制器的内容。<br>

k8s 的控制器是运行在 pod 中的。<br>

几个问题:

- 我们知道 master 节点是可以集群化部署的，如果存在多个 controller 组件，哪个 controller 组件生效呢？
 
    分布式锁，谁抢到锁，说就执行 control 逻辑.

- 如何自定义 controller ?


# controller 选主

基于 etcd 分布式锁选主，谁抢到锁，谁就是 leader.

- 锁的占有者，网络分区了咋办

    controller manager 占锁成功后，会有个异步协程，会定时进行锁续期操作，如果续期失败了，controller manager process 会立刻退出。

- 锁的占有者，异常退出了咋办

    退出时，会尝试释放自己占有的锁。如果异常退出，没有占有锁，其他节点只能等锁过期，然后重新占有锁。  

# controller 工作模型

几个问题：
- informer 是啥
- lister 是啥
- 从 api-server 获取对象的方式是 list&watch，有了 watch 是不是就够了，为什么还要有 list ?
- 一个类型的 controller 对应一个 informer 吗？还是 所有的 controller 公用一个 informer? 
- 不同 controller 如何共享 informer ?

## informer

可以通过注册 event handler 的方式，让 informer 把 delta 数据写入到 work queue 中，然后 sync 可以从 work queue 中
获取数据，执行 sync 逻辑。问题是：对于存量的 unsync 的 api 对象，sync 逻辑 sync 的时机是什么？

我们知道各种各样的 controller 本质就是用来执行 sync 逻辑，从而保证 controller 关注的对象的实际运行状态跟预期的运行状态一致。<br>

因此，controller 需要通过某种方式监控关注对象的变更，进而能够及时执行 sync 操作。那么我们该如何监控某一类 api 对象的变更呢？<br>

这就是 informer 的作用，监控某一类 api 对象的变更，需informer 跟 api 对象 一一对应。要关心该 api 对象变更的 controller 通过给 informer 注册 listener，从而及时感知
此类对象的变更。<br>


### 架构

informer 包括以下几个部分：

- reflector

  通过 ListAndWatch 的方法获取 api 对象的变更，并将对象写入到 delta fifo 中。可以看作是一个生产者。

- delta fifo
  
  是一个 fifo 队列，每个元素对应某个 api 对象的某一类变更事件，包括：Add、Update、Sync、Delete 等。

- local store
  
  informer 会有一个 delta fifo 的消费 goroutine，负责两件事情，将 delta fifo 中的对象 sync 到本地缓存中；将 delta fifo 中的变更事件
  distribute 给所有关心该对象变更的 controller.

  local store 就是此类对象的本地缓存，跟 etcd 中的对象保持最终一致性。

  为什么要缓存 api 对象？为了加速 api 对象的获取提高性能，同时缓解 api-server 的压力。

- event distribute

  将 delta fifo 中的变更，分发给各个 controller.

### reflector

reflector 通过 ListAndWatch 从 api-server 监听对象的变更。关于 watch 的实现原理可以参考 watch.md。<br>

既然可以通过 watch 监控对象的变更，为什么还要使用 list 呢？<br>

因为我们需要首先获取存量数据，即某个时刻所有的对象，这就是 list 的作用。然后基于这个时刻，再通过 watch 的方式获取增量的数据。除此之外，watch 的过程中
可能发生错误，我们会尝试重新 watch，如果还是无法 watch 的话。就需要重新执行 list + watch 逻辑。<br>


#### resync

兜底机制。定时将 store 中的 objects 回灌到 delta fifo 中，重新进行同步。


### listener 实现机制

整体实现思路类似观察者模式，controller 是观察者，delta fifo 的消费者(processor)是被观察者。每个 controller 给 processor 注册一个 listener，
从而将 delta fifo 中的事件传递给 controller. 我们看看 listener 的实现。<br>

```go
// processorListener relays notifications from a sharedProcessor to
// one ResourceEventHandler --- using two goroutines, two unbuffered
// channels, and an unbounded ring buffer.  The `add(notification)`
// function sends the given notification to `addCh`.  One goroutine
// runs `pop()`, which pumps notifications from `addCh` to `nextCh`
// using storage in the ring buffer while `nextCh` is not keeping up.
// Another goroutine runs `run()`, which receives notifications from
// `nextCh` and synchronously invokes the appropriate handler method.
// 
// 上面的注释已经很清楚的阐述了 listener 的实现原理。整个 listener 也是一个 producer/consumer 模型。
// 一个协程从 addCh 读取事件添加到 buffer，并从 buffer 发送到 nextCh
// 另一个协程从 nextCh 读取事件，然后调用 handler 处理事件
type processorListener struct {
    nextCh chan interface{}
    addCh  chan interface{}

    handler ResourceEventHandler


    // pendingNotifications is an unbounded ring buffer that holds all notifications not yet distributed.
    // There is one per listener.
	// 一个 listener 为什么要有一个 buffer 呢？这主要是因为事件的生产速度和消费速度不一致，消费逻辑可能比较慢，
    // 因此需要一个队列缓存这些事件。
    pendingNotifications buffer.RingGrowing
}

// 我们重点关注以下三个方法：add、pop、run
// add delta fifo 的 processor 调用此方法，将事件通知给 listener
func (p *processorListener) add(notification interface{}) {
	p.addCh <- notification
}

// pop 有一个单独的协程执行。
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
		case nextCh <- notification: // 从 buffer 读取事件写入 nextCh
			// Notification dispatched
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil // Disable this select case，注意这种写法。chan 置空，case 分支会被忽略。
			}
		case notificationToAdd, ok := <-p.addCh: // 从 addCh 读取事件存放到 buffer 中。
			if !ok {
				return
			}
			if notification == nil { // No notification to pop (and pendingNotifications is empty)
				// Optimize the case - skip adding to pendingNotifications
				notification = notificationToAdd
				nextCh = p.nextCh
			} else { // There is already a notification waiting to be dispatched
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}

// run 由一个单独的协程实现
func (p *processorListener) run() {
	// this call blocks until the channel is closed.  When a panic happens during the notification
	// we will catch it, **the offending item will be skipped!**, and after a short delay (one second)
	// the next notification will be attempted.  This is usually better than the alternative of never
	// delivering again.
	stopCh := make(chan struct{})
	wait.Until(func() {
		// 从 nextCh 读取事件，然后调用 handler 处理事件
		for next := range p.nextCh {
			switch notification := next.(type) {
			case updateNotification:
				p.handler.OnUpdate(notification.oldObj, notification.newObj)
			case addNotification:
				p.handler.OnAdd(notification.newObj, notification.isInInitialList)
				if notification.isInInitialList {
					p.syncTracker.Finished()
				}
			case deleteNotification:
				p.handler.OnDelete(notification.oldObj)
			default:
				utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
			}
		}
		// the only way to get here is if the p.nextCh is empty and closed
		close(stopCh)
	}, 1*time.Second, stopCh)
}
```

为什么每个 processListener 要实现为生产者、消费者模型呢？

主要是同一个 informer 可以有多个 listener，而不同的 listener 消费速度是不一样的。如果从 informer 收到一个事件，然后 同步调用
所有的 listener(即使并发调用)，则其他 listener 会被最慢的 listener 拖慢处理速度。为了解耦不同 listener，那就每个 listener
搞成一个 producer -> queue -> consumer 的模型。informer 接收到事件后，扔到每个 listener 的内部 queue 中，然后由各个 listener
的 consumer 慢慢消费。

### 编程中队列的使用

可以看到整个 controller 的架构中用到了 队列。我们来扩展下，什么场景下应该使用 controller 呢？

- 多消费者消费速度解耦 

当某个 api 内部要执行比较耗时的操作的时候，就可以考虑使用队列。我们只需要将要处理的任务发送到队列即可，然后在消费者中进行真正的业务逻辑。

比如在 listener 的实现中。我们完全可以不用队列，从 delta fifo 接收到一个对象时，先更新缓存，然后同步执行各个 controller 具体的 sync 逻辑，
但这带来的问题就是如果某个 controller 的 sync 逻辑很慢，整个 delta fifo 的消费速率就会被拖慢。

这里就可以使用队列，异步解耦 delta fifo 的消费逻辑和 controller 的 sync 逻辑。delta fifo 的消费者把对象写入到队列，然后每个 controller
对应一个消费者，消费队列即可。

- 处理并发

如果有很多个场景都需要执行相同的逻辑，那么在这些公用逻辑中就不得不处理并发的问题，要加锁进而保证并发安全。这带来了两个问题：

  - 性能差

    因为涉及到加锁，所以整体性能较差。

  - 代码复杂度高，难维护

    因为涉及到加/解锁，代码复杂度高，不好迭代/维护。

这种场景下，我们也可以考虑使用队列来处理并发的问题。各个场景都是队列的生产者，在生产的时候仍然需要加锁，但这很轻量，因为只需要执行一个生产操作即可。
队列的消费逻辑就很简单，不需要考虑并发的问题。一来降低了代码复杂度，而来提高了程序整体的性能。像旧版 kafka controller 的模型就是此模型，kafka
controller 需要处理各种事件：定时任务的事件、zookeeper watch 事件、来自 cli 的事件等等。如果都同步执行，逻辑会很复杂，涉及很多锁来保证并发安全，
但是通过生产者、消费者模型，整个架构就会简单。

实际上，golang 中的 select 模型就可以理解为是一个多生产者、但消费者的模型，生产者就是多个 case 分支，但消费者就是各个 case 分支的处理逻辑
(任意时刻只有一个 case 分支在跑)，这也是 golang 为什么适合编写中间件、底层组件的原因，select 模型很方便。