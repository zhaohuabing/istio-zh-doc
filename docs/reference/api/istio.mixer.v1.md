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

每个给定的Istio部署有固定的能够理解的属性词汇。这个特定的词汇由当前部署中正在使用的属性生产者集合决定。Istio中首要的属性生产者是Envoy，然后特定的Mixer适配器和服务也会产生属性。

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

定义压缩格式的属性列表，用于传输优化。在此消息中，使用整数索引将字符串引用到两个字符串字典之一。 正整数索引到全局部署级字典，而负整数索引到消息级字典。 消息级字典由此消息的`words`字段携带，部署级字典通过配置确定。

| 字段         | 类型                                                         | 描述                                               |
| ------------ | ------------------------------------------------------------ | -------------------------------------------------- |
| `words`      | `string[]`                                                   | 消息级字典。                                       |
| `strings`    | `map<int32, int32>`                                          | 持有STRING, DNS*NAME, EMAIL*ADDRESS, URI类型的属性 |
| `int64s`     | `map<int32, int64>`                                          | 持有INT64类型的属性                                |
| `doubles`    | `map<int32, double>`                                         | 持有DOUBLE类型的属性                               |
| `bools`      | `map<int32, bool>`                                           | 持有BOOL类型的属性                                 |
| `timestamps` | map< int32, [google.protobuf.Timestamp](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#timestamp) > | 持有TIMESTAMP类型的属性                            |
| `durations`  | map< int32, [google.protobuf.Duration](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#duration) > | 持有DURATION类型的属性                             |
| `bytes`      | `map<int32, bytes>`                                          | 持有BYTES类型的属性                                |
| `stringMaps` | `map<int32, StringMap>`                                      | 持有STRING_MAP类型的属性                           |

### ReferencedAttributes

描述用于决定应答的属性。这可以用来构建应答的缓存。

| 字段               | 类型                                                         | 描述                                                         |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `words`            | `string[]`                                                   | 消息级别字典。参考 [CompressedAttributes](#CompressedAttributes) 获取使用字典的信息。 |
| `attributeMatches` | [ReferencedAttributes.AttributeMatch[]](#ReferencedAttributes.AttributeMatch) | 描述属性集合。                                               |

### ReferencedAttributes.AttributeMatch

描述单个属性的匹配情况。

| 字段        | 类型                                                         | 描述                                                         |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `name`      | `int32`                                                      | 属性的名称。这是字典索引，编码方式和 [CompressedAttributes](#CompressedAttributes) 消息中所有的字符串一致。 |
| `condition` | [ReferencedAttributes.Condition](#ReferencedAttributes.Condition) | 属性值的匹配类型。                                           |
| `regex`     | `string`                                                     | 如果属性是STRING_MAP类型，condition是REGEX，则客户端应使用正则表达式值来匹配map key。 |
| `mapKey`    | `int32`                                                      | STRINGMAP中的key。当一个STRINGMAP属性的多个key被引用时，会有多个AttributeMatch消息，带有不同的mapkey值。如果属性不是STRING_MAP类型，mapKey的值应该被忽略。<br>使用key的索引（从`words`字段而来的消息字典中来，或者从全局字典中来）。<br>如果属性是STRINGMAP类型，又没有提供mapKey的值，则将使用整个STRING_MAP。 |

### ReferencedAttributes.Condition

属性的值是如何匹配的。

| 名称                    | 描述                                 |
| ----------------------- | ------------------------------------ |
| `CONDITION_UNSPECIFIED` | 不应该发生                           |
| `ABSENCE`               | 当属性不存在时匹配                   |
| `EXACT`                 | 当属性的值是精确的每个字节匹配时匹配 |
| REGEX                   | 当属性的值匹配包含的正则表达式时匹配 |

### ReportRequest

用于在执行一个或者多个操作之后报告遥测。

| 字段            | 类型                                             | 描述                                                         |
| --------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| `attributes`    | [CompressedAttributes[\]](#CompressedAttributes) | 用于这次报告的属性。<br>每个 `Attributes` 元素代表单个操作的状态。多个操作可以在单条消息中提供来提供通讯效率。客户端可以积累多个操作，然后在单条消息中将他们都发送出去。 |
| `defaultWords`  | `string[]`                                       | 用于所有属性的默认消息级别字典。单个属性消息可以有他们自己的字典，但是如果他们没有，则如果这个词语集合被提供，就可以替代使用。 |
| globalWordCount | uint32                                           | 全局字典中词语的数量。用来检测客户端和服务器端之间的全局字典是否不同步。 |

### ReportResponse

用来携带对遥测报告的应答。

> 注：这个消息是空消息，没有字段。

### StringMap

string到string的map。在这个map中，key和value都是字典索引（查看 [Attributes](#CompressedAttributes) 消息来获取解释）。

| 字段      | 类型                | 描述                  |
| --------- | ------------------- | --------------------- |
| `entries` | `map<int32, int32>` | 持有名称/值对的集合。 |

### google.rpc.Status

`Status` 类型定义逻辑错误模型，适用于不同的编程环境，包括REST API和RPC API。用于[gRPC](https://github.com/grpc)。错误模型设计为：

- 对于大多数用户，可以简单的使用和理解
- 足够弹性来应对意外需求

#### 概述

`Status` 消息包含三块数据：错误码，错误消息和错误详情。错误码是*google.rpc.Code*的枚举值，但是在需要是可以接受额外的错误码。错误消息应该是面对开发人员的英语消息，帮助开发人员理解和解决错误。如果需要本地化的面对用户的错误消息，在错误详情中放置本地化的消息，或者在客户端做本地化。可选的错误详情可以包含有关错误的任意信息。在包 `google.rpc` 中有预定义定义的错误详情类型，可以用于常见错误条件。

### 语言映射

`Status` 消息是错误模型的逻辑表示，但是它不一定是实际的格式。当 `Status` 消息在不同的客户端类库和不同的协议中暴露时，它可以有不同的映射。例如，在java中它可能映射到某些异常，但是在C中更多的映射到某些错误代码。

### 其他使用

错误模型和`Status` 消息可以在不同的环境中使用，无论带或者不带API，以便在不同环境下提供一致的开发体验。

这个错误模型的使用例子包括：

- 部分错误。如果服务需要返回部分错误给客户端，它可以在正常的应答中嵌入 Status 。
- 工作流错误。典型的工作量有多个步骤。每个步骤需要有一个`Status` 消息用来错误报告。
- 批量操作。如果客户端采用批量请求和批量应答，`Status` 消息可以直接在批量应答直接使用。
- 异步操作。如果一个API调用将异步操作结果内嵌到它的响应中，则这些操作的状态应该使用`Status` 消息直接表述。
- 日志。如果某些API错误被存储在日志中，`Status` 消息可以在脱敏之后直接使用，脱敏是出于必要的安全和隐私原因。

| 字段    | 类型                                                         | 描述                                                         |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| code    | int32                                                        | 状态码，必须是` google.rpc.Code`的枚举值                     |
| message | string                                                       | 面向开发者的错误信息，应该用英文。任何对象用户的错误信息应该本地化然后在 [google.rpc.Status.details](#google.rpc.Status.details) 字段中发送，或者由客户端进行本地化。 |
| details | [google.protobuf.Any](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#any) | 携带错误详情的消息列表。有通用的消息类型集合给API使用        |