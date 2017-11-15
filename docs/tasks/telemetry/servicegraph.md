# 生成服务图示

这个任务展示如何设置和使用Service Graph插件来查看服务的调用关系。

## 开始之前

- 在集群上[安装Istio](../../setup/)并部署应用。

- 安装Prometheus插件。

	`kubectl apply -f install/kubernetes/addons/prometheus.yaml`

	Service Graph必须有Prometheus插件才能工作。

## 生成一个服务图示

1. 要查看Service Mesh的图形展示，首先要安装Servicegraph插件。

	在Kubernetes环境中的安装，运行下面的命令：

    ```bash
    kubectl apply -f install/kubernetes/addons/servicegraph.yaml
    ```

1. 检查集群中服务的运行状态。

	在Kubernetes环境中，运行下面的命令：

    ```bash
    kubectl -n istio-system get svc servicegraph
    ```

	输出内容大致如下所示：

    ```
    NAME           CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    servicegraph   10.59.253.165   <none>        8088/TCP   30s
    ```

1. 向服务发送流量。

	如果是BookInfo示例，使用浏览器打开`http://$GATEWAY_URL/productpage`，或者使用控制台命令：

   ```bash
   curl http://$GATEWAY_URL/productpage
   ```

	刷新几次浏览器，或者重复执行几次命令，会产生一定数量的流量。

	注意：`$GATEWAY_URL`的来由可参看[BookInfo指南](../../guides/bookinfo.md)。

1. 打开Servicegraph UI。

	如果是Kubernetes环境，执行下面的命令：

    ```bash
    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8088:8088 &   
    ```

	使用浏览器访问[http://localhost:8088/dotviz](http://localhost:8088/dotviz)，浏览器中会展示类似的图形：

	![Example Servicegraph](./img/servicegraph-example.png)

## 关于Servicegraph插件

Servicegraph服务是一个示例服务，他提供了一个生成和展现Service Mesh中服务关系的功能，包含如下几个服务端点：

- `/graph`：提供了servicegraph的JSON序列化展示

- `/dotgraph`：提供了servicegraph的Dot序列化展示

- `dotviz`：提供了servicegraph的可视化展示

所有的端点都可以使用一个可选参数`time_horizon`，这个参数控制图形生成的时间跨度。

另外一个可选参数就是`filter_empty=true`，在`time_horizon`所限定的时间段内，这一参数可以限制只显示流量大于零的node和edge。

## 清理

* 在Kubernetes环境中，执行下面的命令可以移除ServiceGraph插件：

    ```bash
    kubectl delete -f install/kubernetes/addons/servicegraph.yaml
    ```

* 如果不想继续后续内容，参考[BookInfo清理](../../guides/bookinfo.md#清理)的介绍来关闭应用。
