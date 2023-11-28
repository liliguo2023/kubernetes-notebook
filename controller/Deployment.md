## Deployment Controller

>  kube-controller-manager 内嵌随 Kubernetes 一起发布的核心控制回路，在 Kubernetes 中，每个控制器是一个控制回路，通过 API 服务器监视集群的共享状态， 并尝试进行更改以将当前状态转为期望状态(kubernetes中文官方截取内容)
>
>  deployment controller 通过监听deployment、replicaset、pod等资源，从而实现deployment的创建、删除、扩容、回滚、暂停恢复等操作
>
>  最好熟练使用deployment，方便模拟deployment的多种使用情况
>
>  版本:  v1.28.0

~~~go
// kubernetes Controller列表
// cmd/kube-controller-manager/app/controllermanager.go:426
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}

	// All of the controllers must have unique names, or else we will explode.
	register := func(name string, fn InitFunc) {
		if _, found := controllers[name]; found {
			panic(fmt.Sprintf("controller name %q was registered twice", name))
		}
		controllers[name] = fn
	}

	...
	register(names.DeploymentController, startDeploymentController)

	return controllers
}
~~~



~~~go
// 根据传入的informer构造Deployment Controller，并通过Run()函数启动
// cmd/kube-controller-manager/app/apps.go:76
func startDeploymentController(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {
	dc, err := deployment.NewDeploymentController(
		ctx,
		controllerContext.InformerFactory.Apps().V1().Deployments(),
		controllerContext.InformerFactory.Apps().V1().ReplicaSets(),
		controllerContext.InformerFactory.Core().V1().Pods(),
		controllerContext.ClientBuilder.ClientOrDie("deployment-controller"),
	)
	if err != nil {
		return nil, true, fmt.Errorf("error creating Deployment controller: %v", err)
	}
	go dc.Run(ctx, int(controllerContext.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs))
	return nil, true, nil
}
~~~



````go
// pkg/controller/deployment/deployment_controller.go:66
type DeploymentController struct {
	// 用于操作RS，弃养、认领等
	rsControl controller.RSControlInterface
  // 客户端
	client    clientset.Interface

  //事件相关
	eventBroadcaster record.EventBroadcaster
	eventRecorder    record.EventRecorder

	// 主要处理函数，负责处理从queue中消费的对象
	syncHandler func(ctx context.Context, dKey string) error
	// 负责将对象放到queue中
	enqueueDeployment func(deployment *apps.Deployment)

	// 用于从indexer中获取数据
	dLister appslisters.DeploymentLister
	rsLister appslisters.ReplicaSetLister
	podLister corelisters.PodLister

	// 用于判断同步状态，实际调用的是DeltaFIFO的方法， Reflector的List方法写入的数据被消费完后返回true
	dListerSynced cache.InformerSynced
	rsListerSynced cache.InformerSynced
	podListerSynced cache.InformerSynced

	// 延迟队列和令牌桶
	queue workqueue.RateLimitingInterface
}
````



~~~go
// 构造函数
// pkg/controller/deployment/deployment_controller.go:101
func NewDeploymentController(ctx context.Context, dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
	eventBroadcaster := record.NewBroadcaster()
	logger := klog.FromContext(ctx)
	dc := &DeploymentController{
		client:           client,
		eventBroadcaster: eventBroadcaster,
		eventRecorder:    eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
	}
	dc.rsControl = controller.RealRSControl{
		KubeClient: client,
		Recorder:   dc.eventRecorder,
	}

  // 添加Listener，从DeltaFIFO中pop()的数据，根据不同类型，会被下面的函数处理
	dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			dc.addDeployment(logger, obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			dc.updateDeployment(logger, oldObj, newObj)
		},
		// This will enter the sync loop and no-op, because the deployment has been deleted from the store.
		DeleteFunc: func(obj interface{}) {
			dc.deleteDeployment(logger, obj)
		},
	})
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			dc.addReplicaSet(logger, obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			dc.updateReplicaSet(logger, oldObj, newObj)
		},
		DeleteFunc: func(obj interface{}) {
			dc.deleteReplicaSet(logger, obj)
		},
	})
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: func(obj interface{}) {
			dc.deletePod(logger, obj)
		},
	})

  // 入队出队方法
	dc.syncHandler = dc.syncDeployment
	dc.enqueueDeployment = dc.enqueue

	dc.dLister = dInformer.Lister()
	dc.rsLister = rsInformer.Lister()
	dc.podLister = podInformer.Lister()
	dc.dListerSynced = dInformer.Informer().HasSynced
	dc.rsListerSynced = rsInformer.Informer().HasSynced
	dc.podListerSynced = podInformer.Informer().HasSynced
	return dc, nil
}
~~~



~~~go
// pkg/controller/deployment/deployment_controller.go:157
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
	defer utilruntime.HandleCrash()

	// Start events processing pipeline.
	dc.eventBroadcaster.StartStructuredLogging(0)
	dc.eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: dc.client.CoreV1().Events("")})
	defer dc.eventBroadcaster.Shutdown()

	defer dc.queue.ShutDown()

	logger := klog.FromContext(ctx)
	logger.Info("Starting controller", "controller", "deployment")
	defer logger.Info("Shutting down controller", "controller", "deployment")

  // 等待cache数据同步完成，这里调用的就是之前初始化的判断同步状态的方法
	if !cache.WaitForNamedCacheSync("deployment", ctx.Done(), dc.dListerSynced, dc.rsListerSynced, dc.podListerSynced) {
		return
	}

  // 启动worker 可以通过concurrent-deployment-syncs 指定，默认值是5个
	for i := 0; i < workers; i++ {
  // wait.UntilWithContext 保证worker持续运行，间隔是1s，不带抖动
		go wait.UntilWithContext(ctx, dc.worker, time.Second)
	}

	<-ctx.Done()
}
~~~



~~~go
// pkg/controller/deployment/deployment_controller.go:473
func (dc *DeploymentController) worker(ctx context.Context) {
  // 循环运行
	for dc.processNextWorkItem(ctx) {
	}
}

func (dc *DeploymentController) processNextWorkItem(ctx context.Context) bool {
  // 从队列消费数据，没数据阻塞，队列关闭返回false
  // client-go/util/workqueue/Queue.md
	key, quit := dc.queue.Get()
	if quit {
		return false
	}
  // 处理完key之后通知队列
	defer dc.queue.Done(key)

  // 调用主要的处理函数key是namespace/name
	err := dc.syncHandler(ctx, key.(string))
	dc.handleErr(ctx, err, key)

	return true
}
~~~



~~~go
// pkg/controller/deployment/deployment_controller.go:581
func (dc *DeploymentController) syncDeployment(ctx context.Context, key string) error {
	logger := klog.FromContext(ctx)
  // 从key中取出namespace 和name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		logger.Error(err, "Failed to split meta namespace cache key", "cacheKey", key)
		return err
	}

	startTime := time.Now()
	logger.V(4).Info("Started syncing deployment", "deployment", klog.KRef(namespace, name), "startTime", startTime)
	defer func() {
		logger.V(4).Info("Finished syncing deployment", "deployment", klog.KRef(namespace, name), "duration", time.Since(startTime))
	}()

  // 通过namespace 和name 从cache中获取对应的deployment
	deployment, err := dc.dLister.Deployments(namespace).Get(name)
	if errors.IsNotFound(err) {
		logger.V(2).Info("Deployment has been deleted", "deployment", klog.KRef(namespace, name))
		return nil
	}
	if err != nil {
		return err
	}


  // 深拷贝，避免直接操作cache中的数据
	d := deployment.DeepCopy()

	everything := metav1.LabelSelector{}
  // 如果没有Selector 则返回
	if reflect.DeepEqual(d.Spec.Selector, &everything) {
		dc.eventRecorder.Eventf(d, v1.EventTypeWarning, "SelectingAll", "This deployment is selecting all pods. A non-empty selector is required.")
		if d.Status.ObservedGeneration < d.Generation {
			d.Status.ObservedGeneration = d.Generation
			dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
		}
		return nil
	}

	// List ReplicaSets owned by this Deployment, while reconciling ControllerRef
	// through adoption/orphaning.
  // 通过认领和弃养找到属于这个deployment的所有RS
	rsList, err := dc.getReplicaSetsForDeployment(ctx, d)
	if err != nil {
		return err
	}

  // 获取pod列表，key是RS的uid，value是pod指针
	podMap, err := dc.getPodMapForDeployment(d, rsList)
	if err != nil {
		return err
	}

  // 如果deployment已删除，更新deployment status和RS
	if d.DeletionTimestamp != nil {
		return dc.syncStatusOnly(ctx, d, rsList)
	}

  // 暂停/恢复后更新status
  // 测试: kubectl rollout pause/resume deploy/name
	if err = dc.checkPausedConditions(ctx, d); err != nil {
		return err
	}

  // 如果是暂停状态调整deployment和RS
	if d.Spec.Paused {
		return dc.sync(ctx, d, rsList)
	}

	// extensions/v1beta1 and apps/v1beta1 使用，所以就没看
	if getRollbackTo(d) != nil {
		return dc.rollback(ctx, d, rsList)
	}

  // 检测是否有scaling event
	scalingEvent, err := dc.isScalingEvent(ctx, d, rsList)
	if err != nil {
		return err
	}
  // 调整deployment和RS
	if scalingEvent {
		return dc.sync(ctx, d, rsList)
	}

  // 判断Type
	switch d.Spec.Strategy.Type {
  // Recreate，会先停掉所有pod，然后在启动新的pod，没用过
	case apps.RecreateDeploymentStrategyType:
		return dc.rolloutRecreate(ctx, d, rsList, podMap)
	case apps.RollingUpdateDeploymentStrategyType:
  // RollingUpdate
		return dc.rolloutRolling(ctx, d, rsList)
	}
	return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}
~~~











