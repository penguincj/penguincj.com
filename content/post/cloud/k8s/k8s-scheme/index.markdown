


scheme 定义了序列化和反序列化 API 对象的方法，可以从`组、版本和类型` 与 go 结构体的转换，可以在不同版本之间转换 go 的数据结构，scheme 是 API 版本的基础。

# 关键数据结构

## Scheme

Scheme 来实现类型转换，主要的成员有：

- gvkToType map[schema.GroupVersionKind]reflect.Type，保存着类型和 `group version kind` 之间的对应关系
- defaulterFuncs map[reflect.Type]func(interface{})，某个类型的初始化参数配置函数数组。
- converter \*conversion.Converter，负责具体的类型转换

具体的定义如下：

```go
// Scheme defines methods for serializing and deserializing API objects, a type
// registry for converting group, version, and kind information to and from Go
// schemas, and mappings between Go schemas of different versions. A scheme is the
// foundation for a versioned API and versioned configuration over time.
//
// In a Scheme, a Type is a particular Go struct, a Version is a point-in-time
// identifier for a particular representation of that Type (typically backwards
// compatible), a Kind is the unique name for that Type within the Version, and a
// Group identifies a set of Versions, Kinds, and Types that evolve over time. An
// Unversioned Type is one that is not yet formally bound to a type and is promised
// to be backwards compatible (effectively a "v1" of a Type that does not expect
// to break in the future).
//
// Schemes are not expected to change at runtime and are only threadsafe after
// registration is complete.
type Scheme struct {
	// versionMap allows one to figure out the go type of an object with
	// the given version and name.
	gvkToType map[schema.GroupVersionKind]reflect.Type

	// typeToGroupVersion allows one to find metadata for a given go object.
	// The reflect.Type we index by should *not* be a pointer.
	typeToGVK map[reflect.Type][]schema.GroupVersionKind

	// unversionedTypes are transformed without conversion in ConvertToVersion.
	unversionedTypes map[reflect.Type]schema.GroupVersionKind

	// unversionedKinds are the names of kinds that can be created in the context of any group
	// or version
	// TODO: resolve the status of unversioned types.
	unversionedKinds map[string]reflect.Type

	// Map from version and resource to the corresponding func to convert
	// resource field labels in that version to internal version.
	fieldLabelConversionFuncs map[schema.GroupVersionKind]FieldLabelConversionFunc

	// defaulterFuncs is an array of interfaces to be called with an object to provide defaulting
	// the provided object must be a pointer.
	defaulterFuncs map[reflect.Type]func(interface{})

	// converter stores all registered conversion functions. It also has
	// default converting behavior.
	converter *conversion.Converter

	// versionPriority is a map of groups to ordered lists of versions for those groups indicating the
	// default priorities of these versions as registered in the scheme
	versionPriority map[string][]string

	// observedVersions keeps track of the order we've seen versions during type registration
	observedVersions []schema.GroupVersion

	// schemeName is the name of this scheme.  If you don't specify a name, the stack of the NewScheme caller will be used.
	// This is useful for error reporting to indicate the origin of the scheme.
	schemeName string
}
```


# Scheme 初始化

以 ControllerManager 为例来描述 Scheme 的使用。在 ControllerManager 初始化过程中会调用 NewDefaultComponentConfig 来初始化默认参数。

NewDefaultComponentConfig 的流程如下：

- 创建 Scheme，设置初始化参数函数
- 调用 AddToScheme() 将 v1alpha1.KubeControllerManagerConfiguration 类型注册到 Scheme.typeToGVK 中。
- 再次调用 AddToScheme() 将 KubeControllerManagerConfiguration 类型注册到 Scheme.typeToGVK 中。
- 调用 scheme.Default() 来设置 v1alpha1.KubeControllerManagerConfiguration 的默认参数

```go
// "cmd/kube-controller-manager/app/options/options.go"

// NewDefaultComponentConfig returns kube-controller manager configuration object.
func NewDefaultComponentConfig(insecurePort int32) (kubectrlmgrconfig.KubeControllerManagerConfiguration, error) {
	scheme := runtime.NewScheme()
	// 下面是 KubeControllerManagerConfiguration 的两个不同版本，都加入到 Scheme.typeToGVK 中
	if err := kubectrlmgrschemev1alpha1.AddToScheme(scheme); err != nil {
		return kubectrlmgrconfig.KubeControllerManagerConfiguration{}, err
	}
	if err := kubectrlmgrconfig.AddToScheme(scheme); err != nil {
		return kubectrlmgrconfig.KubeControllerManagerConfiguration{}, err
	}

	// 调用默认参数初始化配置函数
	versioned := kubectrlmgrconfigv1alpha1.KubeControllerManagerConfiguration{}
	scheme.Default(&versioned)

	// 将 v1alpha1 版本转换到内部的版本
	internal := kubectrlmgrconfig.KubeControllerManagerConfiguration{}
	if err := scheme.Convert(&versioned, &internal, nil); err != nil {
		return internal, err
	}
	internal.Generic.Port = insecurePort
	return internal, nil
}
```

## 创建 Scheme


创建 Scheme 的流程：

- 实例化 Scheme
- 创建 Converter，用于类型转换
- 初始化一些类型的转换函数

具体代码如下：

```go
// "staging/src/k8s.io/apimachinery/pkg/runtime/scheme.go"
func NewScheme() *Scheme {
	s := &Scheme{
		gvkToType:                 map[schema.GroupVersionKind]reflect.Type{},
		typeToGVK:                 map[reflect.Type][]schema.GroupVersionKind{},
		unversionedTypes:          map[reflect.Type]schema.GroupVersionKind{},
		unversionedKinds:          map[string]reflect.Type{},
		fieldLabelConversionFuncs: map[schema.GroupVersionKind]FieldLabelConversionFunc{},
		defaulterFuncs:            map[reflect.Type]func(interface{}){},
		versionPriority:           map[string][]string{},
		schemeName:                naming.GetNameFromCallsite(internalPackages...),
	}
	s.converter = conversion.NewConverter(s.nameFunc)

	utilruntime.Must(s.AddConversionFuncs(DefaultEmbeddedConversions()...))

	// Enable map[string][]string conversions by default
	utilruntime.Must(s.AddConversionFuncs(DefaultStringConversions...))
	utilruntime.Must(s.RegisterInputDefaults(&map[string][]string{}, JSONKeyMapper, conversion.AllowDifferentFieldTypeNames|conversion.IgnoreMissingFields))
	utilruntime.Must(s.RegisterInputDefaults(&url.Values{}, JSONKeyMapper, conversion.AllowDifferentFieldTypeNames|conversion.IgnoreMissingFields))
	return s
}
```


将 KubeControllerManagerConfiguration 从 v1alpha1 版本转换到内部的版本，用的是 Converter.Convert() 转换。


```go
// Default sets defaults on the provided Object.
func (s *Scheme) Default(src Object) {
	if fn, ok := s.defaulterFuncs[reflect.TypeOf(src)]; ok {
		fn(src)
	}
}
```

defaulterFuncs 是默认参数的配置函数，那这个 `defaulterFuncs` 是在哪里设置的呢？且看后面分解。


`NewScheme()` 创建 Scheme，并初始化用于做资源对象类型转换的重要成员 `Converter`。


## AddToScheme

AddToScheme() 向 Scheme 注册类型以及参数初始化函数，v1alpha1.AddToScheme 函数实际上是 localSchemeBuilder.AddToScheme 的别名，而 localSchemeBuilder 则是 runtime.SchemeBuilder 类型的，我们继续跟踪到 SchemeBuilder 的定义。

`SchemeBuilder` 保存着向 Scheme 添加操作的函数集。调用 `AddToScheme()` 时会对 Scheme 执行所有的操作函数。在创建 SchemeBuilder 时向 Scheme 的处理函数中添加了 addKnownTypes() 来注册 `KubeControllerManagerConfiguration` 类型。


```go
// vendor/k8s.io/apimachinery/pkg/runtime/scheme_builder.go
type SchemeBuilder []func(*Scheme) error

func (sb *SchemeBuilder) Register(funcs ...func(*Scheme) error) {
	for _, f := range funcs {
		*sb = append(*sb, f)
	}
}

func (sb *SchemeBuilder) AddToScheme(s *Scheme) error {
	for _, f := range *sb {
		if err := f(s); err != nil {
			return err
		}
	}
	return nil
}

// NewSchemeBuilder calls Register for you.
func NewSchemeBuilder(funcs ...func(*Scheme) error) SchemeBuilder {
	var sb SchemeBuilder
	sb.Register(funcs...)
	return sb
}
```

### Scheme 类型注册

需要将类型加入到 Scheme.gvkToType 中才能被转换，加入是在调用 NewSchemeBuilder() 创建 SchemeBuilder 的时候的函数入参，下面的 addKnownTypes() 就是一个入参，这个函数在 AddToScheme() 调用中将需要关注的类型加入到 Scheme 中。

```go
// "vendor/k8s.io/kube-controller-manager/config/v1alpha1/register.go"
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&KubeControllerManagerConfiguration{},
	)
	return nil
}
```


各个调用者会封装一个 SchemeBuilder 自己使用，比如在 vendor/k8s.io/kube-controller-manager/config/v1alpha1/register.go 中，TODO：为什么要自己在封装一下？

### Scheme 注册参数配置函数

在 pkg/controller/apis/config/v1alpha1/register.go 中的 init() 中向 Scheme.defaulterFuncs 中注册了默认参数设置接口，分别是如下：

其中 `RegisterDefaults()` 是在自动生成的文件 `zz_generated.defaults.go` 中。

TODO： 这个文件是如何生成的？

```go

type Scheme struct {
	...
	defaulterFuncs map[reflect.Type]func(interface{})
}

func RegisterDefaults(scheme *runtime.Scheme) error {
	scheme.AddTypeDefaultingFunc(&v1alpha1.KubeControllerManagerConfiguration{}, func(obj interface{}) {
		SetObjectDefaults_KubeControllerManagerConfiguration(obj.(*v1alpha1.KubeControllerManagerConfiguration))
	})
	return nil
}

func (s *Scheme) AddTypeDefaultingFunc(srcType Object, fn func(interface{})) {
	s.defaulterFuncs[reflect.TypeOf(srcType)] = fn
}
```

最终以数据类型为 key，以对该数据类型的初始化处理函数为 value，存进了 Scheme 的 defaulterFuncs Map 属性中了。


这个 defaulterFuncs 会在上面的 NewDefaultComponentConfig 中调用。

而 SetObjectDefaults_KubeControllerManagerConfiguration() 会调用各种类型的 controller 的默认配置函数。

```go
func SetObjectDefaults_KubeControllerManagerConfiguration(in *v1alpha1.KubeControllerManagerConfiguration) {
	SetDefaults_KubeControllerManagerConfiguration(in)
	...
	SetDefaults_DeploymentControllerConfiguration(&in.DeploymentController)
}
```

用其中的 DeploymentController 举例，做了一些配置。

```go
func SetDefaults_DeploymentControllerConfiguration(obj *kubectrlmgrconfigv1alpha1.DeploymentControllerConfiguration) {
	zero := metav1.Duration{}
	if obj.ConcurrentDeploymentSyncs == 0 {
		obj.ConcurrentDeploymentSyncs = 5
	}
	if obj.DeploymentControllerSyncPeriod == zero {
		obj.DeploymentControllerSyncPeriod = metav1.Duration{Duration: 30 * time.Second}
	}
}
```

controllermanager 的 groupVersion 的设置如下：

```go
const GroupName = "kubecontrollermanager.config.k8s.io"

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: "v1alpha1"}
```


# 总结

通过 SchemeBuilder 来管理 Scheme 的初始化，通过注册不同的函数来操作 Scheme。

使用 Scheme 的大致流程如下：

- Scheme = runtime.NewScheme() //实例化 Schme
- Scheme.AddTypeDefaultingFunc(<结构体引用>, <给结构体设置默认值的函数>) // 注册默认参数赋值处理函数
- Scheme.Default(<结构体引用>) // 真正设置默认参数

ControllerManager 使用 Scheme 的流程图如下：

<img alt="index-1796c768.png" src="images/index-1796c768.png" width="" height="" >
