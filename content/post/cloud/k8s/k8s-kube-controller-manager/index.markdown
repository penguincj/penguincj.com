



# 关键的结构体

## GenericControllerManagerConfigurationOptions

保存着通用的配置信息，主要从 `GenericControllerManagerConfiguration` 获取的到，而 `GenericControllerManagerConfiguration` 是 Scheme 通过默认参数初始化函数获得的。


主要初始化的流程是通过 NewKubeControllerManagerOptions()

- 首先调用 NewDefaultComponentConfig() 初始化 `KubeControllerManagerConfiguration`
- 然后初始化 GenericControllerManagerConfigurationOptions 中对应的各种 options





# 流程





## Controller 初始化

调用 NewControllerInitializers() 来初始化一个控制器名字与控制器初始化函数的映射表。

func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	...
	controllers["deployment"] = startDeploymentController
	controllers["replicaset"] = startReplicaSetController
 	...
}













# Deployment 的启动流程

Deployment 的启动方法在 `startDeploymentController`

## startDeploymentController


```go
func startDeploymentController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "deployments"}] {
		return nil, false, nil
	}
	dc, err := deployment.NewDeploymentController(
		ctx.InformerFactory.Apps().V1().Deployments(),
		ctx.InformerFactory.Apps().V1().ReplicaSets(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.ClientBuilder.ClientOrDie("deployment-controller"),
	)
	if err != nil {
		return nil, true, fmt.Errorf("error creating Deployment controller: %v", err)
	}
	go dc.Run(int(ctx.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs), ctx.Stop)
	return nil, true, nil
}
```

主要分为以下三个步骤：

1. 判断资源版本是否可用，即deployment应该为apps/v1；
2. 通过 `NewDeploymentController` 创建一个DeploymentController对象；
3. 调用 `run` 启动，循环检测资源的变化，完成最终处理任务。


## NewDeploymentController

kubernetes controller使用的是informer和workqueue的机制，即通过informer监控通知资源的变化，通过workqueue入队处理资源。

进入NewDeploymentController方法，对informer的操作主要有Deployment、ReplicaSet以及Pod，与我们对Deployment了解的资源基本符合，通过各种处理方法处理资源的CRUD等操作。
。

NewDeploymentController主要构建DeploymentController结构体。

该部分主要处理了以下逻辑：

- 构建并运行事件处理器eventBroadcaster。
- 初始化赋值rsControl、clientset、workqueue。
- 添加dInformer、rsInformer、podInformer的ResourceEventHandlerFuncs，其中主要为AddFunc、UpdateFunc、DeleteFunc三类方法。
- 构造deployment、rs、pod的Informer的Lister函数和HasSynced函数。
- 调用syncHandler，来实现syncDeployment。


### eventBroadcaster

调用事件处理器来记录deployment相关的事件。

```go
// NewDeploymentController
eventBroadcaster := record.NewBroadcaster()
eventBroadcaster.StartLogging(klog.Infof)
eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: client.CoreV1().Events("")})

// 构造DeploymentController，包括clientset、workqueue和rsControl。其中rsControl是具体实现rs逻辑的controller。


dc := &DeploymentController{
	client:        client,
	eventRecorder: eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
	queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
}
dc.rsControl = controller.RealRSControl{
	KubeClient: client,
	Recorder:   dc.eventRecorder,
}

dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    dc.addDeployment,
    UpdateFunc: dc.updateDeployment,
    // This will enter the sync loop and no-op, because the deployment has been deleted from the store.
    DeleteFunc: dc.deleteDeployment,
})
rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    dc.addReplicaSet,
    UpdateFunc: dc.updateReplicaSet,
    DeleteFunc: dc.deleteReplicaSet,
})
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    DeleteFunc: dc.deletePod,
})

dc.syncHandler = dc.syncDeployment
dc.enqueueDeployment = dc.enqueue


```

通过 `syncDeployment` 处理Deployment的检测，保证Deployment资源一直能够被处理，`enqueue` 方法保证资源的及时入队。

创建完成之后，接下来就是启动操作。通过Run方法进入到最终的dc.worker方法，最终调用的为dc.syncHandler方法，也就是配置中的syncDeployment方法，通过不停循环检测执行 `syncDeployment` 方法，保证资源的及时处理。

主要的流程如下：
1、通过参数key获取资源的namespace和name；
2、通过namespace和name获取deployment信息；
3、通过Deployment获取其对应的ReplicaSet信息；
4、通过Deployment和ReplicaSet获取相对应的pod信息；
5、根据配置的参数，判断需要进行什么操作，包括sync、rollback、rolloutRecreate以及rolloutRolling操作。
至此，DeploymentController的任务基本完成。我们自定义实现CRD与Controller的步骤基本与此一致，都是通过监听需要的资源，通过对资源列表和资源操作进行比对，判断对资源最终的操作，完成controller任务。


主要的代码如下：

```go
// pkg/controller/deployment/deployment_controller.go
func (dc *DeploymentController) syncDeployment(key string) error {
	startTime := time.Now()
	klog.V(4).Infof("Started syncing deployment %q (%v)", key, startTime)
	defer func() {
		klog.V(4).Infof("Finished syncing deployment %q (%v)", key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	deployment, err := dc.dLister.Deployments(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(2).Infof("Deployment %v has been deleted", key)
		return nil
	}
	if err != nil {
		return err
	}

	// Deep-copy otherwise we are mutating our cache.
	// TODO: Deep-copy only when needed.
	d := deployment.DeepCopy()

	everything := metav1.LabelSelector{}
	if reflect.DeepEqual(d.Spec.Selector, &everything) {
		dc.eventRecorder.Eventf(d, v1.EventTypeWarning, "SelectingAll", "This deployment is selecting all pods. A non-empty selector is required.")
		if d.Status.ObservedGeneration < d.Generation {
			d.Status.ObservedGeneration = d.Generation
			dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(d)
		}
		return nil
	}

	// List ReplicaSets owned by this Deployment, while reconciling ControllerRef
	rsList, err := dc.getReplicaSetsForDeployment(d)
	if err != nil {
		return err
	}
	podMap, err := dc.getPodMapForDeployment(d, rsList)
	if err != nil {
		return err
	}

	if d.DeletionTimestamp != nil {
		return dc.syncStatusOnly(d, rsList)
	}

	// Update deployment conditions with an Unknown condition when pausing/resuming
	// a deployment. In this way, we can be sure that we won't timeout when a user
	// resumes a Deployment with a set progressDeadlineSeconds.
	if err = dc.checkPausedConditions(d); err != nil {
		return err
	}

	if d.Spec.Paused {
		return dc.sync(d, rsList)
	}

	// rollback is not re-entrant in case the underlying replica sets are updated with a new
	// revision so we should ensure that we won't proceed to update replica sets until we
	// make sure that the deployment has cleaned up its rollback spec in subsequent enqueues.
	if getRollbackTo(d) != nil {
		return dc.rollback(d, rsList)
	}

	scalingEvent, err := dc.isScalingEvent(d, rsList)
	if err != nil {
		return err
	}
	if scalingEvent {
		return dc.sync(d, rsList)
	}

	switch d.Spec.Strategy.Type {
	case apps.RecreateDeploymentStrategyType:
		return dc.rolloutRecreate(d, rsList, podMap)
	case apps.RollingUpdateDeploymentStrategyType:
		return dc.rolloutRolling(d, rsList)
	}
	return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}
```








# 调用流程

```go
Main()
	- NewControllerManagerCommand
		- Run(c.Complete(), wait.NeverStop)
		- StartControllers
			- initFn(ctx)
			- startDeploymentController/startStatefulSetController
			- sts.NewStatefulSetController.Run/dc.NewDeploymentController.Run
			- pkg/controller。


```

基本流程如下：

- 构造controller manager option，并转化为Config对象，执行Run函数。
- 基于Config对象创建ControllerContext，其中包含InformerFactory。
- 基于ControllerContext运行各种controller，各种controller的定义在NewControllerInitializers中。
- 执行InformerFactory.Start。
- 每种controller都会构造自身的结构体并执行对应的Run函数。



# 参考

https://juejin.im/post/5caab97de51d452b3e52bec8
