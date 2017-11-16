# 从Prometheus中查询Metrics

此任务向您展示如何使用Prometheus查询Istio监控信息。

作为此任务的一部分，您将安装Prometheus Istio插件，并使用Web界面查询监控信息。

本章使用[BookInfo](../../guides/bookinfo.md)示例应用程序作为例子。

## 开始之前

* [安装Istio](../docs/setup/)，部署应用。

## 查询Istio metrics

1. 要查询Mixer提供的监控信息，先要安装Prometheus插件。

   在Kubernetes中，执行以下命令：

   ```bash
   kubectl apply -f install/kubernetes/addons/prometheus.yaml
   ```

1. 验证服务是否已正常运行。

   在Kubernetes中，执行以下命令：

   ```bash
   kubectl -n istio-system get svc prometheus
   ```

   会看到类似以下的输出：

   ```
   NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
   prometheus   10.59.241.54   <none>        9090/TCP   2m
   ```

1. 访问mesh。

   在浏览器中访问`http://$GATEWAY_URL/productpage`或者用以下命令请求：

   ```bash
   curl http://$GATEWAY_URL/productpage
   ```

   **提示：** `$GATEWAY_URL`的值请参考[BookInfo](../../guides/bookinfo.md)。

1. 打开Prometheus UI。

   在Kubernetes中，执行以下命令：

   ```bash
   kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &
   ```

   浏览器中访问：[http://localhost:9090/graph]（http://localhost:9090/graph）。

1. Prometheus中执行查询。

   在网页的"Expression"输入框中输入：`request_count`，然后点击**Execute**按钮。

   会有类似下面的结果输出：

   <figure><img style="max-width:100%" src="./img/prometheus_query_result.png" alt="Prometheus Query Result" title="Prometheus Query Result" />
    <figcaption>Prometheus Query Result</figcaption></figure>

   再试试其他查询：

   - 查询调用`productpage`服务的总数：

   ```
   request_count{destination_service="productpage.default.svc.cluster.local"}
   ```

   - 查询请求`reviews`的`v3`版本服务的总次数：

   ```
   request_count{destination_service="reviews.default.svc.cluster.local", destination_version="v3"}
   ```

   查询得到的结果就是当前请求`reviews`的`v3`版本服务的总数。

   - 所有`productpage`服务在过去5分钟内的请求成功率：

   ```
   rate(request_count{destination_service=~"productpage.*", response_code="200"}[5m])
   ```

### 关于Prometheus插件

Mixer内置了一个[Prometheus](https://prometheus.io)适配器，并开放了一个服务用于收集监控信息。 Prometheus插件是一个Prometheus服务器，它预置了数据抓取配置，可以从Mixer收集metrics。 它提供了一个持久存储和查询Istio metrics的机制。

配置好的Prometheus插件有三部分：

1. *istio-mesh* （`istio-mixer.istio-system：42422)`）：所有Mixer产生的mesh metrics。

2. *Mixer* （`istio-mixer.istio-system：9093`）：所有特定的Mixer metrics。 用于监控Mixer自身。

3. *envoy* （`istio-mixer.istio-system：9102`）：由envoy生成原始统计信息（并从statsd翻译成Prometheus）。

有关查询Prometheus的更多信息，请阅读他们的[querying docs](https://prometheus.io/docs/prometheus/latest/querying/basics/)。

## 清除

* 在Kubernetes中，执行如下命令删除Prometheus：

   ```bash
   kubectl delete -f install/kubernetes/addons/prometheus.yaml
   ```

* 移除任何可能在运行的`kubectl port-forward`进程：

   ```bash
   killall kubectl
   ```

* 如果您不打算继续后续的章节，请参阅[BookInfo cleanup](../../guides/bookinfo.md#cleanup)说明去关闭应用程序。

## 进一步阅读

* 请参阅[深入遥测](../../guides/telemetry.md)指南。













