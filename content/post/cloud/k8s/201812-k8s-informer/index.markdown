


K8S 各个组件经常用到的 `Informer` 机制（即 List-Watch），该部分代码主要位于 `client-go` 这个第三方包中。

# 技术架构

K8S 控制器的通用模型如下图：

<img alt="index-bfa766bf.png" src="images/index-bfa766bf.png" width="" height="" >

具体的处理流程如下：

<img alt="index-3ec91223.png" src="images/index-3ec91223.png" width="" height="" >

此部分的代码逻辑主要位于 `/vendor/k8s.io/client-go/tools/cache` 包中，代码目录结构如下：

```
ache
├── controller.go  # 包含：Config、Run、processLoop、NewInformer、NewIndexerInformer
├── delta_fifo.go  # 包含：NewDeltaFIFO、DeltaFIFO、AddIfNotPresent
├── expiration_cache.go
├── expiration_cache_fakes.go
├── fake_custom_store.go
├── fifo.go   # 包含：Queue、FIFO、NewFIFO
├── heap.go
├── index.go    # 包含：Indexer、MetaNamespaceIndexFunc
├── listers.go
├── listwatch.go   # 包含：ListerWatcher、ListWatch、List、Watch
├── mutation_cache.go
├── mutation_detector.go
├── reflector.go   # 包含：Reflector、NewReflector、Run、ListAndWatch
├── reflector_metrics.go
├── shared_informer.go  # 包含：NewSharedInformer、WaitForCacheSync、Run、HasSynced
├── store.go  # 包含：Store、MetaNamespaceKeyFunc、SplitMetaNamespaceKey
├── testing
│   ├── fake_controller_source.go
├── thread_safe_store.go  # 包含：ThreadSafeStore、threadSafeMap
├── undelta_store.go
```










SharedInformerFactory提供了公共的k8s对象的informers。

# 关键数据结构

```go
/ SharedInformerFactory provides shared informers for resources in all known
// API group versions.
type SharedInformerFactory interface {
	internalinterfaces.SharedInformerFactory
	ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
	WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

	Admissionregistration() admissionregistration.Interface
	Apps() apps.Interface
	Autoscaling() autoscaling.Interface
	Batch() batch.Interface
	Certificates() certificates.Interface
	Coordination() coordination.Interface
	Core() core.Interface
	Events() events.Interface
	Extensions() extensions.Interface
	Networking() networking.Interface
	Policy() policy.Interface
	Rbac() rbac.Interface
	Scheduling() scheduling.Interface
	Settings() settings.Interface
	Storage() storage.Interface
}
```

# 流程

## NewSharedInformerFactory

在 `CreateControllerContext` 创建  `SharedInformerFactory` interface 实现类的 `sharedInformerFactory` 的实例。

```go
// cmd/kube-controller-manager/app/controllermanager.go
func CreateControllerContext(s *config.CompletedConfig, rootClientBuilder, clientBuilder controller.ControllerClientBuilder, stop <-chan struct{}) (ControllerContext, error) {
	sharedInformers := informers.NewSharedInformerFactory(versionedClient, ResyncPeriod(s)())
 	...

	ctx := ControllerContext{
		InformerFactory:    sharedInformers,
	}

}
```

## InformerFactory.Start

`InformerFactory` 实际上是 `SharedInformerFactory`，具体的实现逻辑在client-go中的informer的实现机制。

在 `cmd/kube-controller-manager/app/controllermanager.go` 的 `Run()` 函数中调用了 `InformerFactory` 的 `Start` 方法。

```go
// cmd/kube-controller-manager/app/controllermanager.go
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
	...
	controllerContext.InformerFactory.Start(controllerContext.Stop)
	close(controllerContext.InformersStarted)
	...
}
```

Start 方法是 SharedInformerFactory 继承的 `internalinterfaces.SharedInformerFactory` 中的方法。


```go
// client-go/informers/internalinterfaces/factory_interfaces.go
type SharedInformerFactory interface {
	Start(stopCh <-chan struct{})
	InformerFor(obj runtime.Object, newFunc NewInformerFunc) cache.SharedIndexInformer
}
```

TODO ： 为什么这里要再拆分一个小的 SharedInformerFactory ??


最终调用的是 sharedInformerFactory.Start。Start方法初始化各种类型的informer，并且每个类型运行一个协程来跑 informer.Run()。

```go
// Start initializes all requested informers.
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()

	for informerType, informer := range f.informers {
		if !f.startedInformers[informerType] {
			go informer.Run(stopCh)
			f.startedInformers[informerType] = true
		}
	}
}
```

TODO: 这里 sharedInformerFactory.informers 是空的？？



### sharedIndexInformer.Run

sharedIndexInformer.Run具体运行了sharedIndexInformer的实现逻辑，主要的流程如下：

- 创建 DeltaFIFO
- 创建 Config，然后创建了 controller
- 开启两个协程，分别运行 cacheMutationDetector.Run 和 processor.run
- 最后 controller.Run() 运行主要的逻辑。

具体的代码如下：

```go
// client-go/tools/cache/shared_informer.go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, nil, s.indexer)

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,

		Process: s.HandleDeltas,
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
	s.controller.Run(stopCh)
}
```



#### NewDeltaFIFO

DeltaFIFO是一个对象变化的存储队列，依据先进先出的原则，process的函数接收该队列的Pop方法的输出对象来处理相关功能。

	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, nil, s.indexer)


#### Config

构造controller的配置文件，构造process，即HandleDeltas，该函数为后面使用到的process函数。

```go
cfg := &Config{
	Queue:            fifo,
	ListerWatcher:    s.listerWatcher,
	ObjectType:       s.objectType,
	FullResyncPeriod: s.resyncCheckPeriod,
	RetryOnError:     false,
	ShouldResync:     s.processor.shouldResync,

	Process: s.HandleDeltas,
}
```

#### controller

调用New(cfg)，构建sharedIndexInformer的controller。

```go
func() {
	s.startedLock.Lock()
	defer s.startedLock.Unlock()

	s.controller = New(cfg)
	s.controller.(*controller).clock = s.clock
	s.started = true
}()
```

#### cacheMutationDetector.Run
调用s.cacheMutationDetector.Run，检查缓存对象是否变化。

	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)

1. defaultCacheMutationDetector.Run

	```go
	func (d *defaultCacheMutationDetector) Run(stopCh <-chan struct{}) {
		// we DON'T want protection from panics.  If we're running this code, we want to die
		for {
			d.CompareObjects()

			select {
			case <-stopCh:
				return
			case <-time.After(d.period):
			}
		}
	}
	```

2. CompareObjects

比较缓存的跟备份的是否一致，如果不一致就 `panic`，该协程将退出。

```go
func (d *defaultCacheMutationDetector) CompareObjects() {
	d.lock.Lock()
	defer d.lock.Unlock()

	altered := false
	for i, obj := range d.cachedObjs {
		if !reflect.DeepEqual(obj.cached, obj.copied) {
			fmt.Printf("CACHE %s[%d] ALTERED!\n%v\n", d.name, i, diff.ObjectDiff(obj.cached, obj.copied))
			altered = true
		}
	}

	if altered {
		msg := fmt.Sprintf("cache %s modified", d.name)
		if d.failureFunc != nil {
			d.failureFunc(msg)
			return
		}
		panic(msg)
	}
}
```

TODO 协程遇到 panic 将退出？


#### processor.run

调用 `s.processor.run`，将调用 `sharedProcessor.run`，会调用 `Listener.run`和 `Listener.pop`,执行处理 queue 的函数。

	wg.StartWithChannel(processorStopCh, s.processor.run)

1. sharedProcessor.Run

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		for _, listener := range p.listeners {
			p.wg.Start(listener.run)
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
	<-stopCh
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()
	for _, listener := range p.listeners {
		close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
	}
	p.wg.Wait() // Wait for all .pop() and .run() to stop
}
```

2. listener.run

#### controller.Run

调用 `s.controller.Run`，构建 `Reflector` ，进行对 etcd 的缓存

```go
defer func() {
	s.startedLock.Lock()
	defer s.startedLock.Unlock()
	s.stopped = true // Don't want any new listeners
}()
s.controller.Run(stopCh)
```

```go
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	// 构建Reflector
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.clock = c.clock

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
	defer wg.Wait()
	// 运行Reflector
	wg.StartWithChannel(stopCh, r.Run)

	// 执行processLoop
	wait.Until(c.processLoop, time.Second, stopCh)
}
```

## Reflector

### Reflector

Reflector 的主要作用是 watch 指定的 k8s 资源，并将变化同步到本地是 store 中。Reflector只会放置指定的expectedType类型的资源到store中，除非expectedType为nil。如果resyncPeriod不为零，那么Reflector为以resyncPeriod为周期定期执行list的操作，这样就可以使用Reflector来定期处理所有的对象，也可以逐步处理变化的对象。

```go
type Reflector struct {
	name string
	metrics *reflectorMetrics
	// 期望放入缓存store的资源类型。
	expectedType reflect.Type
	// The destination to sync up with the watch source
	// watch的资源对应的本地缓存。
	store Store
	// listerWatcher is used to perform lists and watches.
	listerWatcher ListerWatcher
	// period controls timing between one watch ending and
	// the beginning of the next one.
	period       time.Duration
	resyncPeriod time.Duration
	ShouldResync func() bool
	// clock allows tests to manipulate time
	clock clock.Clock
	// lastSyncResourceVersion is the resource version token last
	// observed when doing a sync with the underlying store
	// it is thread safe, but not synchronized with the underlying store
	lastSyncResourceVersion string
	// lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
	lastSyncResourceVersionMutex sync.RWMutex
}
```


### Reflector.Run

Reflector.Run 主要执行了 ListAndWatch 的方法。

```go
func (r *Reflector) Run(stopCh <-chan struct{}) {
	klog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedType, r.resyncPeriod, r.name)
	wait.Until(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			utilruntime.HandleError(err)
		}
	}, r.period, stopCh)
}
```

### ListAndWatch

ListAndWatch第一次会列出所有的对象，并获取资源对象的版本号，然后watch资源对象的版本号来查看是否有被变更。首先会将资源版本号设置为0，list()可能会导致本地的缓存相对于etcd里面的内容存在延迟，Reflector会通过watch的方法将延迟的部分补充上，使得本地的缓存数据与etcd的数据保持一致。


```go

// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
// It returns error if ListAndWatch didn't even try to initialize watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	klog.V(3).Infof("Listing and watching %v from %s", r.expectedType, r.name)
	var resourceVersion string

	// Explicitly set "0" as resource version - it's fine for the List()
	// to be served from cache and potentially be delayed relative to
	// etcd contents. Reflector framework will catch up via Watch() eventually.
	// 首先将资源的版本号设置为0，然后通过 List 列出所有的资源
	options := metav1.ListOptions{ResourceVersion: "0"}
	list, err := r.listerWatcher.List(options)
	if err != nil {
		return fmt.Errorf("%s: Failed to list %v: %v", r.name, r.expectedType, err)
	}
	// 获取资源版本号，并将list的内容提取成对象列表。
	listMetaInterface, err := meta.ListAccessor(list)
	if err != nil {
		return fmt.Errorf("%s: Unable to understand list result %#v: %v", r.name, list, err)
	}
	resourceVersion = listMetaInterface.GetResourceVersion()
	items, err := meta.ExtractList(list)
	if err != nil {
		return fmt.Errorf("%s: Unable to understand list result %#v (%v)", r.name, list, err)
	}
	// 将list中对象列表的内容和版本号存储到本地的缓存store中，并全量替换已有的store的内容。
	if err := r.syncWith(items, resourceVersion); err != nil {
		return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
	}

	// 最后设置最新的资源版本号。
	r.setLastSyncResourceVersion(resourceVersion)

	// 周期性调用 DeltaFIFO.Resync 进行同步
	resyncerrc := make(chan error, 1)
	cancelCh := make(chan struct{})
	defer close(cancelCh)
	go func() {
		resyncCh, cleanup := r.resyncChan()
		defer func() {
			cleanup() // Call the last one written into cleanup
		}()
		for {
			select {
			case <-resyncCh:
			case <-stopCh:
				return
			case <-cancelCh:
				return
			}
			if r.ShouldResync == nil || r.ShouldResync() {
				klog.V(4).Infof("%s: forcing resync", r.name)
				if err := r.store.Resync(); err != nil {
					resyncerrc <- err
					return
				}
			}
			cleanup()
			resyncCh, cleanup = r.resyncChan()
		}
	}()

	for {
		// give the stopCh a chance to stop the loop, even in case of continue statements further down on errors
		select {
		case <-stopCh:
			return nil
		default:
		}

		// 设置watch的超时时间，默认为5~10分钟。
		timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
		options = metav1.ListOptions{
			ResourceVersion: resourceVersion,
			// We want to avoid situations of hanging watchers. Stop any wachers that do not
			// receive any events within the timeout window.
			TimeoutSeconds: &timeoutSeconds,
		}

		w, err := r.listerWatcher.Watch(options)
		if err != nil {
			switch err {
			case io.EOF:
				// watch closed normally
			case io.ErrUnexpectedEOF:
				klog.V(1).Infof("%s: Watch for %v closed with unexpected EOF: %v", r.name, r.expectedType, err)
			default:
				utilruntime.HandleError(fmt.Errorf("%s: Failed to watch %v: %v", r.name, r.expectedType, err))
			}
			// If this is "connection refused" error, it means that most likely apiserver is not responsive.
			// It doesn't make sense to re-list all objects because most likely we will be able to restart
			// watch where we ended.
			// If that's the case wait and resend watch request.
			if urlError, ok := err.(*url.Error); ok {
				if opError, ok := urlError.Err.(*net.OpError); ok {
					if errno, ok := opError.Err.(syscall.Errno); ok && errno == syscall.ECONNREFUSED {
						time.Sleep(time.Second)
						continue
					}
				}
			}
			return nil
		}

		if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
			if err != errorStopRequested {
				klog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedType, err)
			}
			return nil
		}
	}
}
```



#### store.Resync

定时对象同步，store的具体对象为DeltaFIFO，即调用DeltaFIFO.Resync。

```go
// client-go/tools/cache/delta_fifo.go
// Resync will send a sync event for each item
func (f *DeltaFIFO) Resync() error {
	f.lock.Lock()
	defer f.lock.Unlock()

	if f.knownObjects == nil {
		return nil
	}

	keys := f.knownObjects.ListKeys()
	for _, k := range keys {
		if err := f.syncKeyLocked(k); err != nil {
			return err
		}
	}
	return nil
}
```

#### watchHandler

watchHandler主要是通过watch的方式保证当前的资源版本是最新的。


```go
// watchHandler watches w and keeps *resourceVersion up to date.
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
	start := r.clock.Now()
	eventCount := 0

	// Stopping the watcher should be idempotent and if we return from this function there's no way
	// we're coming back in with the same watch interface.
	defer w.Stop()

loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		case event, ok := <-w.ResultChan():
			// 获取watch接口中的事件的channel，来获取事件的内容。

			if !ok {
				break loop
			}
			if event.Type == watch.Error {
				return apierrs.FromObject(event.Object)
			}
			if e, a := r.expectedType, reflect.TypeOf(event.Object); e != nil && e != a {
				utilruntime.HandleError(fmt.Errorf("%s: expected type %v, but watch event object had type %v", r.name, e, a))
				continue
			}
			meta, err := meta.Accessor(event.Object)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
				continue
			}
			newResourceVersion := meta.GetResourceVersion()
			// 当获得添加、更新、删除的事件时，将对应的对象更新到本地缓存store中。
			switch event.Type {
			case watch.Added:
				err := r.store.Add(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", r.name, event.Object, err))
				}
			case watch.Modified:
				err := r.store.Update(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", r.name, event.Object, err))
				}
			case watch.Deleted:
				// TODO: Will any consumers need access to the "last known
				// state", which is passed in event.Object? If so, may need
				// to change this.
				err := r.store.Delete(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", r.name, event.Object, err))
				}
			default:
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
			}

			// 更新当前的最新版本号。
			*resourceVersion = newResourceVersion
			r.setLastSyncResourceVersion(newResourceVersion)
			eventCount++
		}
	}

	watchDuration := r.clock.Now().Sub(start)
	if watchDuration < 1*time.Second && eventCount == 0 {
		return fmt.Errorf("very short watch: %s: Unexpected watch close - watch lasted less than a second and no items received", r.name)
	}
	klog.V(4).Infof("%s: Watch close - %v total %v items received", r.name, r.expectedType, eventCount)
	return nil
}
```

通过对Reflector模块的分析，可以看到多次使用到本地缓存store模块，而store的数据由DeltaFIFO赋值而来，以下针对DeltaFIFO和store做分析。

## DeltaFIFO

DeltaFIFO是一个生产者与消费者的队列，其中Reflector是生产者，消费者调用Pop()的方法。

DeltaFIFO主要用在以下场景：

- 希望对象变更最多处理一次
- 处理对象时，希望查看自上次处理对象以来发生的所有事情
- 要处理对象的删除
- 希望定期重新处理对象


```go
type DeltaFIFO struct {
	// lock/cond protects access to 'items' and 'queue'.
	lock sync.RWMutex
	cond sync.Cond

	// We depend on the property that items in the set are in
	// the queue and vice versa, and that all Deltas in this
	// map have at least one Delta.
	items map[string]Deltas
	queue []string

	// populated is true if the first batch of items inserted by Replace() has been populated
	// or Delete/Add/Update was called first.
	populated bool
	// initialPopulationCount is the number of items inserted by the first call of Replace()
	initialPopulationCount int

	// keyFunc is used to make the key used for queued item
	// insertion and retrieval, and should be deterministic.
	keyFunc KeyFunc

	// knownObjects list keys that are "known", for the
	// purpose of figuring out which items have been deleted
	// when Replace() or Delete() is called.
	knownObjects KeyListerGetter

	// Indication the queue is closed.
	// Used to indicate a queue is closed so a control loop can exit when a queue is empty.
	// Currently, not used to gate any of CRED operations.
	closed     bool
	closedLock sync.Mutex
}
```




DeltaFIFO由NewDeltaFIFO初始化，并赋值给config.Queue。

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, nil, s.indexer)

	cfg := &Config{
		Queue:            fifo,
		...
	}
    ...
}
```

### NewDeltaFIFO

```go
// client-go/tools/cache/delta_fifo.go
// NewDeltaFIFO returns a Store which can be used process changes to items.
//
// keyFunc is used to figure out what key an object should have. (It's
// exposed in the returned DeltaFIFO's KeyOf() method, with bonus features.)
//
// 'keyLister' is expected to return a list of keys that the consumer of
// this queue "knows about". It is used to decide which items are missing
// when Replace() is called; 'Deleted' deltas are produced for these items.
// It may be nil if you don't need to detect all deletions.
// TODO: consider merging keyLister with this object, tracking a list of
//       "known" keys when Pop() is called. Have to think about how that
//       affects error retrying.
// NOTE: It is possible to misuse this and cause a race when using an
//       external known object source.
//       Whether there is a potential race depends on how the comsumer
//       modifies knownObjects. In Pop(), process function is called under
//       lock, so it is safe to update data structures in it that need to be
//       in sync with the queue (e.g. knownObjects).
//
//       Example:
//       In case of sharedIndexInformer being a consumer
//       (https://github.com/kubernetes/kubernetes/blob/0cdd940f/staging/
//       src/k8s.io/client-go/tools/cache/shared_informer.go#L192),
//       there is no race as knownObjects (s.indexer) is modified safely
//       under DeltaFIFO's lock. The only exceptions are GetStore() and
//       GetIndexer() methods, which expose ways to modify the underlying
//       storage. Currently these two methods are used for creating Lister
//       and internal tests.
//
// Also see the comment on DeltaFIFO.
func NewDeltaFIFO(keyFunc KeyFunc, knownObjects KeyListerGetter) *DeltaFIFO {
	f := &DeltaFIFO{
		items:        map[string]Deltas{},
		queue:        []string{},
		keyFunc:      keyFunc,
		knownObjects: knownObjects,
	}
	f.cond.L = &f.lock
	return f
}
```

controller.Run的部分调用了NewReflector。

```go

func (c *controller) Run(stopCh <-chan struct{}) {
	...
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
    ...
}
```

NewReflector构造函数，将c.config.Queue赋值给Reflector.store的属性。

```go
func NewReflector(lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
	return NewNamedReflector(getDefaultReflectorName(internalPackages...), lw, expectedType, store, resyncPeriod)
}

// NewNamedReflector same as NewReflector, but with a specified name for logging
func NewNamedReflector(name string, lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
	reflectorSuffix := atomic.AddInt64(&reflectorDisambiguator, 1)
	r := &Reflector{
		name: name,
		// we need this to be unique per process (some names are still the same)but obvious who it belongs to
		metrics:       newReflectorMetrics(makeValidPromethusMetricLabel(fmt.Sprintf("reflector_"+name+"_%d", reflectorSuffix))),
		listerWatcher: lw,
		store:         store,
		expectedType:  reflect.TypeOf(expectedType),
		period:        time.Second,
		resyncPeriod:  resyncPeriod,
		clock:         &clock.RealClock{},
	}
	return r
}
```

### Queue & Store

DeltaFIFO的类型是Queue接口，Reflector.store是Store接口，Queue接口是一个存储队列，Process的方法执行Queue.Pop出来的数据对象

```go
// Queue is exactly like a Store, but has a Pop() method too.
type Queue interface {
	Store

	// Pop blocks until it has something to process.
	// It returns the object that was process and the result of processing.
	// The PopProcessFunc may return an ErrRequeue{...} to indicate the item
	// should be requeued before releasing the lock on the queue.
	Pop(PopProcessFunc) (interface{}, error)

	// AddIfNotPresent adds a value previously
	// returned by Pop back into the queue as long
	// as nothing else (presumably more recent)
	// has since been added.
	AddIfNotPresent(interface{}) error

	// Return true if the first batch of items has been popped
	HasSynced() bool

	// Close queue
	Close()
}
```

## store

Store是一个通用的存储接口，Reflector通过watch server的方式更新数据到store中，store给Reflector提供本地的缓存，让Reflector可以像消息队列一样的工作。

Store实现的是一种可以准确的写入对象和获取对象的机制。

```go
// Store is a generic object storage interface. Reflector knows how to watch a server
// and update a store. A generic store is provided, which allows Reflector to be used
// as a local caching system, and an LRU store, which allows Reflector to work like a
// queue of items yet to be processed.
//
// Store makes no assumptions about stored object identity; it is the responsibility
// of a Store implementation to provide a mechanism to correctly key objects and to
// define the contract for obtaining objects by some arbitrary key type.
type Store interface {
	Add(obj interface{}) error
	Update(obj interface{}) error
	Delete(obj interface{}) error
	List() []interface{}
	ListKeys() []string
	Get(obj interface{}) (item interface{}, exists bool, err error)
	GetByKey(key string) (item interface{}, exists bool, err error)

	// Replace will delete the contents of the store, using instead the
	// given list. Store takes ownership of the list, you should not reference
	// it after calling this function.
	Replace([]interface{}, string) error
	Resync() error
}
```

其中Replace方法会删除原来store中的内容，并将新增的list的内容存入store中，即完全替换数据。

### cache

cache实现了store的接口，而cache的具体实现又是调用ThreadSafeStore接口来实现功能的。

cache的功能主要有以下两点：

- 通过keyFunc计算对象的key
- 调用ThreadSafeStorage接口的方法

```go
// cache responsibilities are limited to:
//	1. Computing keys for objects via keyFunc
//  2. Invoking methods of a ThreadSafeStorage interface
type cache struct {
	// cacheStorage bears the burden of thread safety for the cache
	cacheStorage ThreadSafeStore
	// keyFunc is used to make the key for objects stored in and retrieved from items, and
	// should be deterministic.
	keyFunc KeyFunc
}
```

其中ListAndWatch主要用到以下的方法：

**cache.Replace**

```go
// Replace will delete the contents of 'c', using instead the given list.
// 'c' takes ownership of the list, you should not reference the list again
// after calling this function.
func (c *cache) Replace(list []interface{}, resourceVersion string) error {
	items := map[string]interface{}{}
	for _, item := range list {
		key, err := c.keyFunc(item)
		if err != nil {
			return KeyError{item, err}
		}
		items[key] = item
	}
	c.cacheStorage.Replace(items, resourceVersion)
	return nil
}
```

类似的还有

**cache.Add**
**cache.Update**
**cache.Delete**


### ThreadSafeStore
cache的具体是调用ThreadSafeStore来实现的。

```go
type ThreadSafeStore interface {
	Add(key string, obj interface{})
	Update(key string, obj interface{})
	Delete(key string)
	Get(key string) (item interface{}, exists bool)
	List() []interface{}
	ListKeys() []string
	Replace(map[string]interface{}, string)
	Index(indexName string, obj interface{}) ([]interface{}, error)
	IndexKeys(indexName, indexKey string) ([]string, error)
	ListIndexFuncValues(name string) []string
	ByIndex(indexName, indexKey string) ([]interface{}, error)
	GetIndexers() Indexers

	// AddIndexers adds more indexers to this store.  If you call this after you already have data
	// in the store, the results are undefined.
	AddIndexers(newIndexers Indexers) error
	Resync() error
}
```

**threadSafeMap**

```go
// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}

	// indexers maps a name to an IndexFunc
	indexers Indexers
	// indices maps a name to an Index
	indices Indices
}
```

## processLoop

```go
func (c *controller) Run(stopCh <-chan struct{}) {
	...
	wait.Until(c.processLoop, time.Second, stopCh)
}
```

在controller.Run方法中会调用processLoop，以下分析processLoop的处理逻辑。


```go
// processLoop drains the work queue.
// TODO: Consider doing the processing in parallel. This will require a little thought
// to make sure that we don't end up processing the same object multiple times
// concurrently.
//
// TODO: Plumb through the stopCh here (and down to the queue) so that this can
// actually exit when the controller is stopped. Or just give up on this stuff
// ever being stoppable. Converting this whole package to use Context would
// also be helpful.
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == FIFOClosedError {
				return
			}
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```

processLoop主要处理任务队列中的任务，其中处理逻辑是调用具体的ProcessFunc函数来实现.

### DeltaFIFO.Pop

Pop会阻塞住直到队列里面添加了新的对象，如果有多个对象，按照先进先出的原则处理，如果某个对象没有处理成功会重新被加入该队列中。

Pop中会调用具体的process函数来处理对象。

```go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		for len(f.queue) == 0 {
			// When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
			// When Close() is called, the f.closed is set and the condition is broadcasted.
			// Which causes this loop to continue and return from the Pop().
			if f.IsClosed() {
				return nil, FIFOClosedError
			}

			f.cond.Wait()
		}
		id := f.queue[0]
		f.queue = f.queue[1:]
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}
		item, ok := f.items[id]
		if !ok {
			// Item may have been deleted subsequently.
			continue
		}
		delete(f.items, id)
		err := process(item)
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		// Don't need to copyDeltas here, because we're transferring
		// ownership to the caller.
		return item, err
	}
}
```

### HandleDeltas


```go
cfg := &Config{
	Queue:            fifo,
	ListerWatcher:    s.listerWatcher,
	ObjectType:       s.objectType,
	FullResyncPeriod: s.resyncCheckPeriod,
	RetryOnError:     false,
	ShouldResync:     s.processor.shouldResync,

	Process: s.HandleDeltas,
}
```

其中process函数就是在sharedIndexInformer.Run方法中，给config.Process赋值的HandleDeltas函数。


```go

func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	// from oldest to newest
	for _, d := range obj.(Deltas) {
		switch d.Type {
		case Sync, Added, Updated:
			isSync := d.Type == Sync
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, isSync)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```

根据不同的类型，更新 index，然后调用 `processor.distribute` 方法，该方法将对象加入processorListener的channel中。然后在 processorListener.Run 中使用。


# 7. processor


processor的主要功能就是记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。在sharedIndexInformer.Run部分会调用processor.run。



流程：

- processorListener 的 add 函数负责将 notify 装进 pendingNotifications。
- pop函数取出pendingNotifications的第一个nofify,输出到nextCh channel。
- run函数则负责取出notify，然后根据notify的类型(增加、删除、更新)触发相应的处理函数，这些函数是在不同的 `NewXxxcontroller` 实现中注册的。


```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	...
	wg.StartWithChannel(processorStopCh, s.processor.run)
	...
}
```



## 7.1 sharedProcessor.Run

sharedProcessor.Run 主要是开两个协程调用 listener.run 和 listener.pop

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
   func() {
      p.listenersLock.RLock()
      defer p.listenersLock.RUnlock()
      for _, listener := range p.listeners {
         p.wg.Start(listener.run)
         p.wg.Start(listener.pop)
      }
   }()
   <-stopCh
   p.listenersLock.RLock()
   defer p.listenersLock.RUnlock()
   for _, listener := range p.listeners {
      close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
   }
   p.wg.Wait() // Wait for all .pop() and .run() to stop
}
```


### 7.1.1 listener.pop

pop函数取出pendingNotifications的第一个nofify,输出到nextCh channel。

```go
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
		case nextCh <- notification:
			// Notification dispatched
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil // Disable this select case
			}
		case notificationToAdd, ok := <-p.addCh:
			if !ok {
				return
			}
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
```

### 7.1.2. listener.run

listener.run部分根据不同的更新类型调用不同的处理函数。


```go
func (p *processorListener) run() {
	stopCh := make(chan struct{})
	wait.Until(func() {
		// this gives us a few quick retries before a long pause and then a few more quick retries
		err := wait.ExponentialBackoff(retry.DefaultRetry, func() (bool, error) {
			for next := range p.nextCh {
				switch notification := next.(type) {
				case updateNotification:
					p.handler.OnUpdate(notification.oldObj, notification.newObj)
				case addNotification:
					p.handler.OnAdd(notification.newObj)
				case deleteNotification:
					p.handler.OnDelete(notification.oldObj)
				default:
					utilruntime.HandleError(fmt.Errorf("unrecognized notification: %#v", next))
				}
			}
			// the only way to get here is if the p.nextCh is empty and closed
			return true, nil
		})

		// the only way to get here is if the p.nextCh is empty and closed
		if err == nil {
			close(stopCh)
		}
	}, 1*time.Minute, stopCh)
}
```

其中具体的实现函数handler是在NewDeploymentController（其他不同类型的controller类似）中赋值的，而该handler是一个接口，具体如下：


```go
type ResourceEventHandler interface {
	OnAdd(obj interface{})
	OnUpdate(oldObj, newObj interface{})
	OnDelete(obj interface{})
}
```


### 7.2 ResourceEventHandler

以下以DeploymentController的处理逻辑为例。

在NewDeploymentController部分会注册deployment的事件函数，以下注册了三种类型的事件函数，其中包括：dInformer、rsInformer和podInformer。

```go
// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer extensionsinformers.DeploymentInformer, rsInformer extensionsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
	...
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
    ...
}
```

rsInformer.Informer().AddEventHandler 调用的是 sharedIndexInformer.AddEventHandler()

```go
func (s *sharedIndexInformer) AddEventHandler(handler ResourceEventHandler) {
	s.AddEventHandlerWithResyncPeriod(handler, s.defaultEventHandlerResyncPeriod)
}

func (s *sharedIndexInformer) AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) {
	...
	listener := newProcessListener(handler, resyncPeriod, determineResyncPeriod(resyncPeriod, s.resyncCheckPeriod), s.clock.Now(), initialBufferSize)

	if !s.started {
		s.processor.addListener(listener)
		return
	}

	// in order to safely join, we have to
	// 1. stop sending add/update/delete notifications
	// 2. do a list against the store
	// 3. send synthetic "Add" events to the new handler
	// 4. unblock
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	s.processor.addListener(listener)
	for _, item := range s.indexer.List() {
		listener.add(addNotification{newObj: item})
	}
}
```


在 AddEventHandlerWithResyncPeriod 中调用 newProcessListener 创建 processorListener，赋值 ResourceEventHandler 为 handler。

在 `sharedProcesser.addListener()` 中将这个 listener 加入到 list 中，如果 sharedProcesser 已经开始运行，就运行 listener.pop 和 run。在 run 中调用 handler 的处理函数（AddFunc、UpdateFunc、DeleteFunc）。


### 7.2.1 addDeployment

以下以addDeployment为例，addDeployment主要是将对象加入到enqueueDeployment的队列中。

```go
func (dc *DeploymentController) addDeployment(obj interface{}) {
	d := obj.(*extensions.Deployment)
	glog.V(4).Infof("Adding deployment %s", d.Name)
	dc.enqueueDeployment(d)
}
```

enqueueDeployment的定义

```go
type DeploymentController struct {
	...
	enqueueDeployment func(deployment *extensions.Deployment)
    ...
}
```

将dc.enqueue赋值给dc.enqueueDeployment

	dc.enqueueDeployment = dc.enqueue

dc.enqueue调用了dc.queue.Add(key)

```go
func (dc *DeploymentController) enqueue(deployment *apps.Deployment) {
	key, err := controller.KeyFunc(deployment)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Couldn't get key for object %#v: %v", deployment, err))
		return
	}

	dc.queue.Add(key)
}
```

dc.queue主要记录了需要被同步的deployment的对象，供syncDeployment使用。

```go
dc := &DeploymentController{
	...
	queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
}
```

NewNamedRateLimitingQueue

```go
func NewNamedRateLimitingQueue(rateLimiter RateLimiter, name string) RateLimitingInterface {
	return &rateLimitingType{
		DelayingInterface: NewNamedDelayingQueue(name),
		rateLimiter:       rateLimiter,
	}
}
```

通过以上分析，可以看出processor记录了不同类似的事件函数，其中事件函数在NewXxxController构造函数部分注册，具体事件函数的处理，一般是将需要处理的对象加入对应的controller的任务队列中，然后由类似syncDeployment的同步函数来维持期望状态的同步逻辑。



# 8. 总结

本文分析的部分主要是k8s的informer机制，即List-Watch机制。

## 8.1. Reflector

Reflector的主要作用是watch指定的k8s资源，并将变化同步到本地是store中。Reflector只会放置指定的expectedType类型的资源到store中，除非expectedType为nil。如果resyncPeriod不为零，那么Reflector为以resyncPeriod为周期定期执行list的操作，这样就可以使用Reflector来定期处理所有的对象，也可以逐步处理变化的对象。

## 8.2. ListAndWatch

ListAndWatch第一次会列出所有的对象，并获取资源对象的版本号，然后watch资源对象的版本号来查看是否有被变更。首先会将资源版本号设置为0，list()可能会导致本地的缓存相对于etcd里面的内容存在延迟，Reflector会通过watch的方法将延迟的部分补充上，使得本地的缓存数据与etcd的数据保持一致。

## 8.3. DeltaFIFO

DeltaFIFO是一个生产者与消费者的队列，其中Reflector是生产者，消费者调用Pop()的方法。

DeltaFIFO主要用在以下场景：

- 希望对象变更最多处理一次
- 处理对象时，希望查看自上次处理对象以来发生的所有事情
- 要处理对象的删除
- 希望定期重新处理对象

## 8.4. store

Store是一个通用的存储接口，Reflector通过watch server的方式更新数据到store中，store给Reflector提供本地的缓存，让Reflector可以像消息队列一样的工作。

Store实现的是一种可以准确的写入对象和获取对象的机制。

## 8.5. processor

processor的主要功能就是记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。在sharedIndexInformer.Run部分会调用processor.run。

流程：

- listenser的add函数负责将notify装进pendingNotifications。
- pop函数取出pendingNotifications的第一个nofify,输出到nextCh channel。
- run函数则负责取出notify，然后根据notify的类型(增加、删除、更新)触发相应的处理函数，这些函数是在不同的NewXxxcontroller实现中注册的。
- processor记录了不同类似的事件函数，其中事件函数在NewXxxController构造函数部分注册，具体事件函数的处理，一般是将需要处理的对象加入对应的controller的任务队列中，然后由类似syncDeployment的同步函数来维持期望状态的同步逻辑。

## 8.6. 主要步骤

1. 在controller-manager的Run函数部分调用了InformerFactory.Start的方法，Start方法初始化各种类型的informer，并且每个类型起了个informer.Run的goroutine。
2. informer.Run的部分先生成一个DeltaFIFO的队列来存储对象变化的数据。然后调用processor.Run和controller.Run函数。
3. controller.Run函数会生成一个Reflector，Reflector的主要作用是watch指定的k8s资源，并将变化同步到本地是store中。Reflector以resyncPeriod为周期定期执行list的操作，这样就可以使用Reflector来定期处理所有的对象，也可以逐步处理变化的对象。
4. Reflector接着执行ListAndWatch函数，ListAndWatch第一次会列出所有的对象，并获取资源对象的版本号，然后watch资源对象的版本号来查看是否有被变更。首先会将资源版本号设置为0，list()可能会导致本地的缓存相对于etcd里面的内容存在延迟，Reflector会通过watch的方法将延迟的部分补充上，使得本地的缓存数据与etcd的数据保持一致。
5. controller.Run函数还会调用processLoop函数，processLoop通过调用HandleDeltas，再调用distribute，processorListener.add最终将不同更新类型的对象加入processorListener的channel中，供processorListener.Run使用。
6. processor的主要功能就是记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。processor记录了不同类型的事件函数，其中事件函数在NewXxxController构造函数部分注册，具体事件函数的处理，一般是将需要处理的对象加入对应的controller的任务队列中，然后由类似syncDeployment的同步函数来维持期望状态的同步逻辑。




# 流程


```go

controller.Run()
  controller.processLoop()
    DeltaFIFO.Pop(PopProcessFunc(c.config.Process))
      sharedIndexInformer.HandleDeltas()
	  processor.distribute() 将更新加入到 processorListener 的 channel 中，在 processorListener.Run 中使用



sharedIndexInformer.Run()
  s.processor.run() 起两个协程分别调用 listener.run 和 listener.pop
    listener.run() 调用 XXController 注册的回调函数（OnAdd、OnUpdate、OnDelete）


```


注册 handle 的流程

```
NewDeploymentController()
  dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{})  将 addDeployment、updateDeployment 等传进去。
    newProcessListener()   创建 listener，赋值 handle 字段。
	listener.run()  在 sharedProcessor 就绪后调用 run 函数运行。对象更新时调用 addDeployment、updateDeployment。

```



创建 sharedIndexInformer 的流程









# 问题

1. DeploymentInformer 的作用？？？

2. controller KeyFunc() 的作用？



# 参考
https://www.huweihuang.com/article/source-analysis/kube-controller-manager/sharedIndexInformer/
https://cizixs.com/2018/06/25/kubernetes-resource-management/
