~~~~go

//这段话描述了一个软件组件或类的设计，其中有一个名为sharedProcessor的对象，它包含了一个processorListener的集合，并且有能力将通知对象分发给它的监听器们。在这个场景中，有两种分发操作。
//
//同步分发（sync distributions）：
//
//这种分发操作会将通知对象发送给一部分监听器。
//这个子集的计算是在偶发调用 shouldResync 方法时重新计算的。
//初始时，所有的监听器都会被放入这个子集中。
//换句话说，同步分发是有选择性的，发送给那些需要重新同步的监听器。
//
//非同步分发（non-sync distributions）：
//这种分发操作会将通知对象发送给每一个监听器，而不考虑是否需要重新同步。
//
//换句话说，非同步分发是向每个监听器广播通知，无论其当前状态如何。
//综合起来，sharedProcessor 对象能够以两种方式将通知传递给其监听器集合：一种是有选择性地，只选择需要重新同步的监听器；另一种是无选择地，将通知发送给所有监听器。这种设计可能是为了在不同的场景下提供灵活性，以满足不同的需求。

type sharedProcessor struct {
	listenersStarted bool
	listenersLock    sync.RWMutex
	// Map from listeners to whether or not they are currently syncing
	listeners map[*processorListener]bool
	clock     clock.Clock
	wg        wait.Group
}


~~~~



~~~~go
// 获取 Listener
func (p *sharedProcessor) getListener(registration ResourceEventHandlerRegistration) *processorListener {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	if p.listeners == nil {
		return nil
	}

	if result, ok := registration.(*processorListener); ok {
		if _, exists := p.listeners[result]; exists {
			return result
		}
	}

	return nil
}
~~~~



~~~~~go
// 添加Listener 如果sharedProcessor已经运行，则新增的listener也开始运行
func (p *sharedProcessor) addListener(listener *processorListener) ResourceEventHandlerRegistration {
	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()

	if p.listeners == nil {
		p.listeners = make(map[*processorListener]bool)
	}

	p.listeners[listener] = true

	if p.listenersStarted {
		p.wg.Start(listener.run)
		p.wg.Start(listener.pop)
	}

	return listener
}
~~~~~



~~~~go
// 删除 Listener ，如果已经启动则关闭
func (p *sharedProcessor) removeListener(handle ResourceEventHandlerRegistration) error {
	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()

	listener, ok := handle.(*processorListener)
	if !ok {
		return fmt.Errorf("invalid key type %t", handle)
	} else if p.listeners == nil {
		// No listeners are registered, do nothing
		return nil
	} else if _, exists := p.listeners[listener]; !exists {
		// Listener is not registered, just do nothing
		return nil
	}

	delete(p.listeners, listener)

	if p.listenersStarted {
		close(listener.addCh)
	}

	return nil
}
~~~~



~~~~go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	for listener, isSyncing := range p.listeners {
		switch {
		case !sync:
			// non-sync messages are delivered to every listener
			listener.add(obj)
		case isSyncing:
			// sync messages are delivered to every syncing listener
			listener.add(obj)
		default:
			// skipping a sync obj for a non-syncing listener
		}
	}
}
~~~~



~~~~go
// 非同步分发给所有 listener, 同步消息分发给 syncing listener
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	for listener, isSyncing := range p.listeners {
		switch {
		case !sync:
			// non-sync messages are delivered to every listener
			listener.add(obj)
		case isSyncing:
			// sync messages are delivered to every syncing listener
			listener.add(obj)
		default:
			// skipping a sync obj for a non-syncing listener
		}
	}
}
~~~~



~~~go
// 启动所有 listener，并将 listenersStarted 改为true
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
  // 遍历启动
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		for listener := range p.listeners {
			p.wg.Start(listener.run)
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
	<-stopCh

	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()
  // 收到停止信号后，遍历停止
	for listener := range p.listeners {
		close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
	}

	// Wipe out list of listeners since they are now closed
	// (processorListener cannot be re-used)
	p.listeners = nil

	// Reset to false since no listeners are running
	p.listenersStarted = false

  // 等待所有goroutine结束
	p.wg.Wait() // Wait for all .pop() and .run() to stop
}

~~~



~~~go
// 判断 listener 是否需要resync
// 遍历了所有listener 更新了Listener 是否syncing
func (p *sharedProcessor) shouldResync() bool {
	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()

	resyncNeeded := false
	now := p.clock.Now()
	for listener := range p.listeners {
		// need to loop through all the listeners to see if they need to resync so we can prepare any
		// listeners that are going to be resyncing.
		shouldResync := listener.shouldResync(now)
		p.listeners[listener] = shouldResync

		if shouldResync {
			resyncNeeded = true
			listener.determineNextResync(now)
		}
	}
	return resyncNeeded
}
~~~



~~~go
// 看看 resync时间是否需要修改
func (p *sharedProcessor) resyncCheckPeriodChanged(resyncCheckPeriod time.Duration) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	for listener := range p.listeners {
		resyncPeriod := determineResyncPeriod(
			listener.requestedResyncPeriod, resyncCheckPeriod)
		listener.setResyncPeriod(resyncPeriod)
	}
}
~~~

