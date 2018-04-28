# 属性词汇

属性是整个Istio使用的中心概念。可以在 [这里](../../../concepts/policy-and-control/attributes.md) 找到属性是什么和用于何处的描述。

每个给定的Istio部署有固定的能够理解的属性词汇。这个特定的词汇由当前部署中正在使用的属性生产者集合决定。Istio中首要的属性生产者是Envoy，然后特定的Mixer适配器和服务也会产生属性。

下面这个表格展示一组规范属性和他们各自的类型。大多数Istio部署都有产生这些属性的代理（Envoy或Mixer适配器）。

| 名称 | 类型 | 描述 | Kubernetes 示例 |
|------|------|-------------|--------------------|
| source.ip | ip_address | 客户端IP地址 | 10.0.0.117 |
| source.service | string | 客户端所属服务的全限定名称 | redis-master.my-namespace.svc.cluster.local |
| source.name | string | 来源服务的短名称部分 | redis-master |
| source.namespace | string | 来源服务的命名空间 | my-namespace |
| source.domain | string | 来源服务的域名后缀部分，不包括名称和命名空间 | svc.cluster.local |
| source.uid | string | 平台特有的唯一标识，用于来源服务的客户端实例 | kubernetes://redis-master-2353460263-1ecey.my-namespace |
| source.labels | map[string, string] | 客户端实例附带的键值对map | version => v1 |
| source.user | string | 请求的直接发送者的身份，由mTLS验证 | service-account-foo |
| destination.ip | ip_address | 服务器IP地址 | 10.0.0.104 |
| destination.port | int64 | 服务器IP地址上的接收端口 | 8080 |
| destination.service | string | 服务器端所属服务的全限定名称 | my-svc.my-namespace.svc.cluster.local |
| destination.name | string | 目的地服务的短名称部分 | my-svc |
| destination.namespace | string | 目的地服务的命名空间部分 | my-namespace |
| destination.domain | string | 目的地服务的域名后缀部分，不包括名称和命名空间 | svc.cluster.local |
| destination.uid | string | 平台特有的唯一标识，用于目的地服务的服务器实例 | kubernetes://my-svc-234443-5sffe.my-namespace |
| destination.labels | map[string, string] | 服务器端实例附带的键值对map | version => v2 |
| destination.user | string | 运行目的地应用的用户名 | service-account |
| request.headers | map[string, string] | HTTP 请求 headers. 对于 gRPC, 则是它的 metadata. | |
| request.id | string | 请求的ID，具有统计上较低的碰撞概率。 | |
| request.path | string | HTTP URL 路径，包括 query string | |
| request.host | string | HTTP/1.x host header 或者 HTTP/2 authority header. | redis-master:3337 |
| request.method | string | HTTP method. | |
| request.reason | string | 请求理由，用于审计系统 | |
| request.referer | string | HTTP referer header. | |
| request.scheme | string | 请求的URI Scheme | |
| request.size | int64 | 以字节计算的请求大小。对于HTTP请求，等同于Content-Length header. | |
| request.time | timestamp | 目的地接收到请求的时间戳。这等同于Firebase "now". | |
| request.useragent | string | HTTP User-Agent header. | |
| response.headers | map[string, string] | HTTP 响应 headers. | |
| response.size | int64 | 以字节计算的响应大小。 | |
| response.time | timestamp | 目的地产生应答的时间戳 | |
| response.duration | duration | 生成应答花费的时间数量 | |
| response.code | int64 | 应答的 HTTP 状态码. | |
| connection.id | string | TCP 连接的ID，具有统计上较低的碰撞概率。 | |
| connection.received.bytes | int64 | 对于一条连接，在最后一次Report()之后，目的地服务在此连接上接收到的字节数量 | |
| connection.received.bytes_total | int64 | 在连接的生命周期中，目的地服务接收到的全部字节数量 | |
| connection.sent.bytes | int64 | 对于一条连接，在最后一次Report()之后，目的地服务在此连接上发送的字节数量 | |
| connection.sent.bytes_total | int64 | 在连接的生命周期中，目的地服务发送的全部字节数量 | |
| connection.duration | duration | 连接打开的时间总数量 | |
| context.protocol | string | 请求或者被代理的连接的协议 | tcp |
| context.time | timestamp | Mixer 操作的时间戳. | |
| api.service | string | 公开的服务名。和处于mesh中的服务身份不同，并反映到暴露给客户度的名称。 | my-svc.com |
| api.version | string | API 版本. | v1alpha1 |
| api.operation | string | 用于辨别操作的唯一字符串。在特定的`<service, version>` 描述的所有操作中，这个ID是唯一的。 | getPetsById |
| api.protocol | string | API调用的协议类型。主要用于监控／分析。注意这事暴露给客户端的前端协议，不是后端服务实现的协议。 | "http", “https”, 或 "grpc" |
| request.auth.principal | string | The authenticated principal of the request. This is a string of the issuer (`iss`) and subject (`sub`) claims within a JWT concatenated with “/” with a percent-encoded subject value. | accounts.my-svc.com/104958560606 |
| request.auth.audiences | string | The intended audience(s) for this authentication information. This should reflect the audience (`aud`) claim within a JWT. | ['my-svc.com', 'scopes/read'] |
| request.auth.presenter | string | The authorized presenter of the credential. This value should reflect the optional Authorized Presenter (`azp`) claim within a JWT or the OAuth2 client id. | 123456789012.my-svc.com |
| request.api_key | string | 用于请求的API key | abcde12345 |
| check.error_code | int64 | Mixer Check 调用的[错误码](https://github.com/google/protobuf/blob/master/src/google/protobuf/stubs/status.h#L44). | 5 |
| check.error_message | string | Mixer Check 调用的错误消息. | Could not find the resource |

