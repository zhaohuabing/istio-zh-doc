# 配置请求路由

此任务将演示如何根据权重和HTTP header配置动态请求路由。

## 前提条件

* 参照文档[安装指南](../../setup/index.md)中的步骤安装Istio。

* 部署[BookInfo](../../guides/bookinfo.md) 示例应用程序。

## 基于内容的路由

BookInfo示例部署了三个版本的reviews服务，因此需要设置一个缺省路由。否则当多次访问该应用程序时，会发现有时输出会包含带星级的评价内容，有时又没有。出现该现象的原因是当没有为应用显式指定缺省路由时，Istio会将请求随机路由到该服务的所有可用版本上。

> 请注意：本文档假设还没有设置任何路由规则。如果已经为示例应用程序创建了存在冲突的路由规则，则需要在下面的命令中使用 `replace` 关键字代替 `create`。

1. 将所有微服务的缺省版本设置为v1。

    `istioctl create -f samples/bookinfo/routing/route-rule-all-v1.yaml`

    > 请注意：在kubernetes中部署Istio时，可以在上面及其它所有命令行中用 `kubectl` 代替 `istioctl`。但是目前 `kubectl` 不提供对命令输入参数的验证。

    可以通过下面的命令来显示所有以创建的路由规则。

    `istioctl get virtualservices -o yaml`

    ~~~yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
    name: details
    ...
    spec:
    hosts:
    - details
    http:
    - route:
        - destination:
            name: details
            subset: v1
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
    name: productpage
    ...
    spec:
    gateways:
    - bookinfo-gateway
    - mesh
    hosts:
    - productpage
    http:
    - route:
        - destination:
            name: productpage
            subset: v1
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
    name: ratings
    ...
    spec:
    hosts:
    - ratings
    http:
    - route:
        - destination:
            name: ratings
            subset: v1
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
    name: reviews
    ...
    spec:
    hosts:
    - reviews
    http:
    - route:
        - destination:
            name: reviews
            subset: v1
    ---
    ~~~

    > 注意：上面的代码中定义的`subset`，可以用`istioctl get destinationrules -o yaml`指令来获取。

    规则向代理服务器的分发过程是异步的，所以需要等几秒的传播时间，然后才能尝试后面的内容。

2. 用浏览器打开BookInfo的URL（`http://$GATEWAY_URL/productpage`）。会看到Bookinfo应用的页面，注意`productpage`没有显示评价星星，这是因为`reviews:v1`不访问ratings服务。

   可以看到BookInfo应用程序的productpage页面。

   请注意`productpage`页面显示的内容中不包含带星的评价信息，这是为`reviews:v1`服务不会访问`ratings`服务。

3. 将来自特定用户的请求路由到`reviews:v2`。

   把来自测试用户"jason"的请求路由到`reviews:v2`，以启用`ratings`服务。

   `istioctl replace -f samples/bookinfo/routing/route-rule-reviews-test-v2.yaml`

   确认规则已创建：

   `istioctl get virtualservice reviews -o yaml`

   ~~~yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
    name: reviews
    ...
    spec:
    hosts:
    - reviews
    http:
    - match:
        - headers:
            cookie:
            regex: ^(.*?;)?(user=jason)(;.*)?$
        route:
        - destination:
            name: reviews
            subset: v2
    - route:
        - destination:
            name: reviews
            subset: v1
    ~~~

4. 以"jason"用户登录`productpage`页面。

   此时应该可以在每条评价后面看到星级信息。请注意如果以别的用户登录，还是只能看到`reviews:v1`版本服务呈现出的内容，即不包含星级信息的内容。

## 理解原理

在这个任务中，我们首先使用Istio将100%的请求流量都路由到了BookInfo服务的v1版本。
然后再设置了一条路由规则，该路由规则基于请求的header（例如一个用户cookie）选择性地将特定的流量路由到了reviews服务的v2版本。

一旦reviews服务的v2版本经过测试后满足要求，我们就可以使用Istio将来自所有用户的流量一次性或者渐进地路由到v2版本。我们将在另一个单独的任务中对此进行尝试。

## 清理

* 删除路由规则

  ```bash
  istioctl delete -f samples/bookinfo/kube/route-rule-all-v1.yaml
  ```

* 如果不打算尝试后面的任务，请参照 [BookInfo cleanup](../../guides/bookinfo.md#cleanup) 中的步骤关闭应用程序。

## 下一步

* 更多的内容请参见[](../../concepts/traffic-management/rules-configuration.md/)。