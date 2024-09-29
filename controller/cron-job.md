cron-job 实现了定时任务。cron-job 对象控制的对象是 job。<br>

# 定时如何实现

本质是在 cron-job 的 controller 里面有一个延时队列，通过 heap 保存，有一个定时任务定时从这个 heap 获取到期的 cron-job 对象，扔到
controller 对应的 queue 中，然后有几个 consumer goroutine 负责从这个 queue 中获取 cron-job 并执行。<br>

延迟任务相关的代码如下：

```go
func (q *delayingType[T]) waitingLoop() {
	defer utilruntime.HandleCrash()

	// Make a placeholder channel to use when there are no items in our list
	never := make(<-chan time.Time)

	// Make a timer that expires when the item at the head of the waiting queue is ready
	var nextReadyAtTimer clock.Timer
    
    // 通过 heap 维护所有的延时任务
	waitingForQueue := &waitForPriorityQueue{}
	heap.Init(waitingForQueue)

	waitingEntryByData := map[t]*waitFor{}

	for {
		if q.TypedInterface.ShuttingDown() {
			return
		}

		now := q.clock.Now()

		// 到期的任务加入到 queue 中
		for waitingForQueue.Len() > 0 {
			entry := waitingForQueue.Peek().(*waitFor)
			if entry.readyAt.After(now) {
				break
			}

			entry = heap.Pop(waitingForQueue).(*waitFor)
			q.Add(entry.data.(T))
			delete(waitingEntryByData, entry.data)
		}

		// Set up a wait for the first item's readyAt (if one exists)
		nextReadyAt := never
		if waitingForQueue.Len() > 0 {
			if nextReadyAtTimer != nil {
				nextReadyAtTimer.Stop()
			}
			entry := waitingForQueue.Peek().(*waitFor)
			nextReadyAtTimer = q.clock.NewTimer(entry.readyAt.Sub(now))
			nextReadyAt = nextReadyAtTimer.C()
		}

		select {
        // 这是个兜底的 channel，感觉没必要
		case <-q.heartbeat.C():
		case <-nextReadyAt:
		case waitEntry := <-q.waitingForAddCh: // 延时任务从该 channel 写入
			if waitEntry.readyAt.After(q.clock.Now()) {
				// 未到期的，加入 heap queue
				insert(waitingForQueue, waitingEntryByData, waitEntry)
			} else {
				// 到期的，直接执行
				q.Add(waitEntry.data.(T))
			}

			drained := false
			// 排干 waitingForAddCh 中所有的任务
			for !drained {
				select {
				case waitEntry := <-q.waitingForAddCh:
					if waitEntry.readyAt.After(q.clock.Now()) {
						insert(waitingForQueue, waitingEntryByData, waitEntry)
					} else {
						q.Add(waitEntry.data.(T))
					}
				default:
					drained = true
				}
			}
		}
	}
}
```

# 一些注意点

当 cron-job 再次运行的时候，旧的 job 有可能还在执行中，那么这个时候要不要再次执行 job 呢？这是由参数 `cronJob.Spec.ConcurrencyPolicy` 指定的。
其可能的值如下：

```go
// ConcurrencyPolicy describes how the job will be handled.
// Only one of the following concurrent policies may be specified.
// If none of the following policies is specified, the default one
// is AllowConcurrent.
// +enum
type ConcurrencyPolicy string

const (
	// AllowConcurrent allows CronJobs to run concurrently.
	AllowConcurrent ConcurrencyPolicy = "Allow"

	// ForbidConcurrent forbids concurrent runs, skipping next run if previous
	// hasn't finished yet.
	ForbidConcurrent ConcurrencyPolicy = "Forbid"

	// ReplaceConcurrent cancels currently running job and replaces it with a new one.
	ReplaceConcurrent ConcurrencyPolicy = "Replace"
)
```