# 								Queue

## Interface

````go
type Interface interface {
	Add(item interface{})												// 添加对象
	Len() int																		// 获取队列
	Get() (item interface{}, shutdown bool)			// 获取item
	Done(item interface{})											// 
	ShutDown()
	ShutDownWithDrain()
	ShuttingDown() bool
}
````

## Type

````go
// Type is a work queue (see the package comment).
type Type struct {
	// queue defines the order in which we will work on items. Every
	// element of queue should be in the dirty set and not in the
	// processing set.
	queue []t

	// dirty defines all of the items that need to be processed.
	dirty set							// 需要处理的item队列

	// Things that are currently being processed are in the processing set.
	// These things may be simultaneously in the dirty set. When we finish
	// processing something and remove it from this set, we'll check if
	// it's in the dirty set, and if so, add it to the queue.
	processing set				// 正在处理队列

	cond *sync.Cond

	shuttingDown bool			// 队列状态
	drain        bool

	metrics queueMetrics

	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.WithTicker
}

// 判断 item是否已经在dirty 队列，如果不在则增加
// 再判断item是否已经在processing对了，如果不在则加到 queue队列
func (q *Type) Add(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	if q.shuttingDown {
		return
	}
	if q.dirty.has(item) {
		return
	}

	q.metrics.add(item)

	q.dirty.insert(item)
	if q.processing.has(item) {
		return
	}

	q.queue = append(q.queue, item)
	q.cond.Signal()
}


// 获取要处理的item和队列状态
func (q *Type) Get() (item interface{}, shutdown bool) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
  // 如果待处理队列长度为0并且队列未关闭，持续等待
	for len(q.queue) == 0 && !q.shuttingDown {
		q.cond.Wait()
	}
  // 如果队列已经关闭，直接返回
	if len(q.queue) == 0 {
		// We must be shutting down.
		return nil, true
	}

  // 从queue头部取出要处理的item
	item = q.queue[0]
	// The underlying array still exists and reference this object, so the object will not be garbage collected.
	q.queue[0] = nil
	q.queue = q.queue[1:]

	q.metrics.get(item)

  // 将item加入到processing，并从dirty删除
	q.processing.insert(item)
	q.dirty.delete(item)

	return item, false
}

// 处理完成从 processing 删除，如果在处理时，item再次被添加到dirty则添加到queue，以便后续处理
func (q *Type) Done(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.metrics.done(item)

	q.processing.delete(item)
	if q.dirty.has(item) {
		q.queue = append(q.queue, item)
		q.cond.Signal()
	} else if q.processing.len() == 0 {
		q.cond.Signal()
	}
}

// 关闭队列忽略新增的item
func (q *Type) ShutDown() {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.drain = false
	q.shuttingDown = true
	q.cond.Broadcast()
}

// 关闭队列，等待processing队列处理完成
// 如果处理完item不执行Done() 会一直阻塞
func (q *Type) ShutDownWithDrain() {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.drain = true
	q.shuttingDown = true
	q.cond.Broadcast()

	for q.processing.len() != 0 && q.drain {
		q.cond.Wait()
	}
}


````
