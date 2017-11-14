# 收集监控信息和日志

此任务显示如何配置Istio以自动收集网格中服务的遥测数据。 在此任务结束时，将启用一个新的度量标准和一个新的日志流，用于调用网格中的服务。

[BookInfo](https://istio.io/docs/guides/bookinfo.html)示例应用程序在整个任务中用作示例应用程序。

## 环境准备
---

- 在您的群集中安装Istio并部署一个应用程序。 这个任务假设调音台是以默认配置（--configDefaultNamespace = istio-system）设置的。 如果使用不同的值，则更新此任务中的配置和命令以匹配值。

- 安装Prometheus插件。 普罗米修斯将被用来验证任务的成功。

<pre><code>kubectl apply -f install/kubernetes/addons/prometheus.yaml
</pre></code>

详细信息请看[Prometheus](https://prometheus.io)。

## 收集监控数据
---

1. 创建一个新的YAML文件来保存Istio将自动生成和收集的新度量和日志流的配置。

把以下内容保持成```new_telemetry.yaml```:

<pre><code># Configuration for metric instances
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: doublerequestcount
  namespace: istio-system
spec:
  value: "2" # count each request twice
  dimensions:
    source: source.service | "unknown"
    destination: destination.service | "unknown"
    message: '"twice the fun!"'
  monitored_resource_type: '"UNSPECIFIED"'
---
# Configuration for a Prometheus handler
apiVersion: "config.istio.io/v1alpha2"
kind: prometheus
metadata:
  name: doublehandler
  namespace: istio-system
spec:
  metrics:
  - name: double_request_count # Prometheus metric name
    instance_name: doublerequestcount.metric.istio-system # Mixer instance name (fully-qualified)
    kind: COUNTER
    label_names:
    - source
    - destination
    - message
---
# Rule to send metric instances to a Prometheus handler
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: doubleprom
  namespace: istio-system
spec:
  actions:
  - handler: doublehandler.prometheus
    instances:
    - doublerequestcount.metric
---
# Configuration for logentry instances
apiVersion: "config.istio.io/v1alpha2"
kind: logentry
metadata:
  name: newlog
  namespace: istio-system
spec:
  severity: '"warning"'
  timestamp: request.time
  variables:
    source: source.labels["app"] | source.service | "unknown"
    user: source.user | "unknown"
    destination: destination.labels["app"] | destination.service | "unknown"
    responseCode: response.code | 0
    responseSize: response.size | 0
    latency: response.duration | "0ms"
  monitored_resource_type: '"UNSPECIFIED"'
---
# Configuration for a stdio handler
apiVersion: "config.istio.io/v1alpha2"
kind: stdio
metadata:
  name: newhandler
  namespace: istio-system
spec:
 severity_levels:
   warning: 1 # Params.Level.WARNING
 outputAsJson: true
---
# Rule to send logentry instances to a stdio handler
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: newlogstdio
  namespace: istio-system
spec:
  match: "true" # match for all requests
  actions:
   - handler: newhandler.stdio
     instances:
     - newlog.logentry
---
</pre></code>

2. 推新配置。

<pre><code>istioctl create -f new_telemetry.yaml
</pre></code>

输出类似如下内容：

<pre><code>Created config metric/istio-system/doublerequestcount at revision 1973035
Created config prometheus/istio-system/doublehandler at revision 1973036
Created config rule/istio-system/doubleprom at revision 1973037
Created config logentry/istio-system/newlog at revision 1973038
Created config stdio/istio-system/newhandler at revision 1973039
Created config rule/istio-system/newlogstdio at revision 1973041
</pre></code>

3. 将流量发送到示例应用程序。

参考Bookinfo的例子，在浏览器中访问```http://$GATEWAY_URL/productpage```或者使用以下命令:


<pre><code>curl http://$GATEWAY_URL/productpage
</pre></code>

4. 验证新的监控指标信息已产生并且能被收集。

在Kubernetes环境中，通过执行以下命令为Prometheus设置端口转发：

<pre><code>kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &</pre></code>

通过[Prometheus UI](http://localhost:9090/graph#[{"range_input":"1h","expr":"double_request_count","tab":1}])查看新监控指标信息。

打开Prometheus UI，查询```double_request_count```监控指标。 “Console”（控制台）tab页中有类似如下信息显示：

<pre><code>double_request_count{destination="details.default.svc.cluster.local",instance="istio-mixer.istio-system:42422",job="istio-mesh",message="twice the fun!",source="productpage.default.svc.cluster.local"}	2
double_request_count{destination="ingress.istio-system.svc.cluster.local",instance="istio-mixer.istio-system:42422",job="istio-mesh",message="twice the fun!",source="unknown"}	2
double_request_count{destination="productpage.default.svc.cluster.local",instance="istio-mixer.istio-system:42422",job="istio-mesh",message="twice the fun!",source="ingress.istio-system.svc.cluster.local"}	2
double_request_count{destination="reviews.default.svc.cluster.local",instance="istio-mixer.istio-system:42422",job="istio-mesh",message="twice the fun!",source="productpage.default.svc.cluster.local"}	2</pre></code>

有关查询Prometheus监控指标的更多信息，请参阅[Querying Istio Metrics](https://istio.io/docs/tasks/telemetry/querying-metrics.html)。


5. 验证日志流是否已创建并正在填充请求。

在Kubernetes环境中，使用以下方式搜索Mixer pod窗格的日志：

<pre><code>kubectl -n istio-system logs $(kubectl -n istio-system get pods -l istio=mixer -o jsonpath='{.items[0].metadata.name}') mixer | grep \"instance\":\"newlog.logentry.istio-system\"</pre></code>

输出类似以下内容：

<pre><code>{"level":"warn","ts":"2017-09-21T04:33:31.249Z","instance":"newlog.logentry.istio-system","destination":"details","latency":"6.848ms","responseCode":200,"responseSize":178,"source":"productpage","user":"unknown"}
{"level":"warn","ts":"2017-09-21T04:33:31.291Z","instance":"newlog.logentry.istio-system","destination":"ratings","latency":"6.753ms","responseCode":200,"responseSize":48,"source":"reviews","user":"unknown"}
{"level":"warn","ts":"2017-09-21T04:33:31.263Z","instance":"newlog.logentry.istio-system","destination":"reviews","latency":"39.848ms","responseCode":200,"responseSize":379,"source":"productpage","user":"unknown"}
{"level":"warn","ts":"2017-09-21T04:33:31.239Z","instance":"newlog.logentry.istio-system","destination":"productpage","latency":"67.675ms","responseCode":200,"responseSize":5599,"source":"ingress.istio-system.svc.cluster.local","user":"unknown"}
{"level":"warn","ts":"2017-09-21T04:33:31.233Z","instance":"newlog.logentry.istio-system","destination":"ingress.istio-system.svc.cluster.local","latency":"74.47ms","responseCode":200,"responseSize":5599,"source":"unknown","user":"unknown"}
</pre></code>

## 如何理解监控的配置
---

在本章中，您添加了Istio配置，委托Mixer自动生成并报告网格内所有流量的新监控指标和新日志流。

增加管理三个Mixer功能的配置，分别是：

1. 使用Istio属性生成实例（在本例中为监控指标和日志）。

2. 创建能够处理已生成实例的配置项（配置Mixer适配器）。

3. 按照一套规则分发实例。

## 如何理解监控指标项的配置
---

监控指标项配置指定Mixer把监控指标值发送给Prometheus。它有三部分（或块）的配置组成：instance配置，handler配置和rule配置。

```kind: metric``` 这部分配置定义了一个名为```doublerequestcount```的新监控指标项schema去生成监控指标（或实例）。这个配置的例子告诉Mixer如何根据Envoy返回的属性（同样由Mixer自身生成）为任何给定的请求生成监控指标项。

```doublerequestcount.metric```中的配置指定Mixer为每个实例的值是2。由于Istio为每个请求生成一个实例，这意味着这个监控指标项的值等于收到请求总数的两倍。

每个```doublerequestcount.metric```实例指定一个```dimensions```。Dimensions提供了根据不同的需求和查询方式来分割，聚合和分析度量数据的方法。例如，在对应用程序行为进行故障排除时，可能只关注对某个目标服务的请求。

根据属性值和实际的值配置Dimensions来约定Mixer的行为。例如，对于```source dimensions```，新的配置请求从```source.service```属性取值。如果该属性值未设置，则该规则指定Mixer使用默认值```unknown```。对于```message```，设置值为```“twice the fun！”```将用于所有实例。

```kind: prometheus```：定义了一个名为```doublehandler```的配置项。```spec```配置prometheus适配器代码如何把接收的监控指标值转换为Prometheus后端可处理的格式。此配置指定了一个名为```double_request_count```的新Prometheus指标，有三个标签（与```doublerequestcount.metric```实例配置的相匹配）。

对于```kind：prometheus```的配置项，Mixer实例通过```instance_name```参数与Prometheus的监控指标项相匹配。 ```instance_name```值必须和Mixer实例的名称完全一致（例如：```doublerequestcount.metric.istio-system```）。

```kind: rule```配置定义了一个名为```doubleprom```的新规则。该规则指定Mixer将所有```doublerequestcount.metric```实例发送到```doublehandler.prometheus```处理。由于规则中没有```match```子句，并且由于该规则位于已配置默认命名空间（```istio-system```），因此网格中的所有请求都会执行该规则。

## 如何理解日志的配置
---

日志配置指定Mixer发送日志到stdout(标准输出)。它使用三部分（或块）配置：instance配置，handler配置和rule配置。

```kind: logentry```配置定义了一个名为```newlog```的生成日志（或实例）的模型。这个配置告诉Mixer如何根据Envoy返回的属性为请求生成日志实体。

```severity```参数用于指定任何要生成```logentry```的日志级别。在这个例子中，使用了```“warning”```。该值将通过```logentry```配置项映射到生成的日志级别。

```timestamp```参数为所有日志提供时间信息。在本例中，时间由```request.time```属性来设置，也就是由Envoy提供。

```variables```参数允许操作员配置每个```logentry```应包含哪些值。用表达式控制从Istio属性和设定值的映射关系来构成一个```logentry```。在此示例中，每个```logentry```实例都有一个名为```latency```的字段，对应了属性```response.duration```的值。如果```response.duration```没有值，则```latency```字段将被设置为0ms。

```kind: stdio```定义了一个名为```newhandler```的配置项。```spec```配置```stdio```适配器代码如何处理收到```logentry```实例。 ```severity_levels```参数控制```severity```字段和```logentry```值如何映射到已支持的日志级别。这里，```"warning"```的值被映射到```WARNING```级别的日志。 ```outputAsJson```参数指定适配器生成JSON格式的日志。

```kind: rule```定义了一个名为```newlogstdio```的新规则。该规则指定Mixer将```newlog.logentry```的所有实例发送到```newhandler.stdio```的配置项。由于```match```参数设置为true，所以对网格中的所有请求都执行该规则。

为所有请求都执行该规则是不需要明确配置```match: true```项。```spec```省略```match```参数项配置相当于设置```match：true```。这里是为了说明如何使用```match```表达式来配置控制规则。

## 清理配置项
---

- 删除监控配置：

<pre><code>istioctl delete -f new_telemetry.yaml
</pre></code>

- 如果您不打算研究后续任务，请参阅[BookInfo cleanup](https://istio.io/docs/guides/bookinfo.html#cleanup)的说明关闭应用程序。

## 进一步阅读
---

- 学习更多关于[Mixer](https://istio.io/docs/concepts/policy-and-control/mixer.html)和[Mixer Config](https://istio.io/docs/concepts/policy-and-control/mixer-config.html)。

- 查看完整的[Attribute Vocabulary](https://istio.io/docs/reference/config/mixer/attribute-vocabulary.html)。

- 阅读参考指南[Writing Config](https://istio.io/docs/reference/writing-config.html)。

- 请参阅[In-Depth Telemetry](https://istio.io/docs/guides/telemetry.html)指南。










