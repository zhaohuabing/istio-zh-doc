# Mixer 适配器模型

该软件包定义被适配器代码使用的服务和类型以便服务来自Mixer的请求。该软件包也定义用于创建Mixer模板的类型。

> 原始文件地址：
>
> https://github.com/istio/api/blob/master/mixer/adapter/model/v1beta1/istio.mixer.adapter.model.v1beta1.pb.html

## 服务

### InfrastructureBackend

`InfrastructureBackend`由为Mixer提供遥测和策略功能的后端实现，作为进程外适配器。

`InfrastructureBackend`让Mixer和后端拥有基于会话的模型。在基于会话的模型中，Mixer只将相关配置状态传递给后端一次，并使用会话标识符建立会话。 未来Mixer和后端之间的所有通信都使用会话标识符，该标识符指向先前传入的配置数据。

对于给定的`InfrastructureBackend`，Mixer调用`Validate`函数，接着调用`CreateSession`。从`CreateSession`返回的`session_id`用于进行后续的请求时的处理调用和最后的`CloseSession`函数。 Mixer假定，在有`session_id`的情况下，后端可以检索必要的上下文以服务于包含请求时负载（特定模板的实例protobuf）的处理调用。

```
rpc Validate(ValidateRequest) returns (ValidateResponse)
```

验证处理程序配置以及将被路由到该处理程序的特定模板实例。只有当其关联的`Validate`调用返回成功时，才会调用特定处理程序配置的`CreateSession`。

```
rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse)
```

为给定的处理程序配置和将被路由到该处理程序的特定模板实例创建会话。对于每个处理程序配置，Mixer通过在后端调用`CreateSession`创建单独的会话。

`CreateSessionRequest`包含特定于适配器的处理程序配置以及和实例相关的推断类型信息，处理程序在请求处理期间将接收这些信息。

`CreateSession`必须返回`session_id`，在请求处理过程中，Mixer用它来调用模板特定的Handle函数。`session_id`为Handle函数提供一种方式来检索与会话相关的必要配置。当Mixer配置发生变化时，Mixer将为现有会话无效或不存在的所有处理程序配置重新调用`CreateSession`。

如果给定相同的配置块，后端可以返回相同的会话ID。当部署中的多个Mixer实例都使用相同的配置创建会话时，会发生这种情况。请注意，单个的Mixer实例可以调用`CloseSession`，后端重用`session_id`是假设后端正在做引用计数。

如果后端无法为特定的处理程序配置创建会话并返回非S_OK状态，则Mixer将不会发出与该处理程序配置关联的请求时处理调用。

```
rpc CloseSession(CloseSessionRequest) returns (CloseSessionResponse)
```

关闭与`session_id`关联的会话。当关联的处理程序配置或实例配置更改时，Mixer关闭会话。后端应该清理与由`CloseSessionRequest`引用的`session_id`相关的所有资源。

## 类型

### CheckResult

表示前置条件检查的结果。

| 字段          | 类型                     | 描述                                                         |
| :------------ | :----------------------- | :----------------------------------------------------------- |
| status        | google.rpc.Status        | OK状态码代表前置条件被满足。其他状态码代表前置条件不满足，而detail属性描述为什么 |
| validDuration | google.protobuf.Duration | 结果可以被认为有效的时间数量                                 |
| validUseCount | int32                    | 结果可以被认为有效的使用次数                                 |

### CloseSessionRequest

`CloseSession` 方法的请求消息。

| 字段      | 类型   | 描述             |
| :-------- | :----- | :--------------- |
| `sessionId` | `string` | 要关闭的会话的ID |

### CloseSessionResponse

`CloseSession` 方法的应答消息。

| 字段   | 类型                | 描述                        |
| :----- | :------------------ | :-------------------------- |
| `status` | `google.rpc.Status` | 关闭会话调用的成功/失败状态 |

### CreateSessionRequest

`CreateSession` 方法的请求消息。

| 字段          | 类型             | 描述                                 |
| :------------ | :----------------------------------------------------------- | :----------------------------------- |
| `adapterConfig` | `google.protobuf.Any` | adapter特有配置                      |
| `inferredTypes` | `map<string, google.protobuf.Any>` | 实例名字到它们模板特有推断类型的映射 |

### CreateSessionResponse

`CreateSession` 方法的应答消息。

| 字段      | 类型                | 描述                    |
| :-------- | :------------------ | :---------------------- |
| `sessionId` | `string`              | 创建的会话的ID          |
| `status`    | `google.rpc.Status` | 验证调用的成功/失败状态 |

### Info

Info描述为Mixer提供遥测和策略功能的适配器或者后端，作为进程外适配器。

| 字段         | 类型     | 描述                                                         |
| :----------- | :------- | :----------------------------------------------------------- |
| `name`         | `string`   | 适配器的名称。必须符合RFC 1035的DNS标签，匹配`^[a-z]([-a-z0-9]*[a-z0-9])?$`正则表达式。名称用于Istio配置，因此它应该是描述性的，但是很短。例如：denier供应商适配器应该使用供应商前缀。例如：mycompany-denier |
| `description`  | `string`   | 适配器的用户友好描述                                         |
| `templates`    | `string[]` | 适配器支持的模板名字                                         |
| `config`       | `string`   | 适配器配置的proto  descriptor，Base64编码                    |
| `sessionBased` | `bool`     | 如果后端实现了 `InfrastructureBackend` 服务则为true，否则为false<br> 如果为true，在配置时刻，Mixer调用  `InfrastructureBackend` 的RPC来验证和传递处理程序配置。然后在请求时刻，mixer不再传递处理程序配置，并只使用模板特有处理服务(例如 `HandlerMetricService`, `HandlerListEntryService`, `HandleQuotaService`等)传递模板特有实例负载。如果 `sessionBased` 为false，mixer不期望后端实现了 `InfrastructureBackend` 服务，并只通过模板特有处理服务在请求时刻和后端通讯。没有 `InfrastructureBackend` 服务的情况下，mixer在请求时刻的每次调用时传递处理器配置。 |

### QuotaRequest

表示配额分配请求。

| 字段   | 类型                                  | 描述             |
| :----- | :------------------------------------ | :--------------- |
| `quotas` | `map<string, QuotaRequest.QuotaParams>` | 要分配的单个配额 |

### QuotaRequest.QuotaParams

单个配额分配的参数。

| 字段       | 类型  | 描述                                     |
| :--------- | :---- | :--------------------------------------- |
| `amount`     | `int64` | 要分配的配额数量                         |
| `bestEffort` | `bool`  | 当设置为true时，支持返回比请求要少的配额 |

### QuotaResult

表示多个配额分配的结果。

| 字段   | 类型                            | 描述                               |
| :----- | :------------------------------ | :--------------------------------- |
| `quotas` | `map<string, QuotaResult.Result>` | 分配到的配额，每个请求配额一个实体 |

### QuotaResult.Result

单个配额分配的结果。

| 字段          | 类型                     | 描述                                                         |
| :------------ | :----------------------- | :----------------------------------------------------------- |
| `validDuration` | `google.protobuf.Duration` | 结果可以被认为有效的时间数量                                 |
| `grantedAmount` | `int64`                    | 授予配额的数量。当 `QuotaParams.best_effort` 为true时，将 >=0。如果 `QuotaParams.best_effort` 为false，则要么是0，要么  >= `QuotaParams.amount` |

### ReportResult

表示report调用的结果。

> 备注：这是个空消息，没有任何字段。

### Template

Template提供mixer模板的详情。

| 字段       | 类型   | 描述                                |
| :--------- | :----- | :---------------------------------- |
| descriptor | string | 模板的proto  descriptor，Base64编码 |

### TemplateVariety

模板可用的品类，控制适配器对每个实例执行的语义。

| 名称                                 | 描述                                                         |
| :----------------------------------- | :----------------------------------------------------------- |
| `TEMPLATE_VARIETY_CHECK`               | 使模板适用于Mixer的check调用。此类模板的实例在Mixer的check调用期间创建，并根据规则配置传递给处理程序。 |
| `TEMPLATE_VARIETY_REPORT`              | 使模板适用于Mixer的report调用。此类模板的实例在Mixer的report调用期间创建，并根据规则配置传递给处理程序。 |
| `TEMPLATE_VARIETY_QUOTA`               | 使模板适用于Mixer的quota调用。此类模板的实例在Mixer的quota check调用期间创建，并根据规则配置传递给处理程序。 |
| `TEMPLATE_VARIETY_ATTRIBUTE_GENERATOR` | 使模板适用于Mixer的属性生成阶段。此类模板的实例在预处理属性生成阶段期间创建，并根据规则配置传递给处理程序。 |

### ValidateRequest

`Validate` 方法的请求消息。

| 字段          | 类型           | 描述                                 |
| :------------ | :----------------------------------------------------------- | :----------------------------------- |
| `adapterConfig` | `google.protobuf.Any` | adapter特有配置                      |
| `inferredTypes` | `map<string, google.protobuf.Any>` | 实例名字到它们模板特有推断类型的映射 |

### ValidateResponse

`Validate` 方法的应答消息。

| 字段   | 类型                | 描述                    |
| :----- | :------------------ | :---------------------- |
| `status` | `google.rpc.Status` | 验证调用的成功/失败状态 |

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
| message | string                                                       | 面向开发者的错误信息，应该用英文。任何对象用户的错误信息应该本地化然后在 `google.rpc.Status.details` 字段中发送，或者由客户端进行本地化。 |
| details | `google.protobuf.Any` | 携带错误详情的消息列表。有通用的消息类型集合给API使用        |

