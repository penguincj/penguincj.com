---
title: "Kubernetes Event 事件源码解析"
subtitle: "Kubernetes 事件资源对象的来龙去脉"
summary: "Kubernetes 事件用于记录各组件运行过程中的调试信息"
date: "2019-08-17"
draft: false
tags: ["Kubernetes"]
---


Kubernetes 是一个复杂的分布式应用，组件之间通过 apiserver 获取资源对象的运行态，除此之外还要知道系统中的某个状态变迁和一些其他的事件。事件一般用于调试，用户可以通过 kubectl 命令获取整个集群或者某个 pod 的事件信息。kubectl get events 可以看到所有的事件，kubectl describe pod PODNAME 能看到关于某个 pod 的事件。对于前者很好理解，kubectl 会直接访问 apiserver 的 event 资源，而对于后者 kubectl 还根据 pod 的名字进行搜索，匹配 InvolvedObject 名称和 pod 名称匹配的事件。

事件主要是通过 record 包来实现，跟 `kubectl create -f nginx.yaml --record` 中的 `--record` 标志没有关系，设置这个标志为 true 可以在 annotation 中记录当前命令创建或者升级了该资源。

# 代码架构

```go
staging/src/k8s.io/client-go/tools/record/
├── event.go  // EventBroadcaster EventRecorder EventSink 用户操作广播器、记录器的接口，recorderImpl，eventBroadcasterImpl 是其实现。
└── events_cache.go // EventAggregator -eventLogger EventCorrelator，缓存相关的聚合、过滤

staging/src/k8s.io/client-go/kubernetes/typed/core/v1/
├── event.go // EventInterface events 实现向 apiserver 写入事件，EventsGetter 返回 EventInterface
├── event_expansion.go // +EventSinkImpl 用于触发方保存事件，+EventExpansion 是 EventInterface 的扩展。
└── core_client.go // CoreV1Interface 和 CoreV1Client 表示 V1 的客户端

staging/src/k8s.io/apimachinery/pkg/watch/
├── mux.go // Broadcaster 向 broadcasterWatcher 分发消息
└── watch.go // Event 事件，Interface 是监控和报告事件的接口，由 broadcasterWatcher 实现

vendor/k8s.io/apimachinery/pkg/runtime/
└── interfaces.go // Object 所有资源对象的接口，事件也是一种资源对象

staging/src/k8s.io/api/core/v1/
└── types.go // 定义了很多 Object 的类型，包括 Event
```


# 关键的结构

## Event

Event 表示一个事件，主要有事件类型和资源对象组成

```go
// Event represents a single event to a watched resource.
// +k8s:deepcopy-gen=true
type Event struct {
	Type EventType

	// Object is:
	//  * If Type is Added or Modified: the new state of the object.
	//  * If Type is Deleted: the state of the object immediately before deletion.
	//  * If Type is Error: *api.Status is recommended; other types may make sense
	//    depending on context.
	Object runtime.Object
}
```

用 EventSource 表示事件的发生来源，是由谁在哪个节点触发的。 Component 是 agent 的名字，这里是 `"kube-controller-manager"`

```go
type EventSource struct {
	// Component from which the event is generated.
	// +optional
	Component string `json:"component,omitempty" protobuf:"bytes,1,opt,name=component"`
	// Node name on which the event is generated.
	// +optional
	Host string `json:"host,omitempty" protobuf:"bytes,2,opt,name=host"`
}
```

向 apiserver 写入的底层的事件资源对象表示如下：

```go
// staging/src/k8s.io/api/core/v1/types.go
// Event is a report of an event somewhere in the cluster.
type Event struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata
	metav1.ObjectMeta `json:"metadata" protobuf:"bytes,1,opt,name=metadata"`

	// The object that this event is about.
	InvolvedObject ObjectReference `json:"involvedObject" protobuf:"bytes,2,opt,name=involvedObject"`

	// This should be a short, machine understandable string that gives the reason
	// for the transition into the object's current status.
	// TODO: provide exact specification for format.
	// +optional
	Reason string `json:"reason,omitempty" protobuf:"bytes,3,opt,name=reason"`
 	..
}
```

Event 主要的字段：

- unversioned.TypeMeta，所有 Kubernetes 都有的，资源的类型和版本，对应了 yaml 文件的 Kind 和 apiVersion 字段
- ObjectMera，资源的元数据，比如 name、nemspace、labels、uuid、创建时间等
- 和事件本身息息相关的字段，比如事件消息、来源、类型，以及数量，和第一个事件的发生的时间等。
- InvolvedObject，指向了和事件关联的对象，如果是启动容器的事件，这个对象就是 Pod。

```json
{
    "apiVersion": "v1",
    "count": 2,
    "eventTime": null,
    "firstTimestamp": "2019-11-06T06:05:41Z",
    "involvedObject": {
        "apiVersion": "v1",
        "fieldPath": "spec.containers{busybox}",
        "kind": "Pod",
        "name": "busybox",
        "namespace": "default",
        "resourceVersion": "9543458",
        "uid": "82881b65-9261-11e9-b8c7-f0f279c36caf"
    },
    "kind": "Event",
    "lastTimestamp": "2019-11-06T07:05:42Z",
    "message": "Container image \"busybox\" already present on machine",
    "metadata": {
        "creationTimestamp": "2019-11-06T06:05:53Z",
        "name": "busybox.15d47daa8345b313",
        "namespace": "default",
        "resourceVersion": "9548561",
        "selfLink": "/api/v1/namespaces/default/events/busybox.15d47daa8345b313",
        "uid": "7d5a4c31-005b-11ea-a69a-f0f279c36caf"
    },
    "reason": "Pulled",
    "reportingComponent": "",
    "reportingInstance": "",
    "source": {
        "component": "kubelet",
        "host": "minikube"
    },
    "type": "Normal"
}
```

## 事件的广播

### Broadcaster

`Broadcaster` 是管理事件的接收、观察者的核心结构，主要的成员有：

- watchers，保存 broadcasterWatcher 的映射表，key 是逐渐增加的计数
- incoming，事件的 channel

```go
// Broadcaster distributes event notifications among any number of watchers. Every event
// is delivered to every watcher.
type Broadcaster struct {
	// TODO: see if this lock is needed now that new watchers go through
	// the incoming channel.
	lock sync.Mutex

	watchers     map[int64]*broadcasterWatcher
	nextWatcher  int64
	distributing sync.WaitGroup

	incoming chan Event

	// How large to make watcher's channel.
	// watcher 中消息接收 channel 的大小，分发消息时当一个 watcher channel 满了之后由下面的 FullChannelBehavior 决定是阻塞等待还是跳过这个 watcher 继续分发。
	watchQueueLength int
	// If one of the watch channels is full, don't wait for it to become empty.
	// Instead just deliver it to the watchers that do have space in their
	// channels and move on to the next event.
	// It's more fair to do this on a per-watcher basis than to do it on the
	// "incoming" channel, which would allow one slow watcher to prevent all
	// other watchers from getting new events.
	fullChannelBehavior FullChannelBehavior
}
```

### EventBroadcaster

`EventBroadcaster` 是用户操作广播事件的接口，控制将事件向谁广播出去，比如向 EventSink, watcher, log 等广播。

```go
type EventBroadcaster interface {
	// StartEventWatcher starts sending events received from this EventBroadcaster to the given
	// event handler function. The return value can be ignored or used to stop recording, if
	// desired.
	StartEventWatcher(eventHandler func(*v1.Event)) watch.Interface

	// StartRecordingToSink starts sending events received from this EventBroadcaster to the given
	// sink. The return value can be ignored or used to stop recording, if desired.
	StartRecordingToSink(sink EventSink) watch.Interface

	// StartLogging starts sending events received from this EventBroadcaster to the given logging
	// function. The return value can be ignored or used to stop recording, if desired.
	StartLogging(logf func(format string, args ...interface{})) watch.Interface

	// NewRecorder returns an EventRecorder that can be used to send events to this EventBroadcaster
	// with the event source set to the given event source.
	NewRecorder(scheme *runtime.Scheme, source v1.EventSource) EventRecorder
}
```

eventBroadcasterImpl 是上面的接口的一个实现，其匿名组合了 Broadcaster。

```go
type eventBroadcasterImpl struct {
	*watch.Broadcaster
	sleepDuration time.Duration
}
```

### watcher

`Interface` 是描述事件观察者 watcher 的接口，主要有两个方法，重要的是从 `ResultChan()` 中读取事件消息。

```go
type Interface interface {
	// Stops watching. Will close the channel returned by ResultChan(). Releases
	// any resources used by the watch.
	Stop()

	// Returns a chan which will receive all the events. If an error occurs
	// or Stop() is called, this channel will be closed, in which case the
	// watch should be completely cleaned up.
	ResultChan() <-chan Event
}
```

broadcasterWatcher 是上述接口的一个实现，通过 `result` 接收事件。

```go
// broadcasterWatcher handles a single watcher of a broadcaster
type broadcasterWatcher struct {
	result  chan Event
	stopped chan struct{}
	stop    sync.Once
	id      int64
	m       *Broadcaster
}
```

## 事件触发

### EventRecorder

EventRecorder 是用户触发事件的接口。用于组装一个事件并向内部的广播器广播。

```go
type EventRecorder interface {
	// 组装一个事件
	Event(object runtime.Object, eventtype, reason, message string)

	// Eventf is just like Event, but with Sprintf for the message field.
	Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{})

	// PastEventf is just like Eventf, but with an option to specify the event's 'timestamp' field.
	PastEventf(object runtime.Object, timestamp metav1.Time, eventtype, reason, messageFmt string, args ...interface{})

	// AnnotatedEventf is just like eventf, but with annotations attached
	AnnotatedEventf(object runtime.Object, annotations map[string]string, eventtype, reason, messageFmt string, args ...interface{})
}
```
Eventf 就是封装了类似 Printf 的信息打印机制，内部也会调用 Event，而 PastEventf 允许用户传进来自定义的时间戳，因此可以设置事件产生的时间。

EventRecorder 是由 recorderImpl 实现的。

```go
type recorderImpl struct {
	scheme *runtime.Scheme
	source v1.EventSource
	*watch.Broadcaster
	clock clock.Clock
}
```

## 事件处理

事件机制的目的是在事件发生时有对应的事件处理函数来处理事件。主要的事件处理方式有：

- Logging，记录 log。
- RecordingToSink，将事件记录在缓存中，将事件发给 apiserver。

### EventCorrelator

`EventCorrelator` 做事件的预处理，收集事件，可以过滤、聚合处理事件，防止发生消息泛洪，在收到多个相同的事件时，只需要处理一次，并且可以计算与 apiserver 交互的事件的优先级。具体的规则如下：

- 在 10 分钟之内发生了 10 次相似的事件则可以聚合。只有 `Event.Message` 字段变化的 Event 可以认为是相似的。会新创建一个事件并用 message 来描述这些因为相同原因聚合的事件。
- 相同事件发生后，事件的计数器会增加。
- 同一个源同一个对象资源可以突发 25 次事件。同对象的一个事件在 5 分钟之内会被重新注满。

`EventCorrelateResult` 保存着事件收集的结果。

```go
type EventCorrelator struct {
	// the function to filter the event
	filterFunc EventFilterFunc
	// the object that performs event aggregation
	aggregator *EventAggregator
	// the object that observes events as they come through
	logger *eventLogger
}

// EventCorrelateResult is the result of a Correlate
type EventCorrelateResult struct {
	// the event after correlation
	Event *v1.Event
	// if provided, perform a strategic patch when updating the record on the server
	Patch []byte
	// if true, do no further processing of the event
	Skip bool
}
```

### EventAggregator

通过 EventAggregator 来做事件的聚合

aggregateRecord 记录聚合事件，存储在 cache 中。

```go
// EventAggregator identifies similar events and aggregates them into a single event
type EventAggregator struct {
	sync.RWMutex

	// The cache that manages aggregation state
	cache *lru.Cache

	// The function that groups events for aggregation
	keyFunc EventAggregatorKeyFunc

	// The function that generates a message for an aggregate event
	messageFunc EventAggregatorMessageFunc

	// The maximum number of events in the specified interval before aggregation occurs
	maxEvents uint

	// The amount of time in seconds that must transpire since the last occurrence of a similar event before it's considered new
	maxIntervalInSeconds uint

	// clock is used to allow for testing over a time interval
	clock clock.Clock
}

// aggregateRecord holds data used to perform aggregation decisions
type aggregateRecord struct {
	// we track the number of unique local keys we have seen in the aggregate set to know when to actually aggregate
	// if the size of this set exceeds the max, we know we need to aggregate
	localKeys sets.String
	// The last time at which the aggregate was recorded
	lastTimestamp metav1.Time
}
```  

### eventLogger

通过 eventLogger 来打印发生的事件，并且统计事件发生的频率。EventLogger将相同的Event去重为1个，并通过计数表示它出现的次数。 eventLog 记录观察到的事件的数据，写入到 cache 中的。

```go
// eventLogger logs occurrences of an event
type eventLogger struct {
	sync.RWMutex
	cache *lru.Cache
	clock clock.Clock
}

// eventLog records data about when an event was observed
type eventLog struct {
	// The number of times the event has occurred since first occurrence.
	count uint

	// The time at which the event was first recorded.
	firstTimestamp metav1.Time

	// The unique name of the first occurrence of this event
	name string

	// Resource version returned from previous interaction with server
	resourceVersion string
}
```

### EventSourceObjectSpamFilter

EventSourceObjectSpamFilter 用于事件的限速，主要的成员是 burst 和 qps。

spamRecord 包含了一个 flowcontrol.RateLimiter 令牌桶限速算法。

```go
type EventSourceObjectSpamFilter struct {
	sync.RWMutex

	// the cache that manages last synced state
	cache *lru.Cache

	// burst is the amount of events we allow per source + object
	burst int

	// qps is the refill rate of the token bucket in queries per second
	qps float32

	// clock is used to allow for testing over a time interval
	clock clock.Clock
}

// spamRecord holds data used to perform spam filtering decisions.
type spamRecord struct {
	// rateLimiter controls the rate of events about this object
	rateLimiter flowcontrol.RateLimiter
}
```

### EventSink

EventSink 将事件保存到 apiserver 中。

```go
type EventSink interface {
	Create(event *v1.Event) (*v1.Event, error)
	Update(event *v1.Event) (*v1.Event, error)
	Patch(oldEvent *v1.Event, data []byte) (*v1.Event, error)
}
```


# 流程

以 ControllerManager 为例说明 record 的工作流程。

在 ControllerManager 初始化过程中会调用 KubeControllerManagerOptions.Config() 来初始化 ControllerManager 的主要配置结构 `kubecontrollerconfig.Config`。然后调用 `createRecorder()` 来执行 record 的流程：

- 创建 Broadcaster，这是事件分发的核心
- 添加 logging watcher，记录 log
- 添加 recordingToSink watcher，将事件记录在缓存中。

```go
func (s KubeControllerManagerOptions) Config(allControllers []string, disabledByDefaultControllers []string) (*kubecontrollerconfig.Config, error) {
	....
	eventRecorder := createRecorder(client, KubeControllerManagerUserAgent)
}

func createRecorder(kubeClient clientset.Interface, userAgent string) record.EventRecorder {
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
	// TODO: remove dependency on the legacyscheme
	return eventBroadcaster.NewRecorder(legacyscheme.Scheme, v1.EventSource{Component: userAgent})
}
```

## NewBroadcaster

NewBroadcaster() 会调用 NewBroadcaster() 创建一个 Broadcaster{} 管理事件、watcher、以及事件与 watcher 的分发。主要的流程：

- 创建 Broadcaster，然后进入 Broadcaster.loop() 的 goroutine。
- loop() 会一直监听事件 channel `incoming`，然后调用 distribute() 将事件分发给所有的 watcher。可以保证发送事件的顺序按照事件发生的顺序广播出去。
- 如果是添加 watcher 类型（internalRunFunctionMarker）的事件则调用添加函数。
- distribute() 会将事件通过 channel 发给 watcher。通知到 watcher 的顺序是乱序的。

```go
func NewBroadcaster(queueLength int, fullChannelBehavior FullChannelBehavior) *Broadcaster {
	m := &Broadcaster{
		watchers:            map[int64]*broadcasterWatcher{},
		incoming:            make(chan Event, incomingQueueLength),
		watchQueueLength:    queueLength,
		fullChannelBehavior: fullChannelBehavior,
	}
	m.distributing.Add(1)
	go m.loop()
	return m

}

// loop receives from m.incoming and distributes to all watchers.
func (m *Broadcaster) loop() {
	// Deliberately not catching crashes here. Yes, bring down the process if there's a
	// bug in watch.Broadcaster.
	for event := range m.incoming {
		if event.Type == internalRunFunctionMarker {
			event.Object.(functionFakeRuntimeObject)()
			continue
		}
		m.distribute(event)
	}
	m.closeAll()
	m.distributing.Done()
}

// distribute sends event to all watchers. Blocking.
func (m *Broadcaster) distribute(event Event) {
	m.lock.Lock()
	defer m.lock.Unlock()
	if m.fullChannelBehavior == DropIfChannelFull {
		for _, w := range m.watchers {
			select {
			case w.result <- event:
			case <-w.stopped:
			default: // Don't block if the event can't be queued.
			}
		}
	} else {
		for _, w := range m.watchers {
			select {
			case w.result <- event:
			case <-w.stopped:
			}
		}
	}
}
```

事件的触发是通过 Aciton 方法写入 incoming channel 来实现的，下面会讲到。

## StartLogging

StartLogging() 主要是添加一个事件处理函数记录 log。主要的流程：

- eventBroadcasterImpl.Watch() 创建 watcher
- 然后启动一个 goroutine 来监听事件并执行事件处理函数即记录 log。


```go
// StartLogging starts sending events received from this EventBroadcaster to the given logging function.
// The return value can be ignored or used to stop recording, if desired.
func (eventBroadcaster *eventBroadcasterImpl) StartLogging(logf func(format string, args ...interface{})) watch.Interface {
	return eventBroadcaster.StartEventWatcher(
		func(e *v1.Event) {
			logf("Event(%#v): type: '%v' reason: '%v' %v", e.InvolvedObject, e.Type, e.Reason, e.Message)
		})
}

// StartEventWatcher starts sending events received from this EventBroadcaster to the given event handler function.
// The return value can be ignored or used to stop recording, if desired.
func (eventBroadcaster *eventBroadcasterImpl) StartEventWatcher(eventHandler func(*v1.Event)) watch.Interface {
	watcher := eventBroadcaster.Watch()
	go func() {
		defer utilruntime.HandleCrash()
		for watchEvent := range watcher.ResultChan() {
			event, ok := watchEvent.Object.(*v1.Event)
			if !ok {
				// This is all local, so there's no reason this should
				// ever happen.
				continue
			}
			eventHandler(event)
		}
	}()
	return watcher
}
```


通过 `Watch()` 添加一个 `broadcasterWatcher` 到 Broadcaster 的 watch map 中，这样后续所有的事件都向该 watcher 通告。

```go
func (m *Broadcaster) Watch() Interface {
	var w *broadcasterWatcher
	m.blockQueue(func() {
		m.lock.Lock()
		defer m.lock.Unlock()
		id := m.nextWatcher
		m.nextWatcher++
		w = &broadcasterWatcher{
			result:  make(chan Event, m.watchQueueLength),
			stopped: make(chan struct{}),
			id:      id,
			m:       m,
		}
		m.watchers[id] = w
	})
	return w
}
```

新添加的 watcher 只能观测到添加时间节点之后的事件，这个是通过 blockQueue 来保证的，其原理是：创建一个类型为 `internalRunFunctionMarker` 的 Event，加入到 incoming 中，阻塞这个 channel，直到入参函数也就是创建和添加 broadcasterWatcher 之后才返回。然后这个入参函数会在 Broadcaster.loop() 函数中被调用。


```go
func (b *Broadcaster) blockQueue(f func()) {
	var wg sync.WaitGroup
	wg.Add(1)
	b.incoming <- Event{
		Type: internalRunFunctionMarker,
		Object: functionFakeRuntimeObject(func() {
			defer wg.Done()
			f()
		}),
	}
	wg.Wait()
}
```


## StartRecordingToSink

注册和处理事件的处理函数，主要的处理是将事件进行一系列的聚合、观测和限速过滤后保存到缓存中并且写入到 apiserver 中。

```go
// StartRecordingToSink starts sending events received from the specified eventBroadcaster to the given sink.
// The return value can be ignored or used to stop recording, if desired.
// TODO: make me an object with parameterizable queue length and retry interval
func (eventBroadcaster *eventBroadcasterImpl) StartRecordingToSink(sink EventSink) watch.Interface {
	// The default math/rand package functions aren't thread safe, so create a
	// new Rand object for each StartRecording call.
	randGen := rand.New(rand.NewSource(time.Now().UnixNano()))
	eventCorrelator := NewEventCorrelator(clock.RealClock{})
	return eventBroadcaster.StartEventWatcher(
		func(event *v1.Event) {
			recordToSink(sink, event, eventCorrelator, randGen, eventBroadcaster.sleepDuration)
		})
}
```

### NewEventCorrelator

创建事件聚合器，根据一定的策略来聚合和过滤事件。EventCorrelator 主要的成员：

- filterFunc，根据事件源限速
- aggregator，根据一定周期内发生的相似的事件做聚合
- logger：观测的作用

```go
func NewEventCorrelator(clock clock.Clock) *EventCorrelator {
	cacheSize := maxLruCacheEntries
	spamFilter := NewEventSourceObjectSpamFilter(cacheSize, defaultSpamBurst, defaultSpamQPS, clock)
	return &EventCorrelator{
		filterFunc: spamFilter.Filter,
		aggregator: NewEventAggregator(
			cacheSize,
			EventAggregatorByReasonFunc,
			EventAggregatorByReasonMessageFunc,
			defaultAggregateMaxEvents,
			defaultAggregateIntervalInSeconds,
			clock),

		logger: newEventLogger(cacheSize, clock),
	}
}
```


### recordToSink

recordToSink() 是事件处理函数，每次发生一个事件时都会调用该函数。主要的流程：

- EventCorrelate()，事件的收集，与当前一段时间内的缓存做比较，是否需要过滤、聚合和统计。
- recordEvent()，将事件记录到 apiserver 中。apiserver 收到事件后把事件更新到 ETCD。
- 如果记录失败，比如是与 apiserver 通信失败引起的，则发起重试。

```go
func recordToSink(sink EventSink, event *v1.Event, eventCorrelator *EventCorrelator, randGen *rand.Rand, sleepDuration time.Duration) {
	// Make a copy before modification, because there could be multiple listeners.
	// Events are safe to copy like this.
	eventCopy := *event
	event = &eventCopy
	result, err := eventCorrelator.EventCorrelate(event)
	if err != nil {
		utilruntime.HandleError(err)
	}
	if result.Skip {
		return
	}
	tries := 0
	for {
		if recordEvent(sink, result.Event, result.Patch, result.Event.Count > 1, eventCorrelator) {
			break
		}
		tries++
		if tries >= maxTriesPerEvent {
			klog.Errorf("Unable to write event '%#v' (retry limit exceeded!)", event)
			break
		}
		// Randomize the first sleep so that various clients won't all be
		// synced up if the master goes down.
		if tries == 1 {
			time.Sleep(time.Duration(float64(sleepDuration) * randGen.Float64()))
		} else {
			time.Sleep(sleepDuration)
		}
	}
}
```

#### EventCorrelate

EventCorrelate() 事件的收集，主要的流程：

- 事件聚合，在一段时间内的相似事件做聚合，统一处理，减少处理的资源使用率
- 事件观测，主要是查看发生频率，跟之间保存在 cache 的事件生成一个 patch
- 事件过滤，主要是是否达到限速阈值，达到了阈值则不处理。
- 根据前面的处理生成一个 EventCorrelateResult{}。


本地缓存用的是 KV 型存储，有如下四种key：

- eventKey 是源、资源对象和事件的属性值的组合，用于 cache 的索引
- aggregateKey 与 eventKey 一样
- localKey 是 Event.Message，标识具体的事件，插入到存储在 cache 中 sets.String 类型的 aggregateRecord.localKeys 中。
- spamKey 是源和资源的属性值的组合，没有事件的信息，主要用于过滤。

```go

// EventCorrelate filters, aggregates, counts, and de-duplicates all incoming events
func (c *EventCorrelator) EventCorrelate(newEvent *v1.Event) (*EventCorrelateResult, error) {
	if newEvent == nil {
		return nil, fmt.Errorf("event is nil")
	}
	aggregateEvent, ckey := c.aggregator.EventAggregate(newEvent)
	observedEvent, patch, err := c.logger.eventObserve(aggregateEvent, ckey)
	if c.filterFunc(observedEvent) {
		return &EventCorrelateResult{Skip: true}, nil
	}
	return &EventCorrelateResult{Event: observedEvent, Patch: patch}, err
}
```

##### 事件聚合

EventAggregator.EventAggregate() 实现事件的聚合，具体的流程如下：

- 获取当前时间、事件key，aggregateKey, localKey
- 根据 aggregateKey 在 cache 中查找 aggregateRecord
- 如果查找到的 aggregateRecord 最新更新的时间超过于间隔，则创建一个新的 aggregateRecord
- 如果没有在 cache 中查找到，也创建一个新的 aggregateRecord
- 向上面查找到或者新建的 aggregateRecord 中插入当前的事件 localKey，更新时间戳之后将 aggregateRecord 加入到 cache 中。
- 如果该事件该时间段内发生的次数没有超过阈值（默认 10），则不用聚合，函数处理结束返回。
- 超过阈值后将新创建一个 Event，其中 message 字段会是形如 "combined from similar events): xxxx" 的样式。

```go
func (e *EventAggregator) EventAggregate(newEvent *v1.Event) (*v1.Event, string) {
	now := metav1.NewTime(e.clock.Now())
	var record aggregateRecord
	// eventKey is the full cache key for this event
	eventKey := getEventKey(newEvent)
	// aggregateKey is for the aggregate event, if one is needed.
	aggregateKey, localKey := e.keyFunc(newEvent)

	// Do we have a record of similar events in our cache?
	e.Lock()
	defer e.Unlock()
	value, found := e.cache.Get(aggregateKey)
	if found {
		record = value.(aggregateRecord)
	}

	// Is the previous record too old? If so, make a fresh one. Note: if we didn't
	// find a similar record, its lastTimestamp will be the zero value, so we
	// create a new one in that case.
	maxInterval := time.Duration(e.maxIntervalInSeconds) * time.Second
	interval := now.Time.Sub(record.lastTimestamp.Time)
	if interval > maxInterval {
		record = aggregateRecord{localKeys: sets.NewString()}
	}

	// Write the new event into the aggregation record and put it on the cache
	record.localKeys.Insert(localKey)
	record.lastTimestamp = now
	e.cache.Add(aggregateKey, record)

	// If we are not yet over the threshold for unique events, don't correlate them
	if uint(record.localKeys.Len()) < e.maxEvents {
		return newEvent, eventKey
	}

	// do not grow our local key set any larger than max
	record.localKeys.PopAny()

	// create a new aggregate event, and return the aggregateKey as the cache key
	// (so that it can be overwritten.)
	eventCopy := &v1.Event{
		ObjectMeta: metav1.ObjectMeta{
			Name:      fmt.Sprintf("%v.%x", newEvent.InvolvedObject.Name, now.UnixNano()),
			Namespace: newEvent.Namespace,
		},
		Count:          1,
		FirstTimestamp: now,
		InvolvedObject: newEvent.InvolvedObject,
		LastTimestamp:  now,
		Message:        e.messageFunc(newEvent),
		Type:           newEvent.Type,
		Reason:         newEvent.Reason,
		Source:         newEvent.Source,
	}
	return eventCopy, aggregateKey
}
```

##### 事件观测

作用是什么？主要是为了生成 patch

事件聚合之后进入到事件观测的阶段，主要是通过 eventLogger.eventObserve 来实现的。关键的流程：

- 复制一份事件，根据 eventKey 在 cache 中获取 eventLog
- 如果 cache 中找到了，将复制后的事件的几个属性（Name、ResourceVerison等）设置为观察到的最近的事件的值。对计数值加1.
- 对复制后的事件再次复制，并设置几个属性值（Count、LastTimestamp、Message）为0
- 对赋零前后的事件做 json 序列化之后生成 patch，TODO：为什么要生成 patch ？
- 向 cache 中记录新观察的事件。

具体的代码如下：

```go
// eventObserve records an event, or updates an existing one if key is a cache hit
func (e *eventLogger) eventObserve(newEvent *v1.Event, key string) (*v1.Event, []byte, error) {
	var (
		patch []byte
		err   error
	)
	eventCopy := *newEvent
	event := &eventCopy

	e.Lock()
	defer e.Unlock()

	// Check if there is an existing event we should update
	lastObservation := e.lastEventObservationFromCache(key)

	// If we found a result, prepare a patch
	if lastObservation.count > 0 {
		// update the event based on the last observation so patch will work as desired
		event.Name = lastObservation.name
		event.ResourceVersion = lastObservation.resourceVersion
		event.FirstTimestamp = lastObservation.firstTimestamp
		event.Count = int32(lastObservation.count) + 1

		eventCopy2 := *event
		eventCopy2.Count = 0
		eventCopy2.LastTimestamp = metav1.NewTime(time.Unix(0, 0))
		eventCopy2.Message = ""

		newData, _ := json.Marshal(event)
		oldData, _ := json.Marshal(eventCopy2)
		patch, err = strategicpatch.CreateTwoWayMergePatch(oldData, newData, event)
	}

	// record our new observation
	e.cache.Add(
		key,
		eventLog{
			count:           uint(event.Count),
			firstTimestamp:  event.FirstTimestamp,
			name:            event.Name,
			resourceVersion: event.ResourceVersion,
		},
	)
	return event, patch, err
}
```

##### 事件过滤

主要的目的是限速处理，主要的处理流程：

- 根据 spamKey 从 cache 中查找 spamRecord
- 如果没有找到 spamRecord，则生成一个 spamRecord 并初始化成员 rateLimiter 用于令牌桶流控
- 查看是否被流控，即单位时间内的事件发生次数是否达到了阈值
- 将 spamRecord 再记录到 cache 中。

具体代码如下：

```go
// Filter controls that a given source+object are not exceeding the allowed rate.
func (f *EventSourceObjectSpamFilter) Filter(event *v1.Event) bool {
	var record spamRecord

	// controls our cached information about this event (source+object)
	eventKey := getSpamKey(event)

	// do we have a record of similar events in our cache?
	f.Lock()
	defer f.Unlock()
	value, found := f.cache.Get(eventKey)
	if found {
		record = value.(spamRecord)
	}

	// verify we have a rate limiter for this record
	if record.rateLimiter == nil {
		record.rateLimiter = flowcontrol.NewTokenBucketRateLimiterWithClock(f.qps, f.burst, f.clock)
	}

	// ensure we have available rate
	filter := !record.rateLimiter.TryAccept()

	// update the cache
	f.cache.Add(eventKey, record)

	return filter
}
```


#### recordEvent

事件完成了聚合、观察和过滤之后，就将事件记录到 apiserver 中了，这些过程是由 recordEvent 实现的，主要的流程：

- 判断是否已经存在与 apiserver 中了，如果已经存在，则打 patch
- 如果不存在调用 EventSink.Create() 将事件写入到 apiserver 中，
- 根据 server 返回的状态去更新 EventCorrelator
- 如果 server 返回错误，如果是服务器连接不上的错误则返回 false 之后重试，其他情况则终止对该 event 的处理。


```go
func recordEvent(sink EventSink, event *v1.Event, patch []byte, updateExistingEvent bool, eventCorrelator *EventCorrelator) bool {
	var newEvent *v1.Event
	var err error
	if updateExistingEvent {
		newEvent, err = sink.Patch(event, patch)
	}
	// Update can fail because the event may have been removed and it no longer exists.
	if !updateExistingEvent || (updateExistingEvent && isKeyNotFoundError(err)) {
		// Making sure that ResourceVersion is empty on creation
		event.ResourceVersion = ""
		newEvent, err = sink.Create(event)
	}
	if err == nil {
		// we need to update our event correlator with the server returned state to handle name/resourceversion
		eventCorrelator.UpdateState(newEvent)
		return true
	}

	// If we can't contact the server, then hold everything while we keep trying.
	// Otherwise, something about the event is malformed and we should abandon it.
	switch err.(type) {
	case *restclient.RequestConstructionError:
		// We will construct the request the same next time, so don't keep trying.
		klog.Errorf("Unable to construct event '%#v': '%v' (will not retry!)", event, err)
		return true
	case *errors.StatusError:
		if errors.IsAlreadyExists(err) {
			klog.V(5).Infof("Server rejected event '%#v': '%v' (will not retry!)", event, err)
		} else {
			klog.Errorf("Server rejected event '%#v': '%v' (will not retry!)", event, err)
		}
		return true
	case *errors.UnexpectedObjectError:
		// We don't expect this; it implies the server's response didn't match a
		// known pattern. Go ahead and retry.
	default:
		// This case includes actual http transport errors. Go ahead and retry.
	}
	klog.Errorf("Unable to write event: '%v' (may retry after sleeping)", err)
	return false
}
```


##### EventSink.Create()

创建 apiserver 中的事件对象，EventSink 接口的一个实现是 EventSinkImpl，EventSinkImpl 通过 EventInterface 与 apiserver 通信。

```go
func (e *EventSinkImpl) Create(event *v1.Event) (*v1.Event, error) {
	return e.Interface.CreateWithEventNamespace(event)
}
```

events 是 EventInterface 的一个实现，调用 events.CreateWithEventNamespace() ，最后是通过 `RESTClient` 向 apiserver 发送 http post 提交一个 event，apiserver 会创建和返回一个新的 v1.Event。


```go
// CreateWithEventNamespace makes a new event. Returns the copy of the event the server returns,
// or an error. The namespace to create the event within is deduced from the
// event; it must either match this event client's namespace, or this event
// client must have been created with the "" namespace.
func (e *events) CreateWithEventNamespace(event *v1.Event) (*v1.Event, error) {
	if e.ns != "" && event.Namespace != e.ns {
		return nil, fmt.Errorf("can't create an event with namespace '%v' in namespace '%v'", event.Namespace, e.ns)
	}
	result := &v1.Event{}
	err := e.client.Post().
		NamespaceIfScoped(event.Namespace, len(event.Namespace) > 0).
		Resource("events").
		Body(event).
		Do().
		Into(result)
	return result, err
}
```

## 事件触发

事件触发是调用 EventRecorder 的 Event 来触发的。核心的函数 generateEvent 的主要流程：

- 获取资源对象（比如 job）的 v1.ObjectReference
- 调用 makeEvent 组装结构体 v1.Event
- 执行 recorder.Action(watch.Added, event) 向 Broadcaster.incoming channel 中写入事件，因为这个可能阻塞，所以需要启动一个 goroutine 来做 Action。

```go
func (jm *JobController) syncJob(key string) (bool, error) {

	jm.recorder.Event(&job, v1.EventTypeWarning, failureReason, failureMessage)
	...
}


func (recorder *recorderImpl) Event(object runtime.Object, eventtype, reason, message string) {
	recorder.generateEvent(object, nil, metav1.Now(), eventtype, reason, message)
}

func (recorder *recorderImpl) generateEvent(object runtime.Object, annotations map[string]string, timestamp metav1.Time, eventtype, reason, message string) {
	ref, err := ref.GetReference(recorder.scheme, object)
	if err != nil {
		klog.Errorf("Could not construct reference to: '%#v' due to: '%v'. Will not report event: '%v' '%v' '%v'", object, err, eventtype, reason, message)
		return
	}

	if !validateEventType(eventtype) {
		klog.Errorf("Unsupported event type: '%v'", eventtype)
		return
	}

	event := recorder.makeEvent(ref, annotations, eventtype, reason, message)
	event.Source = recorder.source

	go func() {
		// NOTE: events should be a non-blocking operation
		defer utilruntime.HandleCrash()
		recorder.Action(watch.Added, event)
	}()
}

func (recorder *recorderImpl) makeEvent(ref *v1.ObjectReference, annotations map[string]string, eventtype, reason, message string) *v1.Event {
	t := metav1.Time{Time: recorder.clock.Now()}
	namespace := ref.Namespace
	if namespace == "" {
		namespace = metav1.NamespaceDefault
	}
	return &v1.Event{
		ObjectMeta: metav1.ObjectMeta{
			Name:        fmt.Sprintf("%v.%x", ref.Name, t.UnixNano()),
			Namespace:   namespace,
			Annotations: annotations,
		},
		InvolvedObject: *ref,
		Reason:         reason,
		Message:        message,
		FirstTimestamp: t,
		LastTimestamp:  t,
		Count:          1,
		Type:           eventtype,
	}
}

// Action distributes the given event among all watchers.
func (m *Broadcaster) Action(action EventType, obj runtime.Object) {
	m.incoming <- Event{action, obj}
}
```

# 总结

client 使用 record 的一般流程如下：

- 调用 NewBroadcaster() 生成一个广播器
- 调用 StartLogging，传入 log 记录函数记录 log。
- 调用 StartRecordingToSink，传入 client 对应版本的 Kubernetes client
- 调用 NewRecorder，返回一个 EventRecorder，就可以通过这个接口发送事件了。框架会自动记录 log 和写入 apiserver。

具体代码示例如下：

```go
eventBroadcaster := record.NewBroadcaster()
eventBroadcaster.StartLogging(klog.Infof)
eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
// TODO: remove dependency on the legacyscheme
return eventBroadcaster.NewRecorder(legacyscheme.Scheme, v1.EventSource{Component: userAgent})
```

**设计思想**

- 可靠性尽力而为，首先 Event 都保存在内存中，如果进程重启则 Event 丢失；缓存到 LRU cache 中，cache 中只有 4096 个，如果没来及处理发送给 apiserver 会被后续的事件冲刷掉。
- 性能限制，因为有很多的事件，将事件写入 etcd 会对 etcd 造成很大压力，所以要进行限速，还要对事件做汇聚和去重操作，减少写压力。


# 参考资料

[kubelet 源码分析： 事件处理](https://cizixs.com/2017/06/22/kubelet-source-code-analysis-part4-event/)
