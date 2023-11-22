````go
// SharedInformerOption defines the functional option type for SharedInformerFactory.
type SharedInformerOption func(*sharedInformerFactory) *sharedInformerFactory



type sharedInformerFactory struct {
	client           kubernetes.Interface											// clientset
	namespace        string																		// namespace
	tweakListOptions internalinterfaces.TweakListOptionsFunc	// 过滤器
	lock             sync.Mutex
	defaultResync    time.Duration														// 默认的Resync时间
	customResync     map[reflect.Type]time.Duration						// 独立的Resync时间

	informers map[reflect.Type]cache.SharedIndexInformer			//	实现单例
	// startedInformers is used for tracking which informers have been started.
	// This allows Start() to be called multiple times safely.
	startedInformers map[reflect.Type]bool										// 用于判断SharedIndexInformer是否已经创建
	// wg tracks how many goroutines were started.
	wg sync.WaitGroup
	// shuttingDown is true when Shutdown has been called. It may still be running
	// because it needs to wait for goroutines.
	shuttingDown bool																					// 用于判断是否关闭
}


// NewSharedInformerFactoryWithOptions constructs a new instance of a SharedInformerFactory with additional options.
// 构造函数，通过Options模式调整参数，最后返回sharedInformerFactory
func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) SharedInformerFactory {
	factory := &sharedInformerFactory{
		client:           client,
		namespace:        v1.NamespaceAll,
		defaultResync:    defaultResync,
		informers:        make(map[reflect.Type]cache.SharedIndexInformer),
		startedInformers: make(map[reflect.Type]bool),
		customResync:     make(map[reflect.Type]time.Duration),
	}

	// Apply all options
	for _, opt := range options {
		factory = opt(factory)
	}

	return factory
}


````



~~~~go
// 启动函数
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()

  // 如果已经shuttingDown,则退出
	if f.shuttingDown {
		return
	}

  // 遍历所有SharedIndexInformer，如果没启动的话，在新goroutine中启动
	for informerType, informer := range f.informers {
		if !f.startedInformers[informerType] {
			f.wg.Add(1)
			// We need a new variable in each loop iteration,
			// otherwise the goroutine would use the loop variable
			// and that keeps changing.
			informer := informer
			go func() {
				defer f.wg.Done()
				informer.Run(stopCh)
			}()
			f.startedInformers[informerType] = true
		}
	}
}

// 关闭函数
func (f *sharedInformerFactory) Shutdown() {
	f.lock.Lock()
	f.shuttingDown = true
	f.lock.Unlock()

	// Will return immediately if there is nothing to wait for.
  // 必须等到所有启动的SharedIndexInformer都结束，才退出
	f.wg.Wait()
}
~~~~



~~~~go
// 获取已启动SharedIndexInformer的同步状态
func (f *sharedInformerFactory) WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool {
	// 获取已经启动的informer
  informers := func() map[reflect.Type]cache.SharedIndexInformer {
		f.lock.Lock()
		defer f.lock.Unlock()

		informers := map[reflect.Type]cache.SharedIndexInformer{}
		for informerType, informer := range f.informers {
			if f.startedInformers[informerType] {
				informers[informerType] = informer
			}
		}
		return informers
	}()

	res := map[reflect.Type]bool{}
  // 遍历已启动的SharedIndexInformer，并调用 HasSynced 方法判断是否同步，直到同步完成或者报错、取消
	for informType, informer := range informers {
		res[informType] = cache.WaitForCacheSync(stopCh, informer.HasSynced)
	}
	return res
}
~~~~



````go
// InformerFor returns the SharedIndexInformer for obj using an internal
// client.
// 获取 SharedIndexInformer:
// 	factory := informers.NewSharedInformerFactory(clientset, 0) 初始化sharedInformerFactory
// 	nodeInformer := factory.Core().V1().Nodes().Informer() 获取node的Informer，最终调用的就是 InformerFor 方法
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	f.lock.Lock()
	defer f.lock.Unlock()

  // 如果存在直接返回
	informerType := reflect.TypeOf(obj)
	informer, exists := f.informers[informerType]
	if exists {
		return informer
	}

  // 判断是否有独立的 resync
	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}

  // 使用传进来的函数构造 SharedIndexInformer ，并放到单例map中
	informer = newFunc(f.client, resyncPeriod)
	f.informers[informerType] = informer

	return informer
}
````

