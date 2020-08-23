# WorkQueue



### 1.FIFO队列

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

#### 1.2 高并发

FIFO队列如何保证高并发下，一个元素被添加多次，但同一时刻只会处理一次？

![image-20200821170345421](/Users/attlee/Library/Application Support/typora-user-images/image-20200821170345421.png)

-   结构保证

    dirty和processing字段都为set结构，本质是通过HashMap实现，特性就是保证元素唯一，但不保证无序。所以插入到dirty和processing中的元素绝对没有重复的元素。（当dirty和processing有字段有1时，1就插入不进来。）特别是processing，保证了同一时间，一个元素只有一个goroutine在处理。

-   逻辑保证

    当processing有1，dirty没有1时，dirty 顺利插入1。当processing把1处理完后，如果dirty有1，就将1添加到queue尾部，继续按queue顺序处理。
    
    ```go
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

