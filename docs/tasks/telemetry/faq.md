## 如何查看Mixer的所有配置？

对于*instances*、 *handlers* 及 *rules*的配置都是作为Kubernetes [自定义资源](https://kubernetes.io/docs/concepts/api-extension/custom-resources/)存储的，
因此配置信息可以通过`kubectl`查询Kubernetes [API服务器](https://kubernetes.io/docs/admin/kube-apiserver/)资源来访问。

### 规则

要查看所有规则列表，执行下列命令：

```
kubectl get rules --all-namespaces
```

输出类似于：

```
NAMESPACE      NAME        KIND
default        mongoprom   rule.v1alpha2.config.istio.io
istio-system   promhttp    rule.v1alpha2.config.istio.io
istio-system   promtcp     rule.v1alpha2.config.istio.io
istio-system   stdio       rule.v1alpha2.config.istio.io
```

要查看单个规则配置，执行下列命令：

```
kubectl -n <namespace> get rules <name> -o yaml
```

### Handlers

Handlers的定义，是基于Kubernetes针对适配器的[自定义资源定义](https://kubernetes.io/docs/concepts/api-extension/custom-resources/#customresourcedefinitions)的。

首先，标识适配器类别列表：

```
kubectl get crd -listio=mixer-adapter
```

输出类似于：

```
NAME                           KIND
deniers.config.istio.io        CustomResourceDefinition.v1beta1.apiextensions.k8s.io
listcheckers.config.istio.io   CustomResourceDefinition.v1beta1.apiextensions.k8s.io
memquotas.config.istio.io      CustomResourceDefinition.v1beta1.apiextensions.k8s.io
noops.config.istio.io          CustomResourceDefinition.v1beta1.apiextensions.k8s.io
prometheuses.config.istio.io   CustomResourceDefinition.v1beta1.apiextensions.k8s.io
stackdrivers.config.istio.io   CustomResourceDefinition.v1beta1.apiextensions.k8s.io
statsds.config.istio.io        CustomResourceDefinition.v1beta1.apiextensions.k8s.io
stdios.config.istio.io         CustomResourceDefinition.v1beta1.apiextensions.k8s.io
svcctrls.config.istio.io       CustomResourceDefinition.v1beta1.apiextensions.k8s.io
```

然后，对于列表中的每一个适配器类别，执行下列命令：

```
kubectl get <adapter kind name> --all-namespaces
```

标准输出类似于：

```
NAMESPACE      NAME      KIND
istio-system   handler   stdio.v1alpha2.config.istio.io
```

要查看单个handler的配置，执行下列命令：

```
kubectl -n <namespace> get <adapter kind name> <name> -o yaml
```


### 实例

实例是根据Kubernetes针对实例的[自定义资源定义](https://kubernetes.io/docs/concepts/api-extension/custom-resources/#customresourcedefinitions)来定义的。

首先，标识实例类型列表：

```
kubectl get crd -listio=mixer-instance
```

输出类似于：

```
NAME                             KIND
checknothings.config.istio.io    CustomResourceDefinition.v1beta1.apiextensions.k8s.io
listentries.config.istio.io      CustomResourceDefinition.v1beta1.apiextensions.k8s.io
logentries.config.istio.io       CustomResourceDefinition.v1beta1.apiextensions.k8s.io
metrics.config.istio.io          CustomResourceDefinition.v1beta1.apiextensions.k8s.io
quotas.config.istio.io           CustomResourceDefinition.v1beta1.apiextensions.k8s.io
reportnothings.config.istio.io   CustomResourceDefinition.v1beta1.apiextensions.k8s.io
```

然后，对于列表中的每一个实例类型，执行下列命令：

```
kubectl get <instance kind name> --all-namespaces
```

针对`metrics`的输出类似于：

```
NAMESPACE      NAME                 KIND
default        mongoreceivedbytes   metric.v1alpha2.config.istio.io
default        mongosentbytes       metric.v1alpha2.config.istio.io
istio-system   requestcount         metric.v1alpha2.config.istio.io
istio-system   requestduration      metric.v1alpha2.config.istio.io
istio-system   requestsize          metric.v1alpha2.config.istio.io
istio-system   responsesize         metric.v1alpha2.config.istio.io
istio-system   tcpbytereceived      metric.v1alpha2.config.istio.io
istio-system   tcpbytesent          metric.v1alpha2.config.istio.io
```

要查看单个实例的配置，执行下列命令：

```
kubectl -n <namespace> get <instance kind name> <name> -o yaml
```

## Mixer支持的全部属性表达式是什么？

请参考 [表达式语言参考]({{home}}/docs/reference/config/mixer/expression-language.html)来了解支持的全部属性表达式。

## Mixer提供自监控吗？

Mixer提供的自监控形式包括调试端口、Prometheus指标、访问和应用日志以及为所有请求生成的跟踪数据。

### Mixer监控

Mixer暴露了一个监控端口（默认是 `9093`），并提供了一些很有用的路径，用于Mixer性能观察和审计功能：

- `/metrics` 提供Mixer进程的Prometheus指标、API调用相关的gRPC指标以及适配器调度指标。
- `/debug/pprof` 提供[pprof格式](https://golang.org/pkg/net/http/pprof/) 的profiling数据端口。
- `/debug/vars` 提供一个以JSON格式暴漏服务器指标的端口。

### Mixer日志

Mixer日志可通过 `kubectl logs` 命令来访问，如下：

```
kubectl -n istio-system logs $(kubectl -n istio-system get pods -listio=mixer -o jsonpath='{.items[0].metadata.name}') mixer
```

### Mixer跟踪

Mixer跟踪的生成方式是通过命令行选项 `traceOutput` 来控制的。如果该选项值设置为 `STDOUT` 或 `STDERR` ，跟踪数据就会直接输出到对应位置；
如果设置为一个URL，Mixer就会把Zipkin格式化的数据post到对应地址（比如 `http://zipkin:9411/api/v1/spans`）。

在0.2版本中，Mixer仅支持Zipkin跟踪。

## 如何写一个自定义的Mixer适配器？

要实现一个新的Mixer适配器，请参考 [适配器开发者指南](https://github.com/istio/istio/blob/master/mixer/doc/adapters.md)。
