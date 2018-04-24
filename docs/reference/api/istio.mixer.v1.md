# Mixer

定义Mixer API，sidecar代理将使用这个API来执行前置条件检查，管理配额和提交遥测报告。

## Services

### Mixer

Mixer提供三个核心功能：

- 前提条件检查。在响应来自服务调用者的传入请求之前，使得调用者能够验证许多前提条件。前提条件可以包括服务使用者是否被正确认证，是否在服务的白名单上，是否通过ACL检查等等。
- 配额管理。 使服务能够在多个维度上分配和释放配额，当服务消费者对有限的资源发生抢占时，配额就成了一种有效的资源管理工具，它为服务之间的资源抢占提供相对的公平。频率控制就是配额的一个实例。
- 遥测报告。使服务能够上报日志和监控。在未来，它还将启用针对服务运营商以及服务消费者的跟踪和计费流。

```go
rpc Check(CheckRequest) returns (CheckResponse)
```

在执行操作前，检查前置条件和分配配额。实施的前置条件取决于提供的属性集和当前生效的配置。

```go
rpc Report(ReportRequest) returns (ReportResponse)
```

报告遥测数据，例如日志和度量。要上报的信息取决于提供的属性和当前生效的配置。

## Types

### Attributes

属性代表一组有类型的名称／值对。Mixer的很多API或者使用或者返回属性。

Istio使用属性来控制运行在服务网格中的服务的运行时行为。属性是有命名和有类型的元数据片段，描述ingress和egress流量，和流量发生所在的环境。Istio的属性携带信息的特别片段，例如API请求的错误码，API请求的延迟，或者TCP连接的初始IP地址。例如：

``` bash
request.path: xyz/abc
request.size: 234
request.time: 12:34:56.789 04/17/2017
source.ip: 192.168.0.1
target.service: example
```

每个给定的Istio部署有固定的能够理解的属性词汇表。这个特定的词汇表由当前部署中正在使用的属性生产者集合决定。Istio中首要的属性生产者是Envoy，然后特定的Mixer适配器和服务也会产生属性。

在大多数Istio部署中，可用的通用基准属性集在[这里](../config/mixer/attribute-vocabulary.md)定义。

属性是强类型的。支持的属性类型由 [ValueType](https://github.com/istio/api/blob/master/policy/v1beta1/value_type.proto) 定义。值的每个类型被编码为在消息中表示的所谓传输类型之一。

使用未压缩格式定义属性map。下列场合可能使用这个消息：1）使用静态的每proxy属性配置Istio/Proxy，例如source.uid。2）服务IDL定义，用来为每个请求提取API属性 3）为HTTP请求转发属性，从客户端proxy到服务端proxy

| 字段       | 类型                                                         | 描述                  |
| ---------- | ------------------------------------------------------------ | --------------------- |
| attuibutes | `map<string, [Attributes.AttributeValue](#Attributes.AttributeValue)>` | 属性的map，从名称到值 |

### Attributes.AttributeValue

指定属性的值，使用不同的类型。

| 字段           | 类型                                                         | 描述                                           |
| -------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| stringValue    | string (oneof)                                               | 用于STRING, DNSNAME, EMAILADDRESS和URI类型的值 |
| int64Value     | int64 (oneof)                                                | 用于INT64类型的值                              |
| doubleValue    | `double (oneof)`                                             | 用于DOUBLE类型的值                             |
| boolValue      | bool (oneof)                                                 | 用于BOOL类型的值                               |
| bytesValue     | bytes (oneof)                                                | 用于BYTES类型的值                              |
| timestampValue | [google.protobuf.Timestamp (oneof)](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#timestamp) | 用于 TIMESTAMP类型的值                         |
| durationValue  | [google.protobuf.Duration (oneof)](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#duration) | 用于 DURATION类型的值                          |
| stringMapValue | [Attributes.StringMap (oneof)](#Attributes.StringMap)        | 用于 STRING_MAP类型的值                        |

### Attributes.StringMap

定义string map。

| 字段    | 类型                  | 描述               |
| ------- | --------------------- | ------------------ |
| entries | `map<string, string>` | 持有名称／值对集合 |

### CheckRequest

在执行动作前，使用获取许可或者禁止。

| 字段            | 类型                                                         | 描述                                                         |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| attributes      | [CompressedAttributes](#CompressedAttributes)                | 用于此次请求的属性。<br> mixer的配置决定这些属性将被如何使用以创建在应答中返回的结果。 |
| globalWordCount | uint32                                                       | 全局字典中的词汇的数量，填充属性时使用。这个值用于快速检验客户端是否在使用服务器端理解的字典。 |
| deduplicationId | string                                                       | 用于在RPC失败并重试的情况下消除重复的 check 调用。这可是是每个请求的UUID，同一个调用的重发将使用同样的UUID。 |
| quotas          | map< string, [CheckRequest.QuotaParams](#CheckRequest.QuotaParams) > | 要分别分配的配额。（注：可以同时要求分配多个配额）           |

### CheckRequest.QuotaParams

用于配额分配的参数。

| 字段       | 类型  | 描述                                   |
| ---------- | ----- | -------------------------------------- |
| amount     | int64 | 要分配的配额数量                       |
| bestEffort | bool  | 当为true时，支持返回比请求的更少的配额 |

### CheckResponse.PreconditionResult

表示前置条件检查的结果。

| 字段                 | 类型                                                         | 描述                                                         |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| status               | [google.rpc.Status](#google.rpc.Status)                      | 状态码OK表示所有前置条件均满足。任何其它状态码表示不是所有的前置条件都满足，并且在detail中描述为什么。 |
| validDuration        | [google.protobuf.Duration](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#duration) | 时间量，在此期间这个结果可以认为是有效的                     |
| validUseCount        | int32                                                        | 可使用的次数，在此期间这个结果可以认为是有效的               |
| attributes           | [CompressedAttributes](#CompressedAttributes)                | mixer返回的属性。<br>返回的切确属性集合由mixer配置的adapter决定。这些属性用于传送新属性，这些新属性是Mixer根据输入的属性集合和它的配置派生的。 |
| referencedAttributes | [ReferencedAttributes](#ReferencedAttributes)                | 在匹配条件并生成结果的过程中使用到的全部属性集合。           |

### CheckResponse.QuotaResult

表示配额分配的结果。

| 字段                 | 类型                                                         | 描述                                                         |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| validDuration        | [google.protobuf.Duration](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#duration) | 时间量，在此期间这个结果可以认为是有效的                     |
| grantedAmount        | int64                                                        | 授予的配额数量。当` QuotaParams.best_effort`为true时，这个值将 >= 0。如果` QuotaParams.best_effort`为false，这个值会是0或者>=  `QuotaParams.amount` |
| referencedAttributes | [ReferencedAttributes](#ReferencedAttributes)                | 在匹配条件并生成结果的过程中使用到的全部属性集合。           |

### CompressedAttributes

定义压缩格式的属性列表，用于传输优化。使用这个消息，用这个消息，

| 字段         | 类型                                                         | 描述                                       |
| ------------ | ------------------------------------------------------------ | ------------------------------------------ |
| precondition | [CheckResponse.PreconditionResult](#CheckResponse.PreconditionResult) | 前置条件检查结果。                         |
| quotas       | map< string, [CheckResponse.QuotaResult](#CheckResponse.QuotaResult) > | 与此产生的配额，每个请求的配额对应一个条目 |







