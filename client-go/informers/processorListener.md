~~~go
// processorListener 用于从 sharedProcessor 拿到消息，然后交给 ResourceEventHandler 处理
// add(notification) 将消息写入到addCh，之后pop()读取消息写入到nextCh，run()从nextCh消费消息同步调用handler
type processorListener struct {
  // addCh 和nextCh 用于接收消息
	nextCh chan interface{} 					
	addCh  chan interface{}

	handler ResourceEventHandler							// 用于处理资源对象变更事件的处理程序

	syncTracker *synctrack.SingleFileTracker	// 用于跟踪Listener同步状态

	pendingNotifications buffer.RingGrowing		// 用于存储尚未分发的通知

	requestedResyncPeriod time.Duration				// 监听器希望从 shared informer 进行完全重新同步的时间间隔

	resyncPeriod time.Duration							// 实际用于逻辑的重新同步阈值,当 sharedIndexInformer 不进行重新同步时，该值为零

	nextResync time.Time										// listener 应该进行完全重新同步的最早时间

	resyncLock sync.Mutex										// 锁
}
~~~



~~~go
// 检查是否同步完成，先调用 sharedindexInformer的方法，然后在进行判断
func (p *processorListener) HasSynced() bool {
	return p.syncTracker.HasSynced()
}

// 构造函数
func newProcessListener(handler ResourceEventHandler, requestedResyncPeriod, resyncPeriod time.Duration, now time.Time, bufferSize int, hasSynced func() bool) *processorListener {
	ret := &processorListener{
		nextCh:                make(chan interface{}),
		addCh:                 make(chan interface{}),
		handler:               handler,
		syncTracker:           &synctrack.SingleFileTracker{UpstreamHasSynced: hasSynced},
		pendingNotifications:  *buffer.NewRingGrowing(bufferSize),
		requestedResyncPeriod: requestedResyncPeriod,
		resyncPeriod:          resyncPeriod,
	}

	ret.determineNextResync(now)

	return ret
}
~~~



~~~go
// 增加消息到 addCh，如果是新增通知，并且对象在初始化列表，则将 Listener 状态改为不同步,HasSynced 返回false
func (p *processorListener) add(notification interface{}) {
	if a, ok := notification.(addNotification); ok && a.isInInitialList {
		p.syncTracker.Start()
	}
	p.addCh <- notification
}
~~~



~~~~go
// 判断是否应该resync
func (p *processorListener) shouldResync(now time.Time) bool {
	p.resyncLock.Lock()
	defer p.resyncLock.Unlock()

	if p.resyncPeriod == 0 {
		return false
	}

	return now.After(p.nextResync) || now.Equal(p.nextResync)
}

// 计算Resync时间
func (p *processorListener) determineNextResync(now time.Time) {
	p.resyncLock.Lock()
	defer p.resyncLock.Unlock()

	p.nextResync = now.Add(p.resyncPeriod)
}

// 设置 resyncPeriod
func (p *processorListener) setResyncPeriod(resyncPeriod time.Duration) {
	p.resyncLock.Lock()
	defer p.resyncLock.Unlock()

	p.resyncPeriod = resyncPeriod
}
~~~~



~~~~go
// 想通过 wait.Until 在panic后 休息一秒继续运行？
// 是否panic 依赖 runtime.ReallyCrash

func (p *processorListener) run() {
	// this call blocks until the channel is closed.  When a panic happens during the notification
	// we will catch it, **the offending item will be skipped!**, and after a short delay (one second)
	// the next notification will be attempted.  This is usually better than the alternative of never
	// delivering again.
	stopCh := make(chan struct{})
	wait.Until(func() {
    // 将不同类型的通知交给 handler 不同的方法处理
		for next := range p.nextCh {
			switch notification := next.(type) {
			case updateNotification:
        // 调用注册Listener的OnUpdate(需要自行实现)
				p.handler.OnUpdate(notification.oldObj, notification.newObj)
			case addNotification:
        // 调用注册Listener的OnAdd(需要自行实现)
				p.handler.OnAdd(notification.newObj, notification.isInInitialList)
        // 如果对象在初始化列表，则修改同步状态
				if notification.isInInitialList {
					p.syncTracker.Finished()
				}
			case deleteNotification:
        //调用注册Listener的OnDelete(需要自行实现)
				p.handler.OnDelete(notification.oldObj)
			default:
				utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
			}
		}
		// the only way to get here is if the p.nextCh is empty and closed
		close(stopCh)
	}, 1*time.Second, stopCh)
}

~~~~



~~~~go
// run() 和 pop() 为什么不直接用channel？
// 消费addCh 通知写到nextCh
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
  //退出前关闭nextCh，让 run() 函数可以退出
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
      // 将消息放到nextCh 供run() 消费
		case nextCh <- notification:
			// Notification dispatched
			var ok bool
      // 从 pendingNotifications 拿通知
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil // Disable this select case
			}
      // 从addCh拿通知
		case notificationToAdd, ok := <-p.addCh:
      // addCh 关闭直接返回
			if !ok {
				return
			}
      // 第一次读取到通知做了优化，直接发给nextCh，之后都写到pendingNotifications
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
~~~~

