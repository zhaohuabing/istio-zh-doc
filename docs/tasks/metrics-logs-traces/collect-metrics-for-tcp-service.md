# 收集TCP服务的监控信息

本章介绍如何配置istio去自动收集mesh中的TCP服务指标。本章结尾部分，一个新的监控指标可以用mesh去调用TCP服务。

[BookInfo](https://istio.io/docs/guides/bookinfo.html)示例中的应用是介绍本章内容的一个例子。

## 环境准备
---

- 集群中[安装istio](https://istio.io/docs/setup/)并部署一个应用。

- 本章假定BookInfo中的示例应用被部署在```default```命名空间中。如果部署在不同的命名空间，你需要修改例子中的配置和命令。

- 安装Prometheus插件，Prometheus用来验证任务是否成功执行。

<pre><code>kubectl apply -f install/kubernetes/addons/prometheus.yaml
</pre></code>

详细内容查看[Prometheus](https://prometheus.io)。

## 收集新的监控数据
---

1. 创建一个YAML文件，配置新的监控指标项使Istio可以自动收集监控数据。

把以下内容保持为```tcp_telemetry.yaml```文件：

<pre><code># Configuration for a metric measuring bytes sent from a server
# to a client
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: mongosentbytes
  namespace: default
spec:
  value: connection.sent.bytes | 0 # uses a TCP-specific attribute
  dimensions:
    source_service: source.service | "unknown"
    source_version: source.labels["version"] | "unknown"
    destination_version: destination.labels["version"] | "unknown"
  monitoredResourceType: '"UNSPECIFIED"'
---
# Configuration for a metric measuring bytes sent from a client
# to a server
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: mongoreceivedbytes
  namespace: default
spec:
  value: connection.received.bytes | 0 # uses a TCP-specific attribute
  dimensions:
    source_service: source.service | "unknown"
    source_version: source.labels["version"] | "unknown"
    destination_version: destination.labels["version"] | "unknown"
  monitoredResourceType: '"UNSPECIFIED"'
---
# Configuration for a Prometheus handler
apiVersion: "config.istio.io/v1alpha2"
kind: prometheus
metadata:
  name: mongohandler
  namespace: default
spec:
  metrics:
  - name: mongo_sent_bytes # Prometheus metric name
    instance_name: mongosentbytes.metric.default # Mixer instance name (fully-qualified)
    kind: COUNTER
    label_names:
    - source_service
    - source_version
    - destination_version
  - name: mongo_received_bytes # Prometheus metric name
    instance_name: mongoreceivedbytes.metric.default # Mixer instance name (fully-qualified)
    kind: COUNTER
    label_names:
    - source_service
    - source_version
    - destination_version
---
# Rule to send metric instances to a Prometheus handler
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: mongoprom
  namespace: default
spec:
  match: context.protocol == "tcp"
         && destination.service == "mongodb.default.svc.cluster.local"
  actions:
  - handler: mongohandler.prometheus
    instances:
    - mongoreceivedbytes.metric
    - mongosentbytes.metric
</pre></code>

2. 推送新的配置。

<pre><code>istioctl create -f tcp_telemetry.yaml</pre></code>

输出类似如下内容：

<pre><code>Created config metric/default/mongosentbytes at revision 3852843
Created config metric/default/mongoreceivedbytes at revision 3852844
Created config prometheus/default/mongohandler at revision 3852845
Created config rule/default/mongoprom at revision 3852846</pre></code>

3. 设置并使用MongoDB

a) 安装```v2```版本的```ratings```服务。

如果您启用了自动注入sidecar功能的集群，只需使用```kubectl```部署服务即可：

<pre><code>kubectl apply -f samples/bookinfo/kube/bookinfo-ratings-v2.yaml</pre></code>

如果您正在使用手动注入sidecar，请改用以下命令：

<pre><code>kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/kube/bookinfo-ratings-v2.yaml)</pre></code>

输出如下内容：

<pre><code>deployment "ratings-v2" configured</pre></code>

b) 安装```mongodb```服务：

如果您启用了自动注入sidecar功能的集群，只需使用```kubectl```部署服务即可：

<pre><code>kubectl apply -f samples/bookinfo/kube/bookinfo-db.yaml</pre></code>

如果您正在使用手动注入sidecar，请改用以下命令：

<pre><code>kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/kube/bookinfo-db.yaml)</pre></code>

输出如下内容：

<pre><code>service "mongodb" configured
deployment "mongodb-v1" configured</pre></code>

c) 添加路由规则将流量发送到```ratings```服务的```v2```版本：

<pre><code>istioctl create -f samples/bookinfo/kube/route-rule-ratings-db.yaml</pre></code>

输出如下内容：

<pre><code>Created config route-rule//ratings-test-v2 at revision 7216403
Created config route-rule//reviews-test-ratings-v2 at revision 7216404</pre></code>

4. 把流量发送到示例应用。

参照BookInfo中的例子，使用浏览器访问```http://$GATEWAY_URL/productpage```或者使用如下命令：

<pre><code>curl http://$GATEWAY_URL/productpage</pre></code>

5. 验证新的监控信息是否产生并被收集

在Kubernetes环境中，使用如下命令为Prometheus设置端口转发：

<pre><code>kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &</pre></code>

通过[Prometheus UI](http://localhost:9090/graph#[{"range_input":"1h","expr":"mongo_received_bytes","tab":1}])查看新的监控指标项采集的数据。

使用提供的链接打开Prometheus UI并查询```mongo_received_bytes```监控指标项收集的数据。在Console tab页展示类似以下信息：

<pre><code>mongo_received_bytes{destination_version="v1",instance="istio-mixer.istio-system:42422",job="istio-mesh",source_service="ratings.default.svc.cluster.local",source_version="v2"}	2317</pre></code>

注意：Istio还收集MongoDB特定协议的统计信息。 例如，从```ratings```服务发送的所有OP_QUERY信息的值，用以下监控指标项收集:```envoy_mongo_mongo_collection_ratings_query_total_counter```（[单击此处执行查询](http://localhost:9090/graph#[{"range_input":"1h","expr":"envoy_mongo_mongo_collection_ratings_query_total_counter","tab":1}])）。

## 了解TCP信息收集配置
---

在此任务中，您添加了Istio配置去指定Mixer自动生成mesh内TCP服务所有流量的新监控指标项并收集。

与[收集指标和日志](https://istio.io/docs/tasks/telemetry/metrics-logs.html)类似，新配置由instances，handler和rule组成。 请参阅该任务了解获取监控指标数据收集的完整说明。

TCP服务的监控数据收集配置仅在instances中有有限几个属性有所不同。

### TCP属性

几个特定的TCP属性可以在Istio中启用TCP策略和控制。 这些属性由服务器端的Envoy代理生成，并在连接建立和关闭时转发给Mixer。 另外，上下文属性提供区分```http```和```tcp```协议的能力。

<img src="../img/istio-tcp-attribute-flow.svg"></img>

## 清理
---

- 删除新的配置：

<pre><code>istioctl delete -f tcp_telemetry.yaml</pre></code>

- 删除端口映射：

<pre><code>killall kubectl</pre></code>

- 如果您不打算探索任何后续任务，请参阅[BookInfo cleanup](https://istio.io/docs/guides/bookinfo.html#cleanup)的说明关闭应用程序。

## 进一步阅读
---

- 学习更多关于[Mixer](https://istio.io/docs/concepts/policy-and-control/mixer.html)和[Mixer Config](https://istio.io/docs/concepts/policy-and-control/mixer-config.html)。
- 查看全部[Attribute Vocabulary](https://istio.io/docs/reference/config/mixer/attribute-vocabulary.html)。
- 阅读[Writing Config](https://istio.io/docs/reference/writing-config.html)的参考指南。
- 参考[In-Depth Telemetry]（https://istio.io/docs/guides/telemetry.html）指南。
- 学习更多关于[Querying Istio Metrics](https://istio.io/docs/tasks/telemetry/querying-metrics.html)。
- 学习更多关于[MongoDB-specific statistics generated by Envoy](https://envoyproxy.github.io/envoy/configuration/network_filters/mongo_proxy_filter.html#statistics)




