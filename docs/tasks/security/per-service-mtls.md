# 启用每服务双向认证

在[安装指南](../../setup/install-kubernetes.md)中，我们展示了如何启用边车之间的[双向TLS认证](../../concepts/security/mutual-tls.md)。 这些设置将应用于网格中的所有边车。

在本教程中，您将学习：

* 注解Kubernetes服务以禁用（或启用）一个选定服务的双向TLS身份验证。
* 修改Istio网格配置以解除控制服务的相互TLS身份验证。

## 在开始前

* 了解Isio[双向TLS认证](../../concepts/security/mutual-tls.md)的概念。

* 熟悉[验证Istio双向TLS认证](./mutual-tls.md)。

* 参考[安装指南](../../setup/kubernetes/index.md)中的说明，通过双向TLS认证安装Istio。

* 使用Istio边车启动[httpbin Demo](https://github.com/istio/istio/tree/master/samples/httpbin)。 同样为了验证目的，启动两个[休眠](https://github.com/istio/istio/tree/master/samples/sleep)的实例：一个有边车，一个没有（在不同的命名空间）。 以下是帮助您启动这些服务的命令：


```
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)

kubectl create ns legacy && kubectl apply -f samples/sleep/sleep.yaml -n legacy
```

在这个初始设置中，我们期望默认命名空间中的休眠实例可以与httpbin服务进行通信，但是传统命名空间中的休眠实例不能，因为它没有边车来使能mTLS。


```
kubectl exec $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c sleep -- curl http://httpbin.default:8000/ip -s
```


```
{
  "origin": "127.0.0.1"
}
```


```
kubectl exec $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name} -n legacy) -n legacy -- curl http://httpbin.default:8000/ip -s
```


```
command terminated with exit code 56
```

## 禁用“httpbin”服务的相互TLS身份验证

如果我们希望在不更改网格验证设置下禁用httpbin（在端口8000上）的mTLS，我们可以通过将此注释添加到httpbin服务定义来实现。


```
annotations:
  auth.istio.io/8000: NONE
```

要进行快速验证，运行**kubectl edit svc httpbin**并添加上面的注解（或者编辑原始的httpbin.yaml文件并重新应用它）。 应用更改之后，由于mTLS已被删除sleep.legacy的请求现在应该是成功的。

注意：
注解可以用于相反的方向，例如：对单个服务启用mTLS，只需使用注解值**MUTUAL_TLS**而非**NONE**。 人们可以使用此选项在选定的服务上启用mTLS，而不是在整个网格中启用它。

注解也可以用于没有边车的（服务端）服务，以指示Istio在对该服务进行调用时不为客户端应用mTLS。 事实上如果一个系统有一些服务不由Istio管理（即没有边车），这是一个推荐的解决方案，以解决与这些服务的通信问题。

## 禁用控制服务的相互TLS身份验证
由于不能在Istio 0.3中注解控制服务，例如：API服务器。所以在网格配置中引入[mtls_excluded_services](https://github.com/istio/api/blob/master/mesh/v1alpha1/config.proto#L200:19)来指定不应使用mTLS的服务列表。 如果你的应用程序需要与任何控制服务通信，则应在其中列出其完全限定域名（FQDN）。

作为Demo的一部分，我们将展示该字段的影响。

默认情况下（0.3或更高版本），该列表包含**kubernetes.default.svc.cluster.local**（这是通用设置中的API服务器服务的名称）。 你可以通过运行以下命令来验证：


```
kubectl get configmap -n istio-system istio -o yaml | grep mtlsExcludedServices
```


```
mtlsExcludedServices: ["kubernetes.default.svc.cluster.local"]
```

然后预计请求kubernetes.default服务应该是可能的：


```
kubectl exec $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c sleep -- curl https://kubernetes.default:443/api/ -k -s
```


```
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "104.199.122.14"
    }
  ]
}
```

现在运行**kubectl edit configmap istio -n istio-system**并清除mtlsExcludedServices，在完成后重新启动pilot：


```
kubectl get pod $(kubectl get pod -l istio=pilot -n istio-system -o jsonpath={.items..metadata.name}) -n istio-system -o yaml | kubectl replace --force -f -
```

上面相同的测试请求现在失败了（失败码为35），这是由于休眠的边车再次开启了mTLS：


```
kubectl exec $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c sleep -- curl https://kubernetes.default:443/api/ -k -s
```


```
command terminated with exit code 35
```
