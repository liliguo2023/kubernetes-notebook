````go
type SharedInformer interface {
  // 添加处理器，使用 sharedIndexInformer 的defaultEventHandlerResyncPeriod
	AddEventHandler(handler ResourceEventHandler) (ResourceEventHandlerRegistration, error)
  
	// 添加处理器，可以提供 resyncPeriod
	AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration (ResourceEventHandlerRegistration, error)
                                  
	// 删除处理器
	RemoveEventHandler(handle ResourceEventHandlerRegistration) error

  // 获取Store
	GetStore() Store

  // 获取Controller
	GetController() Controller
  
  // 核心函数
	Run(stopCh <-chan struct{})
  
	HasSynced() bool

	LastSyncResourceVersion() string

	SetWatchErrorHandler(handler WatchErrorHandler) error

	SetTransform(handler TransformFunc) error

	IsStopped() bool
}

type sharedIndexInformer struct {
  // 底层存储
	indexer    Indexer
  
  // controller 会启动Processloop和reflector
	controller Controller

  // 负责启动处理器
	processor             *sharedProcessor
  
  // 用于判断indexer中的对象是否被修改过
	cacheMutationDetector MutationDetector

  // reflector 和apiserver交互使用
	listerWatcher ListerWatcher

	objectType runtime.Object

	objectDescription string

	// resyncCheckPeriod is how often we want the reflector's resync timer to fire so it can call
	// shouldResync to check if any of our listeners need a resync.
	resyncCheckPeriod time.Duration
	// defaultEventHandlerResyncPeriod is the default resync period for any handlers added via
	// AddEventHandler (i.e. they don't specify one and just want to use the shared informer's default
	// value).
	defaultEventHandlerResyncPeriod time.Duration
	// clock allows for testability
	clock clock.Clock

	started, stopped bool
	startedLock      sync.Mutex

	blockDeltas sync.Mutex

	watchErrorHandler WatchErrorHandler

	transform TransformFunc
}
````



~~~~go
// 启动informer
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

  // 判断是否已经启动过
	if s.HasStarted() {
		klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
		return
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

    // 构造Queue
		fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
			KnownObjects:          s.indexer,
			EmitDeltaTypeReplaced: true,
			Transformer:           s.transform,
		})

		cfg := &Config{
			Queue:             fifo,
			ListerWatcher:     s.listerWatcher,
			ObjectType:        s.objectType,
			ObjectDescription: s.objectDescription,
			FullResyncPeriod:  s.resyncCheckPeriod,
			RetryOnError:      false,
			ShouldResync:      s.processor.shouldResync,

			Process:           s.HandleDeltas,
			WatchErrorHandler: s.watchErrorHandler,
		}

    // 构造controller，cfg里面为啥不搞一个clock？
		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
  // cacheMutationDetector 用于检测s.indexer数据是否修改，依赖 KUBE_CACHE_MUTATION_DETECTOR 变量配置，默认false
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
  // 
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
  
  // 启动controller
	s.controller.Run(stopCh)
}

~~~~



````go
// 添加新的 IndexFunc
func (s *sharedIndexInformer) AddIndexers(indexers Indexers) error {
	s.startedLock.Lock()
	defer s.startedLock.Unlock()

	if s.started {
		return fmt.Errorf("informer has already started")
	}

	return s.indexer.AddIndexers(indexers)
}
````



~~~~go
// desired 或者 check 为0，直接返回，否则返回较大的一个
func determineResyncPeriod(desired, check time.Duration) time.Duration {
	if desired == 0 {
		return desired
	}
	if check == 0 {
		klog.Warningf("The specified resyncPeriod %v is invalid because this shared informer doesn't support resyncing", desired)
		return 0
	}
	if desired < check {
		klog.Warningf("The specified resyncPeriod %v is being increased to the minimum resyncCheckPeriod %v", desired, check)
		return check
	}
	return desired
}
~~~~



```go
// 处理器需要实现这个接口
type ResourceEventHandler interface {
	OnAdd(obj interface{}, isInInitialList bool)
	OnUpdate(oldObj, newObj interface{})
	OnDelete(obj interface{})
}

const minimumResyncPeriod = 1 * time.Second


// 添加处理器
func (s *sharedIndexInformer) AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) (ResourceEventHandlerRegistration, error) {
	s.startedLock.Lock()
	defer s.startedLock.Unlock()

	if s.stopped {
		return nil, fmt.Errorf("handler %v was not added to shared informer because it has stopped already", handler)
	}

  // resyncPeriod 可以不开，但是不能太短
	if resyncPeriod > 0 {
		if resyncPeriod < minimumResyncPeriod {
			klog.Warningf("resyncPeriod %v is too small. Changing it to the minimum allowed value of %v", resyncPeriod, minimumResyncPeriod)
			resyncPeriod = minimumResyncPeriod
		}

		if resyncPeriod < s.resyncCheckPeriod {
      // 如果 sharedIndexInformer 启动， 调整 resyncPeriod 为 s.resyncCheckPeriod
			if s.started {
				klog.Warningf("resyncPeriod %v is smaller than resyncCheckPeriod %v and the informer has already started. Changing it to %v", resyncPeriod, s.resyncCheckPeriod, s.resyncCheckPeriod)
				resyncPeriod = s.resyncCheckPeriod
			} else {
				// if the event handler's resyncPeriod is smaller than the current resyncCheckPeriod, update
				// resyncCheckPeriod to match resyncPeriod and adjust the resync periods of all the listeners
				// accordingly
        // 调整 s.resyncCheckPeriod 以及调用 determineResyncPeriod 函数调整listener的resyncPeriod
				s.resyncCheckPeriod = resyncPeriod
				s.processor.resyncCheckPeriodChanged(resyncPeriod)
			}
		}
	}

  // 根据 handler 构造listener
	listener := newProcessListener(handler, resyncPeriod, determineResyncPeriod(resyncPeriod, s.resyncCheckPeriod), s.clock.Now(), initialBufferSize, s.HasSynced)

  // 如果没启动直接添加
	if !s.started {
		return s.processor.addListener(listener), nil
	}

	// in order to safely join, we have to
	// 1. stop sending add/update/delete notifications
	// 2. do a list against the store
	// 3. send synthetic "Add" events to the new handler
	// 4. unblock
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	handle := s.processor.addListener(listener)
	for _, item := range s.indexer.List() {
		// Note that we enqueue these notifications with the lock held
		// and before returning the handle. That means there is never a
		// chance for anyone to call the handle's HasSynced method in a
		// state when it would falsely return true (i.e., when the
		// shared informer is synced but it has not observed an Add
		// with isInitialList being true, nor when the thread
		// processing notifications somehow goes faster than this
		// thread adding them and the counter is temporarily zero).
		listener.add(addNotification{newObj: item, isInInitialList: true})
	}
	return handle, nil
}
```



~~~~go
// controller启动之后 processLoop 消费的数据会传递给这个函数
func (s *sharedIndexInformer) HandleDeltas(obj interface{}, isInInitialList bool) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

  // 如果是Deltas交给processDeltas处理
	if deltas, ok := obj.(Deltas); ok {
		return processDeltas(s, s.indexer, deltas, isInInitialList)
	}
	return errors.New("object given as Process argument is not Deltas")
}

func processDeltas(
	// Object which receives event notifications from the given deltas
	handler ResourceEventHandler,
	clientState Store,
	deltas Deltas,
	isInInitialList bool,
) error {
	// from oldest to newest
	for _, d := range deltas {
		obj := d.Object

    // 根据类型更新indexer数据，并交给handler处理
    // handler就是通过 AddEventHandlerWithResyncPeriod 等添加的处理器
		switch d.Type {
		case Sync, Replaced, Added, Updated:
      // 先从indexer 里面查找obj，如果找到更新 cache的索引信息
			if old, exists, err := clientState.Get(obj); err == nil && exists {
				if err := clientState.Update(obj); err != nil {
					return err
				}
        // 调用sharedIndexInformer 的OnUpdate
				handler.OnUpdate(old, obj)
			} else {
        // 增加 cache 中的索引信息
				if err := clientState.Add(obj); err != nil {
					return err
				}
				handler.OnAdd(obj, isInInitialList)
			}
		case Deleted:
      // 删除cache 中的索引信息
			if err := clientState.Delete(obj); err != nil {
				return err
			}
			handler.OnDelete(obj)
		}
	}
	return nil
}
~~~~



~~~~go
// Conforms to ResourceEventHandler
func (s *sharedIndexInformer) OnAdd(obj interface{}, isInInitialList bool) {
	// Invocation of this function is locked under s.blockDeltas, so it is
	// save to distribute the notification
	s.cacheMutationDetector.AddObject(obj)
	s.processor.distribute(addNotification{newObj: obj, isInInitialList: isInInitialList}, false)
}

// Conforms to ResourceEventHandler
func (s *sharedIndexInformer) OnUpdate(old, new interface{}) {
	isSync := false

	// If is a Sync event, isSync should be true
	// If is a Replaced event, isSync is true if resource version is unchanged.
	// If RV is unchanged: this is a Sync/Replaced event, so isSync is true

  // 判断是否是同步事件
	if accessor, err := meta.Accessor(new); err == nil {
		if oldAccessor, err := meta.Accessor(old); err == nil {
			// Events that didn't change resourceVersion are treated as resync events
			// and only propagated to listeners that requested resync
			isSync = accessor.GetResourceVersion() == oldAccessor.GetResourceVersion()
		}
	}

	// Invocation of this function is locked under s.blockDeltas, so it is
	// save to distribute the notification
	s.cacheMutationDetector.AddObject(new)
  // 调用sharedProcessor.distribute 
	s.processor.distribute(updateNotification{oldObj: old, newObj: new}, isSync)
}

// Conforms to ResourceEventHandler
func (s *sharedIndexInformer) OnDelete(old interface{}) {
	// Invocation of this function is locked under s.blockDeltas, so it is
	// save to distribute the notification
	s.processor.distribute(deleteNotification{oldObj: old}, false)
}
~~~~

