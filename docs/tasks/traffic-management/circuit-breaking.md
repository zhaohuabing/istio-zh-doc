# 断路器

本文中的这一任务展示了弹性应用的熔断能力。开发人员可以凭借这一能力，来限制因为故障、延迟高峰以及其他预计外的网络异常所造成的影响范围。下面将会延时如何针对连接、请求以及外部检测来进行断路器配置。

## 开始之前

- 遵循[安装指南](../../setup)设置Istio。
- 启动[httpbin](../../samples/httpbin)实例作为本任务的后端：

`kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)`

## 断路器

接下来设置一个场景来演示熔断能力。现在我们已经运行了`httpbin`服务。我们想要通过对istio[目标策略](../../reference/config/istio.routing.v1alpha1.md)的配置来设置一个熔断规则。在这之前首先要设置一个到这一目的的路由规则，这一规则会把所有访问`httpbin`的流量路由到`version=v1`去。

### 创建断路策略

1. 创建缺省路由规则，让所有目标为`httpbin`的流量都指向`v1`：

  istioctl create -f samples/httpbin/routerules/httpbin-v1.yaml

2. 创建[目标策略](../../reference/config/istio.routing.v1alpha1.md)来对`httpbin`的调用过程进行断路设置。

  ~~~yaml
  cat <<EOF | istioctl create -f -
  apiVersion: config.istio.io/v1beta1
  kind: DestinationPolicy
  metadata:
  name: httpbin-circuit-breaker
  spec:
  destination:
    name: httpbin
    labels:
      version: v1
  circuitBreaker:
    simpleCb:
      maxConnections: 1
      httpMaxPendingRequests: 1
      sleepWindow: 3m
      httpDetectionInterval: 1s
      httpMaxEjectionPercent: 100
      httpConsecutiveErrors: 1
      httpMaxRequestsPerConnection: 1
  EOF
  ~~~

1. 确认目标策略是否正确创建

  ~~~
  istioctl get destinationpolicy

  NAME                    KIND                                            NAMESPACE
  httpbin-circuit-breaker DestinationPolicy.v1alpha2.config.istio.io      istio-samples
  ~~~

## 设置客户端

接下来我们来定义调用`httpbin`服务的规则。要创建一个客户端，用于向我们的服务发送流量，从而进一步体验断路器功能。我们会使用一个简单的负载测试工具[fortio](https://github.com/istio/fortio)。使用这个客户端我们可以控制连接数量、并发数以及外发HTTP调用的延迟。这一步骤中，我们会设置一个客户端，注入istio sidecar，以此保证客户端的通信也在istio的监管之下：

~~~
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/sample-client/fortio-deploy.yaml)
~~~

这样我们就能够登入客户端 Pod，并使用fortio工具来调用`httpbin`，使用`-curl`开关可以指定我们只执行一次调用。

~~~
FORTIO_POD=$(kubectl get pod | grep fortio | awk '{ print $1 }')
kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -curl  http://httpbin:8000/get

HTTP/1.1 200 OK
server: envoy
date: Tue, 16 Jan 2018 23:47:00 GMT
content-type: application/json
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 445
x-envoy-upstream-service-time: 36

{
  "args": {},
  "headers": {
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "istio/fortio-0.6.2",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "824fbd828d809bf4",
    "X-B3-Traceid": "824fbd828d809bf4",
    "X-Ot-Span-Context": "824fbd828d809bf4;824fbd828d809bf4;0000000000000000",
    "X-Request-Id": "1ad2de20-806e-9622-949a-bd1d9735a3f4"
  },
  "origin": "127.0.0.1",
  "url": "http://httpbin:8000/get"
}
~~~

这里可以看到，调用成功了！接下来我们试试熔断功能。

## 测试断路器

在断路器设置中，我们指定`maxConnections: 1`以及`httpMaxPendingRequests: 1`。这样的设置下，如果我们并发超过一个连接和请求，istio-proxy就会断掉后续的请求和连接。我们试试两个并发链接（`-c 2`），发送20个请求（`-n 20`）：

~~~
kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get

Fortio 0.6.2 running at 0 queries per second, 2->2 procs, for 5s: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 2] for exactly 20 calls (10 per thread + 0)
23:51:10 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 106.474079ms : 20 calls. qps=187.84
Aggregated Function Time : count 20 avg 0.010215375 +/- 0.003604 min 0.005172024 max 0.019434859 sum 0.204307492
# range, mid point, percentile, count
>= 0.00517202 <= 0.006 , 0.00558601 , 5.00, 1
> 0.006 <= 0.007 , 0.0065 , 20.00, 3
> 0.007 <= 0.008 , 0.0075 , 30.00, 2
> 0.008 <= 0.009 , 0.0085 , 40.00, 2
> 0.009 <= 0.01 , 0.0095 , 60.00, 4
> 0.01 <= 0.011 , 0.0105 , 70.00, 2
> 0.011 <= 0.012 , 0.0115 , 75.00, 1
> 0.012 <= 0.014 , 0.013 , 90.00, 3
> 0.016 <= 0.018 , 0.017 , 95.00, 1
> 0.018 <= 0.0194349 , 0.0187174 , 100.00, 1
# target 50% 0.0095
# target 75% 0.012
# target 99% 0.0191479
# target 99.9% 0.0194062
Code 200 : 19 (95.0 %)
Code 503 : 1 (5.0 %)
Response Header Sizes : count 20 avg 218.85 +/- 50.21 min 0 max 231 sum 4377
Response Body/Total Sizes : count 20 avg 652.45 +/- 99.9 min 217 max 676 sum 13049
All done 20 calls (plus 0 warmup) 10.215 ms avg, 187.8 qps
~~~

会看到几乎所有请求都通过了。

~~~bash
Code 200 : 19 (95.0 %)
Code 503 : 1 (5.0 %)
~~~

这是因为istio-proxy允许一定的误差。我们把连接数提高到3：

~~~
kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -c 3 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get

Fortio 0.6.2 running at 0 queries per second, 2->2 procs, for 5s: http://httpbin:8000/get
Starting at max qps with 3 thread(s) [gomax 2] for exactly 30 calls (10 per thread + 0)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 71.05365ms : 30 calls. qps=422.22
Aggregated Function Time : count 30 avg 0.0053360199 +/- 0.004219 min 0.000487853 max 0.018906468 sum 0.160080597
# range, mid point, percentile, count
>= 0.000487853 <= 0.001 , 0.000743926 , 10.00, 3
> 0.001 <= 0.002 , 0.0015 , 30.00, 6
> 0.002 <= 0.003 , 0.0025 , 33.33, 1
> 0.003 <= 0.004 , 0.0035 , 40.00, 2
> 0.004 <= 0.005 , 0.0045 , 46.67, 2
> 0.005 <= 0.006 , 0.0055 , 60.00, 4
> 0.006 <= 0.007 , 0.0065 , 73.33, 4
> 0.007 <= 0.008 , 0.0075 , 80.00, 2
> 0.008 <= 0.009 , 0.0085 , 86.67, 2
> 0.009 <= 0.01 , 0.0095 , 93.33, 2
> 0.014 <= 0.016 , 0.015 , 96.67, 1
> 0.018 <= 0.0189065 , 0.0184532 , 100.00, 1
# target 50% 0.00525
# target 75% 0.00725
# target 99% 0.0186345
# target 99.9% 0.0188793
Code 200 : 19 (63.3 %)
Code 503 : 11 (36.7 %)
Response Header Sizes : count 30 avg 145.73333 +/- 110.9 min 0 max 231 sum 4372
Response Body/Total Sizes : count 30 avg 507.13333 +/- 220.8 min 217 max 676 sum 15214
All done 30 calls (plus 0 warmup) 5.336 ms avg, 422.2 qps
~~~

这样就看到，断路器开始发挥作用了：

~~~
Code 200 : 19 (63.3 %)
Code 503 : 11 (36.7 %)
~~~

只有63%的请求通过了，其他的都被断路器拒之门外，我们可以查询一下istio-proxy的统计数据：

~~~
kubectl exec -it $FORTIO_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats' | grep httpbin | grep pending

cluster.out.httpbin.springistio.svc.cluster.local|http|version=v1.upstream_rq_pending_active: 0
cluster.out.httpbin.springistio.svc.cluster.local|http|version=v1.upstream_rq_pending_failure_eject: 0
cluster.out.httpbin.springistio.svc.cluster.local|http|version=v1.upstream_rq_pending_overflow: 12
cluster.out.httpbin.springistio.svc.cluster.local|http|version=v1.upstream_rq_pending_total: 39
~~~

`upstream_rq_pending_overflow`的值为`12`，说明有`12`次调用被断路器拦截了。

## 清理

1. 清除规则。

  ~~~
  istioctl delete routerule httpbin-default-v1
  istioctl delete destinationpolicy httpbin-circuit-breaker
  ~~~

2. 关闭httpbin服务和客户端。

  ~~~
  kubectl delete deploy httpbin fortio-deploy
  kubectl delete svc httpbin
  ~~~

## 下一步

阅读[目标策略参考](../../reference/config/istio.routing.v1alpha1.md#DestinationPolicy)，其中会有更详尽的断路器相关内容。
