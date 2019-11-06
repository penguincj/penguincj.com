


K8S 里管理着各种资源，如 Pod、Service、RC、namespaces 等资源，这些资源以 GroupVersion 的方式组织在一起。每一种资源都属于一个Group，而资源还有版本之分，如v1、v1beta1等。


一个对象由 Group（API组）、Version（API版本）和Resource（API资源类型）三个部分组成的。资源的 URL 格式：/prefix/group/version/... (例如：/apis/extensions/v1beta1)

主要有如下的两种 group：

- 核心对象，如Pod、Node，不需要 group，直接在 /api/v1 下解析
- 非核心对象的前缀是 /apis ，然后是 group，比如 /apis/extensions/v1beta1


# APIGroupVersion

通过 `APIGroupVersion` 来组织 API 资源，该字段主要有如下几部分：

- Storage 是 etcd 的接口，这是一个map类型，每一种资源都会与etcd建立一个连接；
- GroupVersion 表示该 APIGroupVersion 属于哪个 Group、哪个version；
- Serializer 用于序列化，反序列化；
- Convertor提供各个不同版本进行转化的接口；
- Mapper实现了RESTMapper接口。




# 参考

https://cizixs.com/2018/06/25/kubernetes-resource-management/
