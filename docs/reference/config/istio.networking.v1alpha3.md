Route Rules Alpha 3

关系到流量路由的配置。这里有几个很有用的术语，它们在流量路由的上下文中定义。

`服务/Service` - 应用行为单元，绑定到服务注册表中的唯一名称。服务由多个网络端点组成，这些端点由运行在 pod，容器，虚拟机等上的工作负载实例实现。

`服务版本(子集)/Service versions (subsets)` - 在持续部署场景中，对于给定的服务，可以有不同的实例子集，分别运行应用程序二进制的不同变体。这些变体不一定是不同的API版本。他们可以是同一服务的迭代更改，部署在不同的环境中（prod，staging，dev等）。发生这种情况的常见情景包括A / B测试，金丝雀推出等。特定版本的选择可以基于各种标准（headers，url等）和/或通过分配给每个版本的权重来决定。每个服务都有一个由其所有实例组成的默认版本。

`源/Source` - 调用服务的下游客户端。

`主机/Host` - 客户端尝试连接服务时使用的地址。

`访问模式/Access model` - 应用程序只寻址目标服务（主机）而无需了解单个服务版本（子集）。版本的实际选择取决于代理/sidecar，从而使应用程序代码能够将其本身从依赖服务的演变中解耦出来。

## ConnectionPoolSettings

上游主机的连接池设置。这些设置适用于上游服务中的每台主机。更多详细信息，请参阅 Envoy 的 [断路器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/circuit_breaking) 。连接池设置可以应用在 TCP 级别以及 HTTP 级别。

例如，以下规则设置到被称为 myredissrv 的 redis 服务的连接限制为100，连接超时为30ms

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-redis
spec:
  name: myredissrv
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
```

| 字段   | 类型                                  | 描述                          |
| ------ | ------------------------------------- | ----------------------------- |
| `tcp`  | `ConnectionPoolSettings.TCPSettings`  | HTTP和TCP上游连接的通用设置。 |
| `http` | `ConnectionPoolSettings.HTTPSettings` | HTTP 连接池设置。             |

## ConnectionPoolSettings.HTTPSettings

适用于 HTTP1.1/HTTP2/GRPC 连接的配置.

| 字段                       | 类型    | 描述                                                         |
| -------------------------- | ------- | ------------------------------------------------------------ |
| `http1MaxPendingRequests`  | `int32` | 对目标的最大挂起HTTP请求数。默认1024。                       |
| `http2MaxRequests`         | `int32` | 对后端的最大请求数。默认1024。                               |
| `maxRequestsPerConnection` | `int32` | 每个到后端的连接的最大请求数。将此参数设置为1将禁用keep alive。 |
| `maxRetries`               | `int32` | 给定时间内，集群中所有主机的最大重试次数。默认为3。          |

## ConnectionPoolSettings.TCPSettings

HTTP 和 TCP 上游连接通用的配置.

| 字段             | 类型                       | 描述                                  |
| ---------------- | -------------------------- | ------------------------------------- |
| `maxConnections` | `int32`                    | 到目标主机的 HTTP1 / TCP 最大连接数。 |
| `connectTimeout` | `google.protobuf.Duration` | TCP 连接超时时间。                    |

## CorsPolicy

描述给定服务的跨源资源共享（CORS）策略。有关跨源资源共享的更多详细信息，请参阅https://developer.mozilla.org/en-US/docs/Web/HTTP/AccesscontrolCORS。例如，以下规则将使用 HTTP POST / GET 的跨源请求限制为源自example.com域的跨源请求，并将 Access-Control-Allow-Credentials header设置为false。此外，它只暴露 X-Foo-bar header 并设置1天的到期时间。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        name: ratings
        subset: v1
    corsPolicy:
      allowOrigin:
      - example.com
      allowMethods:
      - POST
      - GET
      allowCredentials: false
      allowHeaders:
      - X-Foo-Bar
      maxAge: "1d"
```

| 字段               | 类型                        | 描述                                                         |
| ------------------ | --------------------------- | ------------------------------------------------------------ |
| `allowOrigin`      | `string[]`                  | 允许执行CORS请求的来源列表。内容将被序列化到Access-Control-Allow-Origin header中。通配符 `*` 将允许所有的来源。 |
| `allowMethods`     | `string[]`                  | 允许访问资源的 HTTP 方法列表。内容将被序列化到 Access-Control-Allow-Methods header中。 |
| `allowHeaders`     | `string[]`                  | 请求资源时可以使用的 HTTP header 列表。序列化为 Access-Control-Allow-Headers header。 |
| `exposeHeaders`    | `string[]`                  | 浏览器允许访问的 HTTP header 白名单。序列化到 Access-Control-Expose-Headers header。 |
| `maxAge`           | `google.protobuf.Duration`  | 指定预检请求结果可以缓存的时间长度。转换为 Access-Control-Max-Age header。 |
| `allowCredentials` | `google.protobuf.BoolValue` | 指示是否允许调用方使用证书发送实际请求（不是预检）。转换为 Access-Control-Allow-Credentials header。 |

## Destination

目的地指示网络可寻址服务，在处理路由规则后将请求/连接发送到这些服务。`destination.host` 应明确引用到服务注册中的服务。Istio的服务注册由平台服务注册（例如Kubernetes服务，Consul服务）中的所有服务以及通过[ServiceEntry](#ServiceEntry)资源声明的服务组成。

Kubernetes用户请注意：当使用短名称（例如“reviews”而不是“reviews.default.svc.cluster.local”）时，Istio将根据规则的命名空间解释短名称，而不是服务的命名空间。在"default"命名空间中包含主机“reviews”的规则将被解释为“reviews.default.svc.cluster.local”，而不管与review服务相关联的实际命名空间。为避免可能的错误配置，建议始终使用完全限定域名而不是短名称。

以下示例默认将所有流量路由到reviews服务的标签为“version：v1”（即子集v1）的pod，其中一些流量为v2子集，在kubernetes环境中。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
  namespace: foo
spec:
  hosts:
  - reviews # 解释为 reviews.foo.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews # 解释为 reviews.foo.svc.cluster.local
        subset: v2
  - route:
    - destination:
        host: reviews # 解释为 reviews.foo.svc.cluster.local
        subset: v1
```

而关联的 DestinationRule

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
  namespace: foo
spec:
  host: reviews # 解释为 reviews.foo.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

以下 VirtualService 为 Kubernetes 中的 productpage.prod.svc.cluster.local 服务的所有调用设置5秒的超时时间。请注意，此规则中没有定义子集。Istio 将从服务注册中获取所有的 productpage.prod.svc.cluster.local 的服务实例，并填充 sidecar 的负载平衡池。另请注意，此规则在 istio-system 命名空间中设置，但使用 productpage 服务的完全限定域名productpage.prod.svc.cluster.local。因此，规则的名称空间不会影响到 productpage 服务名称的解析。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-productpage-rule
  namespace: istio-system
spec:
  hosts:
  - productpage.prod.svc.cluster.local # 忽略规则命名空间
  http:
  - timeout: 5s
    route:
    - destination:
        host: productpage.prod.svc.cluster.local
```

要控制绑定到网格外服务的流量路由，必须先使用 ServiceEntry 资源将外部服务添加到 Istio 的内部服务注册中。然后可以定义 VirtualServices 来控制绑定到这些外部服务的流量。例如，以下规则定义 wikipedia.org 的服务并为 http 请求设置5秒的超时时间。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-wikipedia
spec:
  hosts:
  - wikipedia.org
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: example-http
    protocol: HTTP
  resolution: DNS

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-wiki-rule
spec:
  hosts:
  - wikipedia.org
  http:
  - timeout: 5s
    route:
    - destination:
        host: wikipedia.org

```

| 字段     | 类型           | 描述                                                         |
| -------- | -------------- | ------------------------------------------------------------ |
| `host`   | `string`       | 必填. 服务注册表服务的名称。从平台的服务注册（例如Kubernetes服务，Consul服务等）和ServiceEntry声明的主机中查找服务名称。如果在两者中找不到目的地，则转发到该目的地的流量将被丢弃。<br />Kubernetes用户请注意：当使用短名称（例如“reviews”而不是“reviews.default.svc.cluster.local”）时，Istio将根据规则的命名空间解释短名称，而不是服务的命名空间。在"default"命名空间中包含主机“reviews”的规则将被解释为“reviews.default.svc.cluster.local”，而不管与review服务相关联的实际命名空间。为避免可能的错误配置，建议始终使用完全限定域名而不是短名称。 |
| `subset` | `string`       | 服务中子集的名称。 仅适用于网格内的服务。该子集必须在相应的 DestinationRule 中定义。 |
| `port`   | `PortSelector` | 指定要寻址的主机上的端口。如果服务仅公开单个端口，则不需要显式选择端口。 |

## DestinationRule

`DestinationRule` 定义策略， 这些策略适用于在路由发生之后准备用于服务的流量。这些规则指定负载均衡的配置，来自sidecar的连接池大小和异常值检测设置，以检测并逐出负载均衡池中的不健康主机。例如，ratings服务的简单负载均衡策略如下所示：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
```

通过定义一个命名的子集并覆盖在服务级别指定的设置，可以指定特定于版本的策略。以下规则针对所有到达名为 testversion 子集的流量使用轮循(round robin)负载均衡策略，该子集由带有标签（version:v3）的端点（例如，pod）组成。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

**注意：**在路由规则明确向此子集发送流量之前，为子集指定的策略不会生效。

流量策略也可以定制为特定的端口。以下规则对所有到80端口的流量使用最小连接(least connection)负载均衡策略，而对到9080端口的流量使用轮循(round robin)负载均衡设置。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings-port
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy: # 适用于所有端口
    portLevelSettings:
    - port:
        number: 80
      loadBalancer:
        simple: LEAST_CONN
    - port:
        number: 9080
      loadBalancer:
        simple: ROUND_ROBIN
```

| 字段            | 类型            | 描述                                                         |
| --------------- | --------------- | ------------------------------------------------------------ |
| `host`          | `string`        | 必填. 服务注册表服务的名称。从平台的服务注册（例如Kubernetes服务，Consul服务等）和ServiceEntry声明的主机中查找服务名称。如果在两者中找不到目的地，则转发到该目的地的流量将被丢弃。<br />Kubernetes用户请注意：当使用短名称（例如“reviews”而不是“reviews.default.svc.cluster.local”）时，Istio将根据规则的命名空间解释短名称，而不是服务的命名空间。在"default"命名空间中包含主机“reviews”的规则将被解释为“reviews.default.svc.cluster.local”，而不管与review服务相关联的实际命名空间。为避免可能的错误配置，建议始终使用完全限定域名而不是短名称。<br />请注意，host字段同时适用于HTTP和TCP服务。 |
| `trafficPolicy` | `TrafficPolicy` | 要应用的流量策略（负载均衡策略，连接池大小，异常检测）。     |
| `subsets`       | `Subset[]`      | 代表服务的单个版本的一个或多个命名集。流量策略可以在子集级别上被覆盖。 |

## DestinationWeight

Each routing rule is associated with one or more service versions (see glossary in beginning of document). Weights associated with the version determine the proportion of traffic it receives. For example, the following rule will route 25% of traffic for the “reviews” service to instances with the “v2” tag and the remaining traffic (i.e., 75%) to “v1”.

每个路由规则都与一个或多个服务版本相关联（请参阅文档开头的术语表）。与版本关联的权重决定了它收到的流量的比例。例如，以下规则将将“reviews”服务的25％的流量路由到具有“v2”标签的实例并将剩余流量（即75％）路由到“v1”。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
      weight: 25
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
      weight: 75
```

而关联的 DestinationRule

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews.prod.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

Traffic can also be split across two entirely different services without having to define new subsets. For example, the following rule forwards 25% of traffic to reviews.com to dev.reviews.com

流量也可以分成两个完全不同的服务，而不必定义新的子集。例如，以下规则将到 reviews.com 的25％的流量转发到dev.reviews.com

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route-two-domains
spec:
  hosts:
  - reviews.com
  http:
  - route:
    - destination:
        host: dev.reviews.com
      weight: 25
    - destination:
        host: reviews.com
      weight: 75
```



| 字段          | 类型                        | 描述                                                         |
| ------------- | --------------------------- | ------------------------------------------------------------ |
| `destination` | [Destination](#Destination) | 必须。destination唯一标识请求/连接应转发到服务实例。         |
| `weight`      | `int32`                     | 必须。要转发到服务版本的流量比例（0-100）。目标之间的权重总和应该为100.如果规则中只有目的地，则权重值假定为100。 |

## Gateway

`网关`描述运行在网格边缘的负载均衡器，用于接收传入或传出的HTTP/TCP连接。该规范描述应该公开的一组端口，使用的协议类型，负载均衡器的SNI配置等。

例如，以下网关配置将搭建一个代理作为负载均衡器，用于暴露入口的端口80和9080（http），443（https）和端口2379（TCP）。该网关将应用于在带有`app：my-gateway-controller`标签的pod上运行的代理。虽然Istio会配置代理以监听这些端口，但用户有责任确保允许这些端口的外部流量进入网格。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    app: my-gatweway-controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - uk.bookinfo.com
    - eu.bookinfo.com
    tls:
      httpsRedirect: true # sends 302 redirect for http requests
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - uk.bookinfo.com
    - eu.bookinfo.com
    tls:
      mode: SIMPLE #enables HTTPS on this port
      serverCertificate: /etc/certs/servercert.pem
      privateKey: /etc/certs/privatekey.pem
  - port:
      number: 9080
      name: http-wildcard
      protocol: HTTP
    hosts:
    - "*"
  - port:
      number: 2379 # to expose internal service via external port 2379
      name: mongo
      protocol: MONGO
    hosts:
    - "*"
```

上面的网关规范描述负载均衡器的L4-L6属性。然后可以将 `VirtualService` 绑定到网关，以便控制到达特定主机或网关端口的流量转发。

例如，以下 VirtualService 会将“https://uk.bookinfo.com/reviews”，“https://eu.bookinfo.com/reviews”，“http://uk.bookinfo.com:9080/reviews“，”http://eu.bookinfo.com:9080/reviews“的流量拆分为端口9080上的内部 reviews 服务的两个版本（prod和qa）。此外，包含cookie "user：dev-123" 的请求将被送到qa版本的特殊端口7777。 对于 "reviews.prod.svc.cluster.local" 服务的请求，同样的规则也适用于网格内部。该规则适用于端口443,9080。请注意，“http://uk.bookinfo.com”被重定向到“https://uk.bookinfo.com”（即80重定向到443）。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-rule
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  - uk.bookinfo.com
  - eu.bookinfo.com
  gateways:
  - my-gateway
  - mesh # 适用于网格内的所有sidecar
  http:
  - match:
    - headers:
        cookie:
          user: dev-123
    route:
    - destination:
        port:
          number: 7777
        name: reviews.qa.svc.cluster.local
  - match:
      uri:
        prefix: /reviews/
    route:
    - destination:
        port:
          number: 9080 # 如果是reviews服务的唯一端口，可以省略
        name: reviews.prod.svc.cluster.local
      weight: 80
    - destination:
        name: reviews.qa.svc.cluster.local
      weight: 20
```

以下 VirtualService 将从 “172.17.16.0/24” 子网到（外部）27017端口的流量转发到内部Mongo服务器的5555端口。此规则不适用于网格内部，因为网关列表省略了保留名称 `mesh`。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-Mongo
spec:
  hosts:
  - mongosvr.prod.svc.cluster.local #name of internal Mongo service
  gateways:
  - my-gateway
  tcp:
  - match:
    - port:
        number: 27017
      sourceSubnet: "172.17.16.0/24"
    route:
    - destination:
        name: mongo.prod.svc.cluster.local
```

| 字段       | 类型                  | 描述                                                         |
| ---------- | --------------------- | ------------------------------------------------------------ |
| `servers`  | [`Server[]`](#Server) | 必须: 服务期规范列表                                         |
| `selector` | `map<string, string>` | 一个或多个标签，用于指示应用此网关配置的特定的Pod / VM集合。标签搜索的范围取决于平台。例如，在Kubernetes上，范围包括在所有可访问的命名空间中运行的pod。 |

## HTTPFaultInjection

HTTPFaultInjection 可以用来指定一个或多个要注入的故障，同时将http请求转发到路由中指定的目的地。故障规范是 VirtualService 规则的一部分。故障包括中止来自下游服务的Http请求，和/或延迟请求的代理。**故障规则必须**有延迟或中止，或两者都有。

注意：延迟和中止故障也是相互独立的，即使两者同时指定。

## HTTPFaultInjection.Abort

中止规范用于提前中止带有预定错误代码的请求。以下示例将为“ratings”服务“v1”的10％的请求返回HTTP 400错误代码。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
    fault:
      abort:
        percent: 10
        httpStatus: 400
```

`httpStatus`字段用于指示返回给调用者的HTTP状态码。可选的`percent`字段（0到100之间的值）用于仅中止特定百分比的请求。如果未指定，则所有请求都会中止。

| 字段         | 类型            | 描述                                          |
| ------------ | --------------- | --------------------------------------------- |
| `percent`    | `int32`         | 中止请求的百分比（0-100），使用提供的错误码。 |
| `httpStatus` | `int32 (oneof)` | 必须。用来种植HTTP请求的HTTP 状态码。         |

## HTTPFaultInjection.Delay

延迟规范用于将延迟注入请求转发路径。以下示例将引入5秒的延迟，给“reviews”服务的“v1”版本的10％的请求，这些服务来自所有带有 env: prod 标签的pod。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - match:
    - sourceLabels:
        env: prod
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
    fault:
      delay:
        percent: 10
        fixedDelay: 5s
```

The *fixedDelay* field is used to indicate the amount of delay in seconds. An optional *percent* field, a value between 0 and 100, can be used to only delay a certain percentage of requests. If left unspecified, all request will be delayed.

`fixedDelay`字段用于指定以秒为单位的延迟量。可选的`percent`字段（0到100之间的值）可用于延迟一定比例的请求。如果未指定，所有请求将被延迟。

| 字段         | 类型                               | 描述                                                         |
| ------------ | ---------------------------------- | ------------------------------------------------------------ |
| `percent`    | `int32`                            | 将要注入延迟的请求百分比(0-100).                             |
| `fixedDelay` | `google.protobuf.Duration (oneof)` | 必须. 在转发请求之前增加固定的延迟。格式： 1h/1m/1s/1ms. 必须大于等于 1ms. |

## HTTPMatchRequest

HttpMatchRequest 指定了一组要满足的规范，以便将规则应用于HTTP请求。例如以下限制规范仅匹配URL路径以 `/ratings/v2/` 开头并且请求包含值为 `user=jason` 的cookie的请求。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - headers:
        cookie:
          regex: "^(.*?;)?(user=jason)(;.*)?"
        uri:
          prefix: "/ratings/v2/"
    route:
    - destination:
        host: ratings.prod.svc.cluster.local
```

HTTPMatchRequest **不可以**为空。

| 字段           | 类型                          | 描述                                                         |
| -------------- | ----------------------------- | ------------------------------------------------------------ |
| `uri`          | [`StringMatch`](#StringMatch) | 用来匹配值的URI，区分大小写，格式如下：<br />- `exact: "value"` 用于精确字符串匹配<br />-`perfix: "value"` 用于基于前缀的匹配<br />- `regex: "value"` 用于ECMAscript风格的基于正则表达式的匹配 |
| `scheme`       | [`StringMatch`](#StringMatch) | URI的scheme值，区分大小写，格式如下：<br />- `exact: "value"` 用于精确字符串匹配<br />-`perfix: "value"` 用于基于前缀的匹配<br />- `regex: "value"` 用于ECMAscript风格的基于正则表达式的匹配 |
| `method`       | [`StringMatch`](#StringMatch) | HTTP方法的值，区分大小写，格式如下：<br />- `exact: "value"` 用于精确字符串匹配<br />-`perfix: "value"` 用于基于前缀的匹配<br />- `regex: "value"` 用于ECMAscript风格的基于正则表达式的匹配 |
| `authority`    | [`StringMatch`](#StringMatch) | HTTP Authority的值，区分大小写，格式如下：<br />- `exact: "value"` 用于精确字符串匹配<br />-`perfix: "value"` 用于基于前缀的匹配<br />- `regex: "value"` 用于ECMAscript风格的基于正则表达式的匹配 |
| `headers`      | `map<string, StringMatch>`    | header key 必须是小写字母，并使用连字符作为分隔符，例如 ` *x-request-id*`。 <br /><br />- `exact: "value"` 用于精确字符串匹配<br />-`perfix: "value"` 用于基于前缀的匹配<br />- `regex: "value"` 用于ECMAscript风格的基于正则表达式的匹配<br /><br />注意：key `uri`, `scheme`, `method`, 和 `authority` 将被忽略。 |
| `port`         | `uint32`                      | 指定将要寻址的主机上的端口。许多服务只暴露单个端口，或者用它们支持的协议给端口打了标签，在这种情况下，不需要显式选择端口。 |
| `sourceLabels` | `map<string, string>`         | 一个或多个标签，用于限制规则的适用性，使之只适用于具有给定标签的工作负载。如果 VirtualService 在顶部指定有网关列表，则它应该包括保留的 `mesh` 网管，以便适用这个字段。 |
| `gateways`     | `string[]`                    | 要应用规则的网关名称。VirtualService (如果有)顶部的网关名称将被覆盖。网关匹配独立于sourceLabels。 |

## HTTPRedirect

HTTPRedirect 可以用来向调用者发送302重定向响应，响应中的 Authority / Host 和URI可以与指定的值交换。例如，以下规则将 ratings 服务上的 `/v1/getProductRatings` API的请求重定向到 bookratings 服务提供的 `/v1/bookRatings`。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - uri:
        exact: /v1/getProductRatings
  redirect:
    uri: /v1/bookRatings
    authority: newratings.default.svc.cluster.local
  ...
```

| 字段        | 类型     | 描述                                                         |
| ----------- | -------- | ------------------------------------------------------------ |
| `uri`       | `string` | 在重定向时，使用此值覆盖URL的Path部分。 请注意，整个路径将被替换，而不管请求URI是精确路径匹配还是前缀匹配。 |
| `authority` | `string` | 在重定向时，使用该值覆盖URL的 Authority/Host 部分。          |

## HTTPRetry

描述在HTTP请求失败时使用的重试策略。例如，以下规则将在调用 ratings:v1 服务时的重试最大次数设置为3，每次重试尝试时有2秒的超时时间。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
```

| 字段            | 类型                       | 描述                                                         |
| --------------- | -------------------------- | ------------------------------------------------------------ |
| `attempts`      | `int32`                    | 必须。给定请求的重试次数。重试间隔将自动确定（25ms +）。实际尝试的重试次数取决于httpReqTimeout。 |
| `perTryTimeout` | `google.protobuf.Duration` | 对于给定请求，每次尝试重试的超时时间。格式： 1h/1m/1s/1ms. 必须大于等于 1ms. |

## HTTPRewrite

HTTPRewrite 可用于在将请求转发到目的地之前重写HTTP请求的特定部分。重写原语只能用于 DestinationWeights。以下示例演示如何在进行实际API调用之前重写到 ratings 服务的 api 调用（/ ratings）的URL前缀。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: /ratings
    rewrite:
      uri: /v1/bookRatings
    route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
```

| 字段        | 类型     | 描述                                                         |
| ----------- | -------- | ------------------------------------------------------------ |
| `uri`       | `string` | 使用这个值重写URI的path（或prefix）部分。如果原始URI是基于前缀匹配的，则此字段中提供的值将替换相应的被匹配的前缀。 |
| `authority` | `string` | 使用这个值重写 Authority/Host header。                       |

## HTTPRoute

描述用于路由 HTTP/1.1，HTTP2和gRPC流量的匹配条件和行动。有关使用示例，请参阅VirtualService。

| 字段               | 类型                       | 描述                                                         |
| ------------------ | -------------------------- | ------------------------------------------------------------ |
| `match`            | `HTTPMatchRequest[]`       | Match conditions to be satisfied for the rule to be activated. All conditions inside a single match block have AND semantics, while the list of match blocks have OR semantics. The rule is matched if any one of the match blocks succeed.要激活的规则满足匹配条件。 单个匹配块内的所有条件都具有AND语义，而匹配块列表具有OR语义。 如果任何一个匹配块成功，则匹配该规则。 |
| `route`            | `DestinationWeight[]`      | A http rule can either redirect or forward (default) traffic. The forwarding target can be one of several versions of a service (see glossary in beginning of document). Weights associated with the service version determine the proportion of traffic it receives. |
| `redirect`         | `HTTPRedirect`             | A http rule can either redirect or forward (default) traffic. If traffic passthrough option is specified in the rule, route/redirect will be ignored. The redirect primitive can be used to send a HTTP 302 redirect to a different URI or Authority. |
| `rewrite`          | `HTTPRewrite`              | Rewrite HTTP URIs and Authority headers. Rewrite cannot be used with Redirect primitive. Rewrite will be performed before forwarding. |
| `websocketUpgrade` | `bool`                     | Indicates that a HTTP/1.1 client connection to this particular route should be allowed (and expected) to upgrade to a WebSocket connection. The default is false. Istio’s reference sidecar implementation (Envoy) expects the first request to this route to contain the WebSocket upgrade headers. Otherwise, the request will be rejected. Note that Websocket allows secondary protocol negotiation which may then be subject to further routing rules based on the protocol selected. |
| `timeout`          | `google.protobuf.Duration` | Timeout for HTTP requests.                                   |
| `retries`          | `HTTPRetry`                | Retry policy for HTTP requests.                              |
| `mirror`           | `Destination`              | Mirror HTTP traffic to a another destination in addition to forwarding the requests to the intended destination. Mirrored traffic is on a best effort basis where the sidecar/gateway will not wait for the mirrored cluster to respond before returning the response from the original destination. Statistics will be generated for the mirrored destination. |
| `corsPolicy`       | `CorsPolicy`               | Cross-Origin Resource Sharing policy (CORS). Refer to https://developer.mozilla.org/en-US/docs/Web/HTTP/Access*control*CORS for further details about cross origin resource sharing. |
| `appendHeaders`    | `map<string, string>`      | Additional HTTP headers to add before forwarding a request to the destination service. |

## L4MatchAttributes

L4 connection match attributes. Note that L4 connection matching support is incomplete.

| Field               | Type                  | Description                                                  |
| ------------------- | --------------------- | ------------------------------------------------------------ |
| `destinationSubnet` | `string`              | IPv4 or IPv6 ip address of destination with optional subnet. E.g., a.b.c.d/xx form or just a.b.c.d. This is only valid when the destination service has several IPs and the application explicitly specifies a particular IP. |
| `port`              | `PortSelector`        | Specifies the port on the host that is being addressed. Many services only expose a single port or label ports with the protocols they support, in these cases it is not required to explicitly select the port. Note that selection priority is to first match by name and then match by number.Names must comply with DNS label syntax (rfc1035) and therefore cannot collide with numbers. If there are multiple ports on a service with the same protocol the names should be of the form-. |
| `sourceSubnet`      | `string`              | IPv4 or IPv6 ip address of source with optional subnet. E.g., a.b.c.d/xx form or just a.b.c.d |
| `sourceLabels`      | `map<string, string>` | One or more labels that constrain the applicability of a rule to workloads with the given labels. If the VirtualService has a list of gateways specified at the top, it should include the reserved gateway “mesh” in order for this field to be applicable. |
| `gateways`          | `string[]`            | Names of gateways where the rule should be applied to. Gateway names at the top of the VirtualService (if any) are overridden. The gateway match is independent of sourceLabels. |

## LoadBalancerSettings

Load balancing policies to apply for a specific destination. See Envoy’s load balancing [documentation](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing.html) for more details.

For example, the following rule uses a round robin load balancing policy for all traffic going to the ratings service.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  name: ratings
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
```

The following example uses the consistent hashing based load balancer for the same ratings service using the Cookie header as the hash key.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  name: ratings
  trafficPolicy:
    loadBalancer:
      consistentHash:
        http_header: Cookie
```

| Field            | Type                                            | Description |
| ---------------- | ----------------------------------------------- | ----------- |
| `simple`         | `LoadBalancerSettings.SimpleLB (oneof)`         |             |
| `consistentHash` | `LoadBalancerSettings.ConsistentHashLB (oneof)` |             |

## LoadBalancerSettings.ConsistentHashLB

Consistent hashing (ketama hash) based load balancer for even load distribution/redistribution when the connection pool changes. This load balancing policy is applicable only for HTTP-based connections. A user specified HTTP header is used as the key with [xxHash](http://cyan4973.github.io/xxHash) hashing.

| Field             | Type     | Description                                                  |
| ----------------- | -------- | ------------------------------------------------------------ |
| `httpHeader`      | `string` | REQUIRED. The name of the HTTP request header that will be used to obtain the hash key. If the request header is not present, the load balancer will use a random number as the hash, effectively making the load balancing policy random. |
| `minimumRingSize` | `uint32` | The minimum number of virtual nodes to use for the hash ring. Defaults to 1024. Larger ring sizes result in more granular load distributions. If the number of hosts in the load balancing pool is larger than the ring size, each host will be assigned a single virtual node. |

## LoadBalancerSettings.SimpleLB

Standard load balancing algorithms that require no tuning.

| Name          | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| `ROUND_ROBIN` | Round Robin policy. Default                                  |
| `LEAST_CONN`  | The least request load balancer uses an O(1) algorithm which selects two random healthy hosts and picks the host which has fewer active requests. |
| `RANDOM`      | The random load balancer selects a random healthy host. The random load balancer generally performs better than round robin if no health checking policy is configured. |
| `PASSTHROUGH` | This option will forward the connection to the original IP address requested by the caller without doing any form of load balancing. This option must be used with care. It is meant for advanced use cases. Refer to Original Destination load balancer in Envoy for further details. |

## OutlierDetection

A Circuit breaker implementation that tracks the status of each individual host in the upstream service. While currently applicable to only HTTP services, future versions will support opaque TCP services as well. For HTTP services, hosts that continually return errors for API calls are ejected from the pool for a pre-defined period of time. See Envoy’s [outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/outlier) for more details.

The following rule sets a connection pool size of 100 connections and 1000 concurrent HTTP2 requests, with no more than 10 req/connection to “reviews” service. In addition, it configures upstream hosts to be scanned every 5 mins, such that any host that fails 7 consecutive times with 5XX error code will be ejected for 15 minutes.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-cb-policy
spec:
  name: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    outlierDetection:
      http:
        consecutiveErrors: 7
        interval: 5m
        baseEjectionTime: 15m
```

| Field  | Type                            | Description                                  |
| ------ | ------------------------------- | -------------------------------------------- |
| `http` | `OutlierDetection.HTTPSettings` | Settings for HTTP1.1/HTTP2/GRPC connections. |

## OutlierDetection.HTTPSettings

Outlier detection settings for HTTP1.1/HTTP2/GRPC connections.

| Field                | Type                       | Description                                                  |
| -------------------- | -------------------------- | ------------------------------------------------------------ |
| `consecutiveErrors`  | `int32`                    | Number of 5XX errors before a host is ejected from the connection pool. Defaults to 5. |
| `interval`           | `google.protobuf.Duration` | Time interval between ejection sweep analysis. format: 1h/1m/1s/1ms. MUST BE >=1ms. Default is 10s. |
| `baseEjectionTime`   | `google.protobuf.Duration` | Minimum ejection duration. A host will remain ejected for a period equal to the product of minimum ejection duration and the number of times the host has been ejected. This technique allows the system to automatically increase the ejection period for unhealthy upstream servers. format: 1h/1m/1s/1ms. MUST BE >=1ms. Default is 30s. |
| `maxEjectionPercent` | `int32`                    | Maximum % of hosts in the load balancing pool for the upstream service that can be ejected. Defaults to 10%. |

## Port

Port describes the properties of a specific port of a service.

| Field      | Type     | Description                                                  |
| ---------- | -------- | ------------------------------------------------------------ |
| `number`   | `uint32` | REQUIRED: A valid non-negative integer port number.          |
| `protocol` | `string` | The protocol exposed on the port. MUST BE one of HTTP\|HTTPS\|GRPC\|HTTP2\|MONGO\|TCP. |
| `name`     | `string` | Label assigned to the port.                                  |

## PortSelector

PortSelector specifies the name or number of a port to be used for matching or selection for final routing.

| Field    | Type             | Description       |
| -------- | ---------------- | ----------------- |
| `number` | `uint32 (oneof)` | Valid port number |
| `name`   | `string (oneof)` | Port name         |

## Server

Server describes the properties of the proxy on a given load balancer port. For example,

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-ingress
spec:
  selector:
    app: my-ingress-controller
  servers:
  - port:
      number: 80
      protocol: HTTP2
```

Another example

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-tcp-ingress
spec:
  selector:
    app: my-tcp-ingress-controller
  servers:
  - port:
      number: 27018
      protocol: MONGO
```

The following is an example of TLS configuration for port 443

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-tls-ingress
spec:
  selector:
    app: my-tls-ingress-controller
  servers:
  - port:
      number: 443
      protocol: HTTP
    tls:
      mode: simple
      serverCertificate: /etc/certs/server.pem
      privateKey: /etc/certs/privatekey.pem
```

| Field   | Type                | Description                                                  |
| ------- | ------------------- | ------------------------------------------------------------ |
| `port`  | `Port`              | REQUIRED: The Port on which the proxy should listen for incoming connections |
| `hosts` | `string[]`          | A list of hosts exposed by this gateway. While typically applicable to HTTP services, it can also be used for TCP services using TLS with SNI. Standard DNS wildcard prefix syntax is permitted.A VirtualService that is bound to a gateway must having a matching host in its default destination. Specifically one of the VirtualService destination hosts is a strict suffix of a gateway host or a gateway host is a suffix of one of the VirtualService hosts. |
| `tls`   | `Server.TLSOptions` | Set of TLS related options that govern the server’s behavior. Use these options to control if all http requests should be redirected to https, and the TLS modes to use. |

## Server.TLSOptions

| Field               | Type                        | Description                                                  |
| ------------------- | --------------------------- | ------------------------------------------------------------ |
| `httpsRedirect`     | `bool`                      | If set to true, the load balancer will send a 302 redirect for all http connections, asking the clients to use HTTPS. |
| `mode`              | `Server.TLSOptions.TLSmode` | Optional: Indicates whether connections to this port should be secured using TLS. The value of this field determines how TLS is enforced. |
| `serverCertificate` | `string`                    | REQUIRED if mode is “simple” or “mutual”. The path to the file holding the server-side TLS certificate to use. |
| `privateKey`        | `string`                    | REQUIRED if mode is “simple” or “mutual”. The path to the file holding the server’s private key. |
| `caCertificates`    | `string`                    | REQUIRED if mode is “mutual”. The path to a file containing certificate authority certificates to use in verifying a presented client side certificate. |
| `subjectAltNames`   | `string[]`                  | A list of alternate names to verify the subject identity in the certificate presented by the client. |

## Server.TLSOptions.TLSmode

TLS modes enforced by the proxy

| Name          | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| `PASSTHROUGH` | If set to “passthrough”, the proxy will forward the connection to the upstream server selected based on the SNI string presented by the client. |
| `SIMPLE`      | If set to “simple”, the proxy will secure connections with standard TLS semantics. |
| `MUTUAL`      | If set to “mutual”, the proxy will secure connections to the upstream using mutual TLS by presenting client certificates for authentication. |

## StringMatch

Describes how to match a given string in HTTP headers. Match is case-sensitive.

| Field    | Type             | Description                        |
| -------- | ---------------- | ---------------------------------- |
| `exact`  | `string (oneof)` | exact string match                 |
| `prefix` | `string (oneof)` | prefix-based match                 |
| `regex`  | `string (oneof)` | ECMAscript style regex-based match |

## Subset

A subset of endpoints of a service. Subsets can be used for scenarios like A/B testing, or routing to a specific version of a service. Refer to VirtualService documentation for examples of using subsets in these scenarios. In addition, traffic policies defined at the service-level can be overridden at a subset-level. The following rule uses a round robin load balancing policy for all traffic going to a subset named testversion that is composed of endpoints (e.g., pods) with labels (version:v3).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  name: ratings
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

Note that policies specified for subsets will not take effect until a route rule explicitly sends traffic to this subset.

| Field           | Type                  | Description                                                  |
| --------------- | --------------------- | ------------------------------------------------------------ |
| `name`          | `string`              | REQUIRED. name of the subset. The service name and the subset name can be used for traffic splitting in a route rule. |
| `labels`        | `map<string, string>` | REQUIRED. Labels apply a filter over the endpoints of a service in the service registry. See route rules for examples of usage. |
| `trafficPolicy` | `TrafficPolicy`       | Traffic policies that apply to this subset. Subsets inherit the traffic policies specified at the DestinationRule level. Settings specified at the subset level will override the corresponding settings specified at the DestinationRule level. |

## TCPRoute

Describes match conditions and actions for routing TCP traffic. The following routing rule forwards traffic arriving at port 2379 named Mongo from 172.17.16.* subnet to another Mongo server on port 5555.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-Mongo
spec:
  hosts:
  - myMongosrv 
  tcp:
  - match:
    - port:
        name: Mongo # only applies to ports named Mongo
      sourceSubnet: "172.17.16.0/24"
    route:
    - destination:
        name: mongo.prod
```

| Field   | Type                  | Description                                                  |
| ------- | --------------------- | ------------------------------------------------------------ |
| `match` | `L4MatchAttributes[]` | Match conditions to be satisfied for the rule to be activated. All conditions inside a single match block have AND semantics, while the list of match blocks have OR semantics. The rule is matched if any one of the match blocks succeed. |
| `route` | `DestinationWeight[]` | The destination to which the connection should be forwarded to. Currently, only one destination is allowed for TCP services. When TCP weighted routing support is introduced in Envoy, multiple destinations with weights can be specified. |

## TLSSettings

SSL/TLS related settings for upstream connections. See Envoy’s [TLS context](https://www.envoyproxy.io/docs/envoy/latest/api-v1/cluster_manager/cluster_ssl.html#config-cluster-manager-cluster-ssl) for more details. These settings are common to both HTTP and TCP upstreams.

For example, the following rule configures a client to use mutual TLS for connections to upstream database cluster.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: db-mtls
spec:
  name: mydbserver
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

The following rule configures a client to use TLS when talking to a foreign service whose domain matches *.foo.com.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: tls-foo
spec:
  name: "*.foo.com"
  trafficPolicy:
    tls:
      mode: SIMPLE
```

| Field               | Type                  | Description                                                  |
| ------------------- | --------------------- | ------------------------------------------------------------ |
| `mode`              | `TLSSettings.TLSmode` | REQUIRED: Indicates whether connections to this port should be secured using TLS. The value of this field determines how TLS is enforced. |
| `clientCertificate` | `string`              | REQUIRED if mode is “mutual”. The path to the file holding the client-side TLS certificate to use. |
| `privateKey`        | `string`              | REQUIRED if mode is “mutual”. The path to the file holding the client’s private key. |
| `caCertificates`    | `string`              | OPTIONAL: The path to the file containing certificate authority certificates to use in verifying a presented server certificate. If omitted, the proxy will not verify the server’s certificate. |
| `subjectAltNames`   | `string[]`            | A list of alternate names to verify the subject identity in the certificate. If specified, the proxy will verify that the server certificate’s subject alt name matches one of the specified values. |
| `sni`               | `string`              | SNI string to present to the server during TLS handshake.    |

## TLSSettings.TLSmode

TLS connection mode

| Name      | Description                                                  |
| --------- | ------------------------------------------------------------ |
| `DISABLE` | If set to “disable”, the proxy will use not setup a TLS connection to the upstream server. |
| `SIMPLE`  | If set to “simple”, the proxy will originate a TLS connection to the upstream server. |
| `MUTUAL`  | If set to “mutual”, the proxy will secure connections to the upstream using mutual TLS by presenting client certificates for authentication. |

## TrafficPolicy

Traffic policies to apply for a specific destination. See DestinationRule for examples.

| Field              | Type                     | Description                                                  |
| ------------------ | ------------------------ | ------------------------------------------------------------ |
| `loadBalancer`     | `LoadBalancerSettings`   | Settings controlling the load balancer algorithms.           |
| `connectionPool`   | `ConnectionPoolSettings` | Settings controlling the volume of connections to an upstream service |
| `outlierDetection` | `OutlierDetection`       | Settings controlling eviction of unhealthy hosts from the load balancing pool |
| `tls`              | `TLSSettings`            | TLS related settings for connections to the upstream service. |

## VirtualService

A VirtualService defines a set of traffic routing rules to apply when a host is addressed. Each routing rule defines matching criteria for traffic of a specific protocol. If the traffic is matched, then it is sent to a named destination service (or subset/version of it) defined in the registry.

The source of traffic can also be matched in a routing rule. This allows routing to be customized for specific client contexts.

The following example routes all HTTP traffic by default to pods of the reviews service with label “version: v1”. In addition, HTTP requests containing /wpcatalog/, /consumercatalog/ url prefixes will be rewritten to /newcatalog and sent to pods with label “version: v2”. The rules will be applied at the gateway named “bookinfo” as well as at all the sidecars in the mesh (indicated by the reserved gateway name “mesh”).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  gateways: # if omitted, defaults to "mesh"
  - bookinfo
  - mesh
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        name: reviews
        subset: v2
  - route:
    - destination:
        name: reviews
        subset: v1
```

A subset/version of a route destination is identified with a reference to a named service subset which must be declared in a corresponding DestinationRule.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  name: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

A host name can be defined by only one VirtualService. A single VirtualService can be used to describe traffic properties for multiple HTTP and TCP ports.

| Field      | Type          | Description                                                  |
| ---------- | ------------- | ------------------------------------------------------------ |
| `hosts`    | `string[]`    | REQUIRED. The destination address for traffic captured by this virtual service. Could be a DNS name with wildcard prefix or a CIDR prefix. Depending on the platform, short-names can also be used instead of a FQDN (i.e. has no dots in the name). In such a scenario, the FQDN of the host would be derived based on the underlying platform.For example on Kubernetes, when hosts contains a short name, Istio will interpret the short name based on the namespace of the rule. Thus, when a client namespace applies a rule in the “default” namespace containing a name “reviews, Istio will setup routes to the “reviews.default.svc.cluster.local” service. However, if a different name such as “reviews.sales.svc.cluster.local” is used, it would be treated as a FQDN during virtual host matching. In Consul, a plain service name would be resolved to the FQDN “reviews.service.consul”.Note that the hosts field applies to both HTTP and TCP services. Service inside the mesh, i.e., those found in the service registry, must always be referred to using their alphanumeric names. IP addresses or CIDR prefixes are allowed only for services defined via the Gateway. |
| `gateways` | `string[]`    | The names of gateways and sidecars that should apply these routes. A single VirtualService is used for sidecars inside the mesh as well as for one or more gateways. The selection condition imposed by this field can be overridden using the source field in the match conditions of HTTP/TCP routes. The reserved word “mesh” is used to imply all the sidecars in the mesh. When this field is omitted, the default gateway (“mesh”) will be used, which would apply the rule to all sidecars in the mesh. If a list of gateway names is provided, the rules will apply only to the gateways. To apply the rules to both gateways and sidecars, specify “mesh” as one of the gateway names. |
| `http`     | `HTTPRoute[]` | An ordered list of route rules for HTTP traffic. The first rule matching an incoming request is used. |
| `tcp`      | `TCPRoute[]`  | An ordered list of route rules for TCP traffic. The first rule matching an incoming request is used. |