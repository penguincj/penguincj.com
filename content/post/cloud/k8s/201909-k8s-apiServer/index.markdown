



# 1. 整体结构

<img alt="index-b0df5d91.png" src="images/index-b0df5d91.png" width="" height="" >

master 节点中运行中非常重要的组件 `api server`，来代理各个组件对 ETCD 中保存资源的访问。封装了 ETCD 的接口，通过 RESTFul API 向外提供服务。除此之外还提供了身份认证鉴权等功能。


# 2. 主要的流程

## 2.1 ServerRunOptions

```go
// cmd/kube-apiserver/app/options/options.go
type ServerRunOptions struct {
	GenericServerRunOptions *genericoptions.ServerRunOptions // 服务器通用的运行参数
	Etcd                    *genericoptions.EtcdOptions
	SecureServing           *genericoptions.SecureServingOptionsWithLoopback
	InsecureServing         *genericoptions.DeprecatedInsecureServingOptionsWithLoopback
	Audit                   *genericoptions.AuditOptions
	Features                *genericoptions.FeatureOptions
	Admission               *kubeoptions.AdmissionOptions
	Authentication          *kubeoptions.BuiltInAuthenticationOptions
	Authorization           *kubeoptions.BuiltInAuthorizationOptions
	CloudProvider           *kubeoptions.CloudProviderOptions
	StorageSerialization    *kubeoptions.StorageSerializationOptions
	APIEnablement           *genericoptions.APIEnablementOptions

	AllowPrivileged           bool // 是否配置超级权限，即允许Pod中运行的容器拥有系统特权
	EnableLogsHandler         bool
	EventTTL                  time.Duration // 事件留存时间， 默认1h
	KubeletConfig             kubeletclient.KubeletClientConfig
	KubernetesServiceNodePort int
	MaxConnectionBytesPerSec  int64
	ServiceClusterIPRange     net.IPNet // TODO: make this a list
	ServiceNodePortRange      utilnet.PortRange
	// 指定的话，可以通过SSH指定的秘钥文件和用户名对Node进行访问
	SSHKeyfile                string
	SSHUser                   string

	ProxyClientCertFile string
	ProxyClientKeyFile  string

	EnableAggregatorRouting bool

	MasterCount            int
	EndpointReconcilerType string

	ServiceAccountSigningKeyFile     string
	ServiceAccountIssuer             serviceaccount.TokenGenerator
	ServiceAccountTokenMaxExpiration time.Duration
}
```

#### 2.1.1 genericoptions.ServerRunOptions

genericoptions.ServerRunOptions 包含着运行 api server 的通用选项。

```go
// ServerRunOptions contains the options while running a generic api server.
type ServerRunOptions struct {
	// 用于广播给集群的所有成员自己的IP地址，不指定的话就使用"--bind-address"的IP地址
	AdvertiseAddress net.IP

	// CORS 跨域资源共享
	CorsAllowedOriginList       []string
	// 用于生成该master对外的URL地址
	ExternalHost                string
	MaxRequestsInFlight         int
	MaxMutatingRequestsInFlight int
	RequestTimeout              time.Duration
	MinRequestTimeout           int
	JSONPatchMaxCopyBytes int64
	MaxRequestBodyBytes int64
	TargetRAMMB         int
}
```


# 问题
1. CORS 跨域资源共享 是什么？



# 涉及到的包


"k8s.io/apiserver/pkg/server"
一些默认配置

"k8s.io/apimachinery/pkg/runtime/serializer"







# 参考

https://cizixs.com/2018/06/25/kubernetes-resource-management/
