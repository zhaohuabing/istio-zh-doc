# Istio Ingress控制器

此任务将演示如何通过配置Istio将服务发布到service mesh集群外部。在Kubernetes环境中，[Kubernetes Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/) 允许用户指定某个服务是否要公开到集群外部。然而，Ingress Resource规范非常精简，只允许用户设置主机，路径，以及后端服务。下面是Istio ingress已知的局限：

1. Istio支持不使用annotation的标准Kubernetes Ingress规范。不支持Ingress资源规范中的`ingress.kubernetes.io` annotation。`kubernetes.io/ingress.class: istio`之外的任何annotation将被忽略。
2. 不支持路径中的正则表达式
3. Ingress中不支持故障注入

## 前提条件

* 参照文档[安装指南](../../setup/index.md)中的步骤安装Istio。

* 确保当前的目录是`istio`目录。

* 启动 [httpbin](https://github.com/istio/istio/tree/master/samples/httpbin) 示例, 我们会把这个服务作为目标服务发布到外部。

    如果你安装了[Istio-Initializer](../../../setup/kubernetes/sidecar-injection.md#automatic-sidecar-injection), 请执行：

    ```console
    kubectl apply -f samples/httpbin/httpbin.yaml
    ```

    如果没有Istio-Initializer, 请执行：

    ```console
    kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
    ```

## 配置ingress (HTTP)

1. 为httpbin服务创建一个基本的Ingress Resource

	```bash
    cat <<EOF | kubectl create -f -
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: simple-ingress
      annotations:
        kubernetes.io/ingress.class: istio
    spec:
      rules:
      - http:
          paths:
          - path: /status/.*
            backend:
              serviceName: httpbin
              servicePort: 8000
          - path: /delay/.*
            backend:
              serviceName: httpbin
              servicePort: 8000
    EOF
	```

    `/.*`是特殊的Istio表示方法，用于表示前缀匹配，特别是(prefix: /)形式的[规则匹配配置]（前缀：/）。

## 验证 ingress

1. 确定ingress URL：

   * 如果你的集群运行的环境支持外部的负载均衡器，请使用ingress的外部地址：

     ```bash
     kubectl get ingress simple-ingress -o wide
     ```

     ```bash
     NAME             HOSTS     ADDRESS                 PORTS     AGE
     simple-ingress   *         130.211.10.121          80        1d
     ```

     ```bash
     export INGRESS_HOST=130.211.10.121
     ```

   * 如果不支持负载均衡器，使用ingress控制器的pod的hostIP：

     ```bash
     kubectl get po -l istio=ingress -o jsonpath='{.items[0].status.hostIP}'
     ```

     ```bash
     169.47.243.100
     ```

     同时也找到istio-ingress服务的80端口的nodePort映射端口:

     ```bash
     kubectl -n istio-system get svc istio-ingress
     ```

     ```bash
    NAME            CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    istio-ingress   10.10.10.155   <pending>     80:31486/TCP,443:32254/TCP   32m
     ```

     ```bash
     export INGRESS_HOST=169.47.243.100:31486
     ```

2. 使用*curl*访问httpbin服务：

   ```bash
   curl -I http://$INGRESS_HOST/status/200
   ```

	```bash
    HTTP/1.1 200 OK
    server: envoy
    date: Mon, 29 Jan 2018 04:45:49 GMT
    content-type: text/html; charset=utf-8
    access-control-allow-origin: *
    access-control-allow-credentials: true
    content-length: 0
    x-envoy-upstream-service-time: 48
	```

3. 如果访问其他的没有明确公开的URL，应该收到HTTP404错误

    ```console
    curl -I http://$INGRESS_HOST/headers
    ```

    ```console
    HTTP/1.1 404 Not Found
    date: Mon, 29 Jan 2018 04:45:49 GMT
    server: envoy
    content-length: 0
    ```

## 配置安全ingress(HTTPS)

1. 为ingress生成证书和密钥

   使用[OpenSSL](https://www.openssl.org/)创建测试私钥和证书

   ```console
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=foo.bar.com"
   ```

2. 创建secret

	使用`kubectl`在命名空间`istio-system`创建`istio-ingress-certs`secret。

	> 注意：secret在命名空间`istio-system`中必须被称为`istio-ingress-certs`，因为它要被加载到Isito Ingress。

   ```console
   kubectl create -n istio-system secret tls istio-ingress-certs --key /tmp/tls.key --cert /tmp/tls.crt
   ```

3.为httpbin服务创建Ingress Resource

    ```bash
    cat <<EOF | kubectl create -f -
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: secure-ingress
      annotations:
        kubernetes.io/ingress.class: istio
    spec:
      tls:
        - secretName: istio-ingress-certs # currently ignored
      rules:
      - http:
          paths:
          - path: /status/.*
            backend:
              serviceName: httpbin
              servicePort: 8000
          - path: /delay/.*
            backend:
              serviceName: httpbin
              servicePort: 8000
    EOF
    ```

   > 注意: Envoy当前只允许一个TLS ingress密钥，因为[SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) 尚未支持。也就是说看，ingress 中的 secretName 字段并没有被使用。

## 再次验证ingress

> 译者注：注意这里的命令和输出与前面的稍有不同。

1. 确定ingress URL：

   * 如果你的集群运行的环境支持外部的负载均衡器，请使用ingress的外部地址：

     ```bash
     kubectl get ingress secure-ingress -o wide
     ```

    ```bash
    NAME             HOSTS     ADDRESS                 PORTS     AGE
    secure-ingress   *         130.211.10.121          80        1d
    ```

     ```bash
     export INGRESS_HOST=130.211.10.121
     ```

   * 如果不支持负载均衡器，使用ingress控制器的pod的hostIP：

     ```bash
     kubectl -n istio-system get po -l istio=ingress -o jsonpath='{.items[0].status.hostIP}'
     ```

     ```bash
     169.47.243.100
     ```

     同时也找到istio-ingress服务的80端口的nodePort映射端口:

     ```bash
     kubectl -n istio-system get svc istio-ingress
     ```

     ```bash
    NAME            CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    istio-ingress   10.10.10.155   <pending>     80:31486/TCP,443:32254/TCP   32m
     ```

     ```bash
     export INGRESS_HOST=169.47.243.100:31486
     ```

2. 使用*curl*访问httpbin服务：

   ```bash
   curl -I http://$INGRESS_HOST/status/200
   ```

	```bash
    HTTP/1.1 200 OK
    server: envoy
    date: Mon, 29 Jan 2018 04:45:49 GMT
    content-type: text/html; charset=utf-8
    access-control-allow-origin: *
    access-control-allow-credentials: true
    content-length: 0
    x-envoy-upstream-service-time: 96
	```

3. 如果访问其他的没有明确公开的URL，应该收到HTTP404错误

    ```console
    curl -I http://$INGRESS_HOST/headers
    ```

    ```console
    HTTP/1.1 404 Not Found
    date: Mon, 29 Jan 2018 04:45:49 GMT
    server: envoy
    content-length: 0
    ```

## 和ingress一起使用istio路由规则

Istio的路由规则可以在路由请求到后端服务时获取更大的控制度。例如，下列路由规则为到httpbin服务上的`/delay`URL的所有请求设置4秒的超时时间。

```bash
cat <<EOF | istioctl create -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: status-route
spec:
  destination:
    name: httpbin
  match:
    # Optionally limit this rule to istio ingress pods only
    source:
      name: istio-ingress
      labels:
        istio: ingress
    request:
      headers:
        uri:
          prefix: /delay/ #must match the path specified in ingress spec
              # if using prefix paths (/delay/.*), omit the .*.
              # if using exact match, use exact: /status
  route:
  - weight: 100
  httpReqTimeout:
    simpleTimeout:
      timeout: 4s
EOF
```

如果用URL`http://$INGRESS_HOST/delay/10`来发起对ingress的调用，会发现调用在4秒之后返回，而不是期待的10秒延迟。

可以使用路由规则的其他特性如重定向，重写，路由到多个版本，基于HTTP header的正则表达式匹配，websocket升级，超时，重试，等。更多详情请参考[路由规则](https://istio.io/docs/reference/config/istio.routing.v1alpha1.html)。

> 注意1： 在Ingress中故障注入不可用
> 注意2： 当请求匹配路由规则时，使用完全相同的路径或者前缀，就像在Ingress规范中使用的那样。

## 理解Ingress

Ingress为外部流量提供网关来进入Istio服务网格，并使得Istio的流量管理和策略的特性对edge服务可用。

Ingress规范中的servicePort字段可以是一个端口数字（整型）或者一个名称。为了正确工作，端口名称必须遵循Istio端口命名约定(例如，`grpc-*`, `http2-*`, `http-*`, 等)。使用的名称必须匹配后端服务声明中的端口名称。

在前面的步骤中，我们在Isito服务网格中创建服务，并展示了如何暴露服务的HTTP和HTTPS终端到外部流量。也展示了如何使用Istio路由规则来控制ingress流量。

## 清理

1. 删除secret和Ingress Resource定义

	```bash
    kubectl delete ingress simple-ingress secure-ingress
	kubectl delete -n istio-system secret istio-ingress-certs
	```

2. 关闭[httpbin](https://github.com/istio/istio/tree/master/samples/httpbin)服务

	```bash
    kubectl delete -f samples/httpbin/httpbin.yaml
	```

## 进阶阅读

* 进一步了解和学习[Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/).
* 进一步了解和学习[routing rules](../../concepts/traffic-management/rules-configuration.md).
