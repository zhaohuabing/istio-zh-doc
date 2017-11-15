# Mixer配置

本节介绍Mixer的配置模型。

## 背景

Istio是一个具有数百个独立功能的复杂系统。Istio部署可能涉及数十个服务的蔓延事件，这些服务有一群Envoy代理和Mixer实例来支持它们。在大型部署中，许多不同的运维人员（每个运维人员都有不同的范围和责任范围）可能涉及管理整体部署。

Mixer的配置模式可以利用其所有功能和灵活性，同时保持使用的相对简单。该模型的范围特征使大型支持组织能够轻松地集中管理复杂的部署。该模型的一些主要功能包括：

- **专为运维人员而设计**：服务运维人员通过操纵配置资源来控制Mixer部署中的所有操作和策略切面。

- **灵活**：配置模型围绕Istio的[属性](./attributes.md)构建，使运维人员能够对部署中使用的策略和生成的遥测进行前所未有的控制。

- **健壮**：配置模型旨在提供最大的静态正确性保证，以帮助减少因为错误的配置变更导致的服务中断事故。

- **扩展**：该模型旨在支持Istio的整体可扩展性思路。可以将新的或自定义的[适配器](./mixer.md#适配器)添加到Istio中，并可以使用与现有适配器相同的通用机制进行完全操作。

## 概念

Mixer是一种属性处理机器。请求到达Mixer时带有一组[属性](./attributes.md) ，并且基于这些属性，Mixer会生成对各种基础设施后端的调用。这些后端包括频率限制、访问控制、策略实施等各种系统。该属性集确定Mixer为给定的请求用哪些参数调用哪个后端。为了隐藏各个后端的细节，Mixer使用称为[适配器](./mixer.md#适配器)的模块。

<img src="./img/mixer-config/machine.svg" alt="Attribute Machine" title="Attribute Machine" />

Mixer的配置有几个中心职责：

- 描述哪些适配器正在使用以及它们的操作方式。

- 描述如何将请求属性映射到适配器参数中。

- 描述使用特定参数调用适配器的时机。

配置基于适配器和模板来完成：

- **适配器** 封装了Mixer和特定基础设施后端之间的接口。

- **模板** 定义了从特定请求的属性到适配器输入的映射关系。一个适配器可以支持任意数量的模板。

配置使用YAML格式来表示,围绕几个核心抽象构建：

|概念                     |描述|
|----------------------------|-----------|
|[Handler](#handlers)|Handlers就是一个配置完成的适配器。适配器的构造器参数就是Handler的配置。|
|[实例](#instances)|一个（请求）实例就是请求属性到一个模板的映射结果。这种映射来自于实例的配置。|
|[规则](#rules)|规则确定了何时使用一个特定的模板配置来调用一个Handler。|

配置的资源对象可以使用Kubernetes的资源语法来进行表述：

```yaml
apiVersion: config.istio.io/v1alpha2
kind: rule, adapter kind, or template kind
metadata:
  name: shortname
  namespace: istio-system
spec:
  # 不同kind会有各自特定的配置
```

- **apiVersion**：常量，取决于Istio的版本

- **kind**：Mixer中的适配器和模板都有对应的类型。

- **name**：配置资源名称。

- **namespace**：配置所在的命名空间。

- **spec**：`kind`所对应的配置。

### Handler

[适配器](mixer.md#适配器)封装了Mixer和特定外部基础设施后端进行交互的必要接口，例如[Prometheus](https://prometheus.io/)、[New Relic](https://newrelic.com/)或者[Stackdriver](https://cloud.google.com/logging)。各种适配器都需要参数配置才能工作。例如日志适配器可能需要IP地址和端口来进行日志的输出。

这里的例子配置了一个类型为`listchecker`的适配器。listchecker适配器使用一个列表来检查输入。如果配置的是白名单模式且输入值存在于列表之中，就会返回成功的结果。

```yaml
apiVersion: config.istio.io/v1alpha2
kind: listchecker
metadata:
  name: staticversion
  namespace: istio-system
spec:
  providerUrl: http://white_list_registry/
  blacklist: false
```

`{metadata.name}.{kind}.{metadata.namespace}`是Handler的完全限定名。上面定义的对象的FQDN就是`staticversion.listchecker.istio-system`，他必须是唯一的。`spec`中的数据结构则依赖于对应的适配器的要求。

有些适配器实现的功能就不仅仅是把Mixer和后端连接起来。例如`prometheus`适配器使用一种可配置的方式，消费指标并对其进行聚合：

```yaml
apiVersion: config.istio.io/v1alpha2
kind: prometheus
metadata:
  name: handler
  namespace: istio-system
spec:
  metrics:
  - name: request_count
    instance_name: requestcount.metric.istio-system
    kind: COUNTER
    label_names:
    - destination_service
    - destination_version
    - response_code
  - name: request_duration
    instance_name: requestduration.metric.istio-system
    kind: DISTRIBUTION
    label_names:
    - destination_service
    - destination_version
    - response_code
    buckets:
      explicit_buckets:
        bounds: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
```

每个适配器都定义了自己格式的配置数据。适配器及其配置的详尽列表可以在[这里](../../reference/config/mixer/adapters/)找到。

### 实例

配置实例将请求中的属性映射成为适配器的输入。下面的例子，是一个指标实例的配置，用于生成`requestduration`指标：

```yaml
apiVersion: config.istio.io/v1alpha2
kind: metric
metadata:
  name: requestduration
  namespace: istio-system
spec:
  value: response.duration | "0ms"
  dimensions:
    destination_service: destination.service | "unknown"
    destination_version: destination.labels["version"] | "unknown"
    response_code: response.code | 200
  monitored_resource_type: '"UNSPECIFIED"'
```

注意Handler配置中需要的所有维度都定义在这一映射之中。

每个模板都有自己格式的配置数据。完整的模板及其特定配置格式可以在[这里](../../reference/config/mixer/template/)查阅。

### 规则

规则用于指定使用特定实例配置调用某一Handler的时机。比如我们想要把`service1`服务中，请求头中带有`x-user`的请求的`requestduration`指标发送给Prometheus Handler:

```yaml
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: promhttp
  namespace: istio-system
spec:
  match: destination.service == "service1.ns.svc.cluster.local" && request.headers["x-user"] == "user1"
  actions:
  - handler: handler.prometheus
    instances:
    - requestduration.metric.istio-system
```

规则对象中包含有一个`match`元素，用于前置检查，如果检查通过则会执行动作列表。动作中包含了一个实例列表，这个列表将会分发给Handler。规则必须使用Handler和实例的完全限定名。如果规则、Handler以及实例全都在同一个命名空间，命名空间后缀就可以在FQDN中省略，例如`handler.prometheus`。

匹配检查使用的是属性表达式，下面将会讲解。

#### 属性表达式

Mixer具有多个独立的[请求处理阶段](./mixer.md#请求阶段) 。**属性处理阶段** 负责摄取一组属性，并产生调用各个适配器时需要的模板的实例。该阶段通过评估一系列的 **属性表达式** 来运行。

在前面的例子中，我们已经看到了一些简单的属性表达式。特别是：

```yaml
  destination_service: destination.service
  response_code: response.code
  destination_version: destination.labels["version"] | "unknown"
```

冒号右侧的序列是属性表达式的最简单形式。前两行只包括了属性名称。`response_code`标签的内容来自于`request.code`属性。

以下是条件表达式的示例：

```yaml
  destination_version: destination.labels["version"] | "unknown"
```

上面的表达式里，`destination_version`标签被赋值为`destination.labels["version"]`，如果`destination.labels["version"]`为空，则使用`"unknown"`代替。

在属性表达式中可用的属性必须符合该部署中的[属性清单](#manifests)。在清单中，每个属性都有一个用于描述属性所代表数据的数据类型。同样的，属性表达式也是有类型的，表达式中的属性类型以对这些属性的操作决定了表达式的数据类型。

有关详细信息，请参阅[属性表达式引用](../../reference/config/mixer/expression-language.md)。

#### 决议

当请求到达时，Mixer会经过多个[请求处理阶段](./mixer.md#请求阶段)。决议阶段涉及确定要用于处理传入请求的确切配置块。例如，到达 Mixer 的 service A 的请求可能与 service B 的请求有一些配置差异。决议决定哪个配置用于请求。

决议依赖众所周知的属性来指导其选择，即所谓的**身份属性**。该属性的值是一个点号分隔的名称，它决定了Mixer在层次结构中从哪里开始查找用于请求的配置块。

这是它的工作原理：

1. 请求到达, Mixer提取身份属性的值以产生当前的查找值。

2. Mixer查找主题与查找值匹配的所有配置块。

3. 如果Mixer找到多个匹配的块，则它只保留具有最大范围的块。

4. Mixer从查找值的点号分割名称中截断最低元素。如果查找值不为空，则Mixer将返回上述步骤2。

在此过程中发现的所有块都组合在一起，形成用于最终的有效配置评估当前的请求。

### 清单

清单捕获特定Istio部署中涉及到的组件的不变量。当前唯一支持的清单是**属性清单**，用于定义由各个组件生成的属性的精确集合。清单由组件生成器提供并插入到部署的配置中。

以下是Istio代理的清单的一部分：

```yaml
manifests:
  - name: istio-proxy
    revision: "1"
    attributes:
      source.name:
        valueType: STRING
        description: The name of the source.
      target.name:
        valueType: STRING
        description: The name of the target
      source.ip:
        valueType: IP_ADDRESS
        description: Did you know that descriptions are optional?
      origin.user:
        valueType: STRING
      request.time:
        valueType: TIMESTAMP
      request.method:
        valueType: STRING
      response.code:
        valueType: INT64
```

## 例子

您可以通过访问[指南](../../guides)找到Mixer配置的完整示例。这里有一些[示例配置](https://github.com/istio/istio/blob/master/mixer/testdata/config)。

## 下一步

* 阅读阐述Mixer适配器模型的[博客文章](https://istio.io/blog/mixer-adapter-model.html)
