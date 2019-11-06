





# 对象


每个对象都要实现 `Object` 的接口，

type Object interface {
	GetObjectKind() schema.ObjectKind
	DeepCopyObject() Object
}



自定义扩展对象

type RawExtension struct {
	// Raw is the underlying serialization of this object.
	//
	// TODO: Determine how to detect ContentType and ContentEncoding of 'Raw' data.
	Raw []byte `protobuf:"bytes,1,opt,name=raw"`
	// Object can hold a representation of this extension - useful for working with versioned
	// structs.
	Object Object `json:"-"`
}







# 重点的包











## scheme


scheme 定义了序列化和反序列化 API 对象的方法，可以从`组、版本和类型` 与 go 结构体的转换，可以在不同版本之间转换 go 的数据结构，scheme 是 API 版本的基础。

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


scheme 通过 conversion.Converter 做类型转换，注册了一些 Converter 的方法。




### Scheme 初始化


调用 `runtime.NewScheme()` 创建 Scheme，注册一系列的默认参数初始化函数。

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

**创建 Scheme**


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





### SchemeBuilder

`SchemeBuilder` 保存着向 Scheme 添加操作的函数集。调用 `AddToScheme()` 时会对 Scheme 执行所有的操作函数。在创建 SchemeBuilder 时向 Scheme 的处理函数中添加了 addKnownTypes() 来注册 `KubeControllerManagerConfiguration` 类型。


可以看到，v1alpha1.AddToScheme 函数实际上是 localSchemeBuilder.AddToScheme 的别名，而 localSchemeBuilder 则是 runtime.SchemeBuilder 类型的，我们继续跟踪到 SchemeBuilder 的定义


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

在 pkg/controller/apis/config/v1alpha1/register.go 中的 init() 中向 Scheme.defaulterFuncs 中注册了默认参数设置接口，分别是如下：

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




## serializer

可以根据具体的版本和类型提供序列化的方式

```go
// CodecFactory provides methods for retrieving codecs and serializers for specific
// versions and content types.
type CodecFactory struct {
	scheme      *runtime.Scheme
	serializers []serializerType
	universal   runtime.Decoder
	accepts     []runtime.SerializerInfo

	legacySerializer runtime.Serializer
}
```


## conversion

vendor/k8s.io/apimachinery/pkg/conversion/converter.go

Scope 被传递到转换函数中，保存正在进行转换的信息，如果系统中有多个转换器存在，就用调用者的转换函数。

```go
// Scope is passed to conversion funcs to allow them to continue an ongoing conversion.
// If multiple converters exist in the system, Scope will allow you to use the correct one
// from a conversion function--that is, the one your conversion function was called by.
type Scope interface {
	// Call Convert to convert sub-objects. Note that if you call it with your own exact
	// parameters, you'll run out of stack space before anything useful happens.
	Convert(src, dest interface{}, flags FieldMatchingFlags) error

	// DefaultConvert performs the default conversion, without calling a conversion func
	// on the current stack frame. This makes it safe to call from a conversion func.
	DefaultConvert(src, dest interface{}, flags FieldMatchingFlags) error

	// SrcTags and DestTags contain the struct tags that src and dest had, respectively.
	// If the enclosing object was not a struct, then these will contain no tags, of course.
	SrcTag() reflect.StructTag
	DestTag() reflect.StructTag

	// Flags returns the flags with which the conversion was started.
	Flags() FieldMatchingFlags

	// Meta returns any information originally passed to Convert.
	Meta() *Meta
}
```

scope 实现了 Scope 接口

```go
// scope contains information about an ongoing conversion.
type scope struct {
	converter *Converter
	meta      *Meta
	flags     FieldMatchingFlags

	// srcStack & destStack are separate because they may not have a 1:1
	// relationship.
	srcStack  scopeStack
	destStack scopeStack
}
```

ConversionFunc 是转移函数，从 a 转换到 b。如果不要 a 和 b 共享字段，需要在调用函数前复制一份 a。

	type ConversionFunc func(a, b interface{}, scope Scope) error

d

通过 ConversionFuncs 来组织转换函数。

```go
type ConversionFuncs struct {
	fns     map[typePair]reflect.Value
	untyped map[typePair]ConversionFunc
}
```

通过 `typePair` 来表示转换的源和目的。

```go
type typePair struct {
	source reflect.Type
	dest   reflect.Type
}

type typeNamePair struct {
	fieldType reflect.Type
	fieldName string
}
```


Scheme 调用 Convert 的时候会提供 Meta 信息。

```go
// Meta is supplied by Scheme, when it calls Convert.
type Meta struct {
	// KeyNameMapping is an optional function which may map the listed key (field name)
	// into a source and destination value.
	KeyNameMapping FieldMappingFunc
	// Context is an optional field that callers may use to pass info to conversion functions.
	Context interface{}
}
```


### 转换


转换的主要流程是 Converter.Convert() 开始的

```go
func (c *Converter) Convert(src, dest interface{}, flags FieldMatchingFlags, meta *Meta) error {
	return c.doConversion(src, dest, flags, meta, c.convert)
}
```



doConversion 的主要流程如下：
- 首先实例化一个 scope
- 看下源目的对在不在 `conversionFuncs` 和 `generatedConversionFuncs` 中，在其中则调用转换函数直接转换。
- 获取源和目的的地址，目的需要 addressAble 和 setAble。
- 调用实际的转换函数 `convert()` 开始转换。

```go
func (c *Converter) doConversion(src, dest interface{}, flags FieldMatchingFlags, meta *Meta, f conversionFunc) error {
	pair := typePair{reflect.TypeOf(src), reflect.TypeOf(dest)}
	scope := &scope{
		converter: c,
		flags:     flags,
		meta:      meta,
	}
	if fn, ok := c.conversionFuncs.untyped[pair]; ok {
		return fn(src, dest, scope)
	}
	if fn, ok := c.generatedConversionFuncs.untyped[pair]; ok {
		return fn(src, dest, scope)
	}
	// TODO: consider everything past this point deprecated - we want to support only point to point top level
	// conversions

	dv, err := EnforcePtr(dest)
	if err != nil {
		return err
	}
	if !dv.CanAddr() && !dv.CanSet() {
		return fmt.Errorf("can't write to dest")
	}
	sv, err := EnforcePtr(src)
	if err != nil {
		return err
	}
	// Leave something on the stack, so that calls to struct tag getters never fail.
	scope.srcStack.push(scopeStackElem{})
	scope.destStack.push(scopeStackElem{})
	return f(sv, dv, scope)
}
```

`convert` 的流程：

- 检查源目的对是否在忽略列表里，在的话直接返回
- 是否在 `conversionFuncs` 或者 `generatedConversionFuncs` 中，在的话调用用户自定义的转换接口
- 否则调用默认转换接口 `defaultConvert`

具体代码如下：

```go
func (c *Converter) convert(sv, dv reflect.Value, scope *scope) error {
	dt, st := dv.Type(), sv.Type()
	pair := typePair{st, dt}

	// ignore conversions of this type
	if _, ok := c.ignoredConversions[pair]; ok {
		if c.Debug != nil {
			c.Debug.Logf("Ignoring conversion of '%v' to '%v'", st, dt)
		}
		return nil
	}

	// Convert sv to dv.
	if fv, ok := c.conversionFuncs.fns[pair]; ok {
		if c.Debug != nil {
			c.Debug.Logf("Calling custom conversion of '%v' to '%v'", st, dt)
		}
		return c.callCustom(sv, dv, fv, scope)
	}
	if fv, ok := c.generatedConversionFuncs.fns[pair]; ok {
		if c.Debug != nil {
			c.Debug.Logf("Calling generated conversion of '%v' to '%v'", st, dt)
		}
		return c.callCustom(sv, dv, fv, scope)
	}

	return c.defaultConvert(sv, dv, scope)
}
```



其中 `nameFunc` 是在 NewScheme() 创建 Converter 传递进去的。

nameFunc 的处理逻辑： 如果这个对象注册了，则返回注册的类型，否则返回 go 结构体的名字。

```go
// nameFunc returns the name of the type that we wish to use to determine when two types attempt
// a conversion. Defaults to the go name of the type if the type is not registered.
func (s *Scheme) nameFunc(t reflect.Type) string {
	// find the preferred names for this type
	gvks, ok := s.typeToGVK[t]
	if !ok {
		return t.Name()
	}

	for _, gvk := range gvks {
		internalGV := gvk.GroupVersion()
		internalGV.Version = APIVersionInternal // this is hacky and maybe should be passed in
		internalGVK := internalGV.WithKind(gvk.Kind)

		if internalType, exists := s.gvkToType[internalGVK]; exists {
			return s.typeToGVK[internalType][0].Kind
		}
	}

	return gvks[0].Kind
}
```



#### defaultConvert








调用 checkField() 检查并做类型转换，如果有一个类型做了转换就返回 true。

如果是 struct 或者 map 可以用 kvValue 接口。

// kvValue lets us write the same conversion logic to work with both maps
// and structs. Only maps with string keys make sense for this.
type kvValue interface {
	// returns all keys, as a []string.
	keys() []string
	// Will just return "" for maps.
	tagOf(key string) reflect.StructTag
	// Will return the zero Value if the key doesn't exist.
	value(key string) reflect.Value
	// Maps require explicit setting-- will do nothing for structs.
	// Returns false on failure.
	confirmSet(key string, v reflect.Value) bool
}


**Converter.convertKV()**

主要做 struct 和一些 map 的转换，主要的流程如下：

- 根据 FieldMatchingFlags 的设置是从 src 还是 dst 开始遍历
- 从选中的源中遍历每一个字段，检查这个字段在另一个结构中是否存在，存在则继续递归调用 Converter.convert() 转换
- 如果不存在但是这个字段在另一个结构体中也是合法的，设置 scope  stack 栈顶的 key 和 tag，继续调用 Converter.convert() 看有没有自定义的转换接口，因为默认的转换已经转换不了了。







### scopeStack

在 scope 中有个 srcStack 和 dstStack 字段，主要用于打印调试的目的，具体的字段如下：

```go
type scope struct {
	// srcStack & destStack are separate because they may not have a 1:1
	// relationship.
	srcStack  scopeStack
	destStack scopeStack
}


type scopeStackElem struct {
	tag   reflect.StructTag
	value reflect.Value
	key   string
}

type scopeStack []scopeStackElem

```


scopeStack 维持一个堆栈，在 defaultConvert 中压栈，在 defaultConvert 处理完之后出栈。

scopeStackElem 的 key 在不同的结构中表示不同的意思：

- slice：slice 的下标
- map：map 的 key






### 流程


```go

(c *Converter) convert(sv, dv reflect.Value, scope *scope) 从 sv 到 dv 的转换
  Converter.defaultConvert() 根据各种类型做转换
    Converter.convertKV() 转换 struct 和一些 map
      Converter.checkField() 根据 flag 检查某个字段，并递归调用 Converter.convert() 做转换
        Converter.convert() 继续转换


```


### 总结

可以递归调用某个函数来做一些需要嵌套的工作，处理好返回即可

可以维护一个 scopeStack 来传递在递归调用中的信息。


### 问题


1. conversion 中有个 DebugLogger 来实现调试打印，如何做的？

	type DebugLogger interface {
		Logf(format string, args ...interface{})
	}


2. scopeStack 作用是什么？ scopeStack 的第 0、1 个元素的特殊性？

	在 doConversion 中向 scope.scopeStack[0] 中 push 了空的结构

3. Convertor 中的 conversionFuncs 用途是什么？性能？`conversionFuncs` 和 `generatedConversionFuncs` 区别是什么？

	看起来是用户添加进去的自定义转换接口。

4. 	Map, Ptr, Slice, Interface, Struct 这些类型有什么特殊性？为什么在转换的时候要特殊对待？
