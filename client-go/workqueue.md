# WorkQueue



### 1. FIFO队列

#### 1.1 结构

```go
type Type struct {
	queue []t      // slice结构，实际存储元素的地方，保证元素有序
	dirty set      // set结构，保证去重
	processing set // set结构，标记正在被处理的元素，保证了一个元素只被处理一次
	cond *sync.Cond
	shuttingDown bool
	metrics queueMetrics
	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.Clock
}

type empty struct{}
type t     interface{}
type set   map[t]empty  // set结构实际是个map,利用key 保证唯一性
```

#### 1.2 高并发保障

FIFO队列如何保证高并发下，一个元素被添加多次，但同一时刻只会处理一次？

![image-20200823102420667](https://raw.githubusercontent.com/attlee-wang/myimage/master/image/image-20200823102420667.png)

-   结构保证

    dirty和processing字段都为set结构，本质是通过HashMap实现，特性就是保证元素唯一，但不保证无序。所以插入到dirty和processing中的元素绝对没有重复的元素。（当dirty和processing有字段有1时，1就插入不进来。）特别是processing，保证了同一时间，一个元素只有一个goroutine在处理。

-   逻辑保证

    当processing有1，dirty没有1时，dirty 顺利插入1。当processing把1处理完后，如果dirty有1，就将1添加到queue尾部，继续按queue顺序处理。
    
    ```go
    // 源码位置：k8s.io\client-go\util\workqueue\queue.go
    func (q *Type) Get() (item interface{}, shutdown bool) {
    	q.cond.L.Lock()
    	defer q.cond.L.Unlock()
    	for len(q.queue) == 0 && !q.shuttingDown {
    		q.cond.Wait()
    	}
    	if len(q.queue) == 0 {
    		// We must be shutting down.
    		return nil, true
    	}
    
        // 每次只取queue的第一个元素
    	item, q.queue = q.queue[0], q.queue[1:]
    
    	q.metrics.get(item)
    
        // 标记正在处理队列
    	q.processing.insert(item)
        // 从dirty中删除正在处理的元素
    	q.dirty.delete(item)
    
    	return item, false
    }
    
    // Add marks item as needing processing.
    func (q *Type) Add(item interface{}) {
    	q.cond.L.Lock()
    	defer q.cond.L.Unlock()
    	if q.shuttingDown {
    		return
    	}
        // 保证唯一性，dirty已经有了，就不在添加
    	if q.dirty.has(item) {
    		return
    	}
    
    	q.metrics.add(item)
    
        // 否则插入
    	q.dirty.insert(item)
        // 正在处理队列正在处理该元素，一样返回
    	if q.processing.has(item) {
    		return
    	}
    
        // 否则才添加入queue队列
    	q.queue = append(q.queue, item)
    	q.cond.Signal()
    }
    
    func (q *Type) Done(item interface{}) {
    	q.cond.L.Lock()
    	defer q.cond.L.Unlock()
    
    	q.metrics.done(item)
    
        // 处理完成，将元素从processing中删除
    	q.processing.delete(item)
        
        // dirty还有该元素，添加入queue队列尾部，等待按序的处理
    	if q.dirty.has(item) {
    		q.queue = append(q.queue, item)
    		q.cond.Signal()
    	}
    }
    ```

### 2. 延迟队列

#### 2.1 接口

延时队列的接口如下：

```go
// DelayingInterface is an Interface that can Add an item at a later time. 
// This makes it easier to requeue items after failures without ending up in a hot-loop.
type DelayingInterface interface {
	Interface //FIFO队列
	// AddAfter adds an item to the workqueue after the indicated duration has passed
	AddAfter(item interface{}, duration time.Duration)//延迟函数，延迟 duration 时间在将元素入队
}
```

可以看出，延迟队列是基于FIFO队列接口封装，在原有功能上增加了AddAfter方法。

看下AddAfter的具体实现：

```
// 源码位置：k8s.io\client-go\util\workqueue\delaying_queue.go
// AddAfter adds the given item to the work queue after the given delay
func (q *delayingType) AddAfter(item interface{}, duration time.Duration) {
	...
	// immediately add things with no delay
	if duration <= 0 { // 如果延迟时间小于等于0,则将元素插入到queue中
		q.Add(item)
		return
	}

	select {
	case <-q.stopCh:
	// unblock if ShutDown() is called
	// 在当前时间增加duration时间，构造waitFor类型放入q.waitingForAddCh中, 即item放入queue的时间为q.clock.Now().Add(duration)
	case q.waitingForAddCh <- &waitFor{data: item, readyAt: q.clock.Now().Add(duration)}:
	}
}
```

即延迟队列原理是：延迟一段时间后再将元素插入FIFO队列。

那么为什么需要延迟队列？

>   结构上有句解释：This makes it easier to requeue items after failures without ending up in a hot-loop. 即避免在失败之后元素重新入队，引起热循环。

元素插入FIFO队列后续是如何处理的？

#### 2.2 运行原理

延时队列的具体结构如下：

```go
// 源码位置：k8s.io\client-go\util\workqueue\delaying_queue.go
// delayingType wraps an Interface and provides delayed re-enquing
type delayingType struct {
	Interface
	// clock tracks time for delayed firing
	clock clock.Clock
	// stopCh lets us signal a shutdown to the waiting loop
	stopCh chan struct{}
	// stopOnce guarantees we only signal shutdown a single time
	stopOnce sync.Once
	// heartbeat ensures we wait no more than maxWait before firing
	heartbeat clock.Ticker
	// waitingForAddCh is a buffered channel that feeds waitingForAdd
    waitingForAddCh chan *waitFor // 默认大小1000，大于等于1000延迟队列会阻塞，后台通过goroutine  waitLoop()函数 消费chanel
	// metrics counts the number of retries
	metrics retryMetrics
}
```

看下延迟队列的waitLoop()函数的具体实现：

```go
// 源码位置：k8s.io\client-go\util\workqueue\delaying_queue.go
// waitingLoop runs until the workqueue is shutdown and keeps a check on the list of items to be added.
func (q *delayingType) waitingLoop() {
	...
    // 建立了一个waitFor结构的优先级队列并初始化
	waitingForQueue := &waitForPriorityQueue{}
	heap.Init(waitingForQueue)

	waitingEntryByData := map[t]*waitFor{}

	for {
		...
		now := q.clock.Now()
		// Add ready entries
		for waitingForQueue.Len() > 0 {
			entry := waitingForQueue.Peek().(*waitFor)
			if entry.readyAt.After(now) {// 时间还没有到
				break
			}

            // 时间已经到了，插入FIFO队列，并把该元素从waitingEntryByData中删除
			entry = heap.Pop(waitingForQueue).(*waitFor)
			q.Add(entry.data)
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
			nextReadyAt = nextReadyAtTimer.C()// 创建了该元素就绪的时钟周期
		}

		select {
		case <-q.stopCh:
			return

		case <-q.heartbeat.C():
			// continue the loop, which will add ready items

		case <-nextReadyAt: //有元素就绪了
			// continue the loop, which will add ready items

		case waitEntry := <-q.waitingForAddCh:
			if waitEntry.readyAt.After(q.clock.Now()) { // 收到新的元素，时间没到就插入优先级队列
				insert(waitingForQueue, waitingEntryByData, waitEntry)//会引发堆调整
			} else {
				q.Add(waitEntry.data) // 时间已经到了，插入FIFO队列
			}

			drained := false
			for !drained { // 连续处理收到的新元素，
				select {
				case waitEntry := <-q.waitingForAddCh:
					if waitEntry.readyAt.After(q.clock.Now()) {
						insert(waitingForQueue, waitingEntryByData, waitEntry)
					} else {
						q.Add(waitEntry.data)
					}
				default:
					drained = true
				}
			}
		}
	}
}

// insert adds the entry to the priority queue, or updates the readyAt if it already exists in the queue
func insert(q *waitForPriorityQueue, knownEntries map[t]*waitFor, entry *waitFor) {
	// if the entry already exists, update the time only if it would cause the item to be queued sooner
	existing, exists := knownEntries[entry.data]
	if exists {
		if existing.readyAt.After(entry.readyAt) {
			existing.readyAt = entry.readyAt
			heap.Fix(q, existing.index) //堆调整
		}

		return
	}

	heap.Push(q, entry)
	knownEntries[entry.data] = entry
}
```

那么延迟队列的运行原理图如下：

<img src="https://raw.githubusercontent.com/attlee-wang/myimage/master/image/image-20200823103339876.png" alt="image-20200823103339876" style="zoom:50%;" />

​		将元素1放入waitingForAddCh字段中，通过waitingLoop函数消费元素数据。当元素的延迟时间不大于当前时间时，将该元素放入优先队列（waitForPriorityQueue）中继续等待。当元素的延迟时间大于当前时间时，则将该元素插入FIFO队列中。同时，也会遍历waitForPriorityQueue中的元素，按照上述逻辑验证时间。

### 3.限速队列

限速队列是利用延迟队列的延迟特性，延迟某个元素的插入FIFO队列的时间，达到限速的目的。接口如下：

```go
// 代码位置：k8s.io\client-go\util\workqueue\default_rate_limiters.go
type RateLimiter interface {
	// When gets an item and gets to decide how long that item should wait
	When(item interface{}) time.Duration //获取指定元素等待的时间
	// Forget indicates that an item is finished being retried. 
    // Doesn't matter whether its for perm failing or for success, we'll stop tracking it
	Forget(item interface{})             //释放指定元素，清空该元素的排队数
	// NumRequeues returns back how many failures the item has had
	NumRequeues(item interface{}) int    //指定元素的排队数，即某个元素的数量
}
```

>   限速周期：从执行AddRateLimited方法到执行完Forget方法之间的时间。

Client-go  提供了如下四种限速算法：

#### 3.1 令牌桶算法(BucketRateLimiter)

令牌桶算法通过Go的第三方库golang.org/x/time/rate实现。

```go
BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)} //每秒10个token, 最多100 token
```

原理：内部实现了一个存放token的桶，初始时桶是空的，limiter会以固定速率往桶里填充token，直到将其填满为止，多余的token会被丢弃。每个元素都会从令牌桶得到一个token，直到得到token的元素才允许通过，而没有得到token的元素处于等待状态。

算法原理图如下：

<img src="https://raw.githubusercontent.com/attlee-wang/myimage/master/image/image-20200827205623064.png" alt="image-20200827205623064" style="zoom:40%;" />

#### 3.2 排队指数算法(ItemExponentialFailureLimiter)

原理：将相同元素的排队数作为指数，排队数增大，速率限制呈指数级增长，但其最大值不会超过maxDelay。

```go
// 代码位置：k8s.io\client-go\util\workqueue\default_rate_limiters.go
func (r *ItemExponentialFailureRateLimiter) When(item interface{}) time.Duration {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

    //获取item元素排队数量并将其加1
	exp := r.failures[item]
	r.failures[item] = r.failures[item] + 1

    //计算其限速时间, 限速时间 = 基础时间*2^exp
	// The backoff is capped such that 'calculated' value never overflows.
	backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
	if backoff > math.MaxInt64 {//大于最大int64，返回最大时间
		return r.maxDelay
	}

	calculated := time.Duration(backoff)
	if calculated > r.maxDelay {/大于最大时间，返回最大时间
		return r.maxDelay
	}

	return calculated
}
```

#### 3.3 计数器算法(ItemFastSlowRateLimiter)

原理：限制一段时间内允许通过的元素数量。例如在1分钟内只允许通过100个元素，每插入一个元素，计数器自增1，当计数器到100的阈值且还在限速周期内时，则不允许元素再通过。

```go
// 代码位置：k8s.io\client-go\util\workqueue\default_rate_limiters.go
func (r *ItemFastSlowRateLimiter) When(item interface{}) time.Duration {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	r.failures[item] = r.failures[item] + 1

	if r.failures[item] <= r.maxFastAttempts { //判断排队数超过maxFastAttempts开始降速
		return r.fastDelay
	}

	return r.slowDelay
}
```

#### 3.4 混合模式(MaxOfRateLimiter)

混合模式是将多种限速算法混合使用，即多种限速算法同时生效。

下面为默认使用排队指数算法和令牌桶算法：

```go
func DefaultControllerRateLimiter() RateLimiter {
	return NewMaxOfRateLimiter(
		NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
		// 10 qps, 100 bucket size.  This is only for retry speed and its only the overall factor (not per item)
		&BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
	)
}
```



### 小结

WorkQueue支持3种队列，分别对应三种不同的使用场景。其中FIFO是基础，延时队列基于FIFO实现，限速队列又基于延迟队列实现。限速队列队列有四种限速算法，对应不同的限速场景。

