# 使用 Istio 复制流量，用于生产环境的测试

> HTTP 流量的路由规则之一

作者：Cristian Posta

原文地址：[https://istio.io/blog/2018/traffic-mirroring.html](https://istio.io/blog/2018/traffic-mirroring.html)


在测试环境或者其他的非生产环境下，要穷举所有案例的可能组合，是一个令人望而却步的高难度任务。有时候我们甚至会发现，在用例分类和组合方面进行大量投入后，仍然无法反应真实环境的实际情况。设想，如果我们具有使用生产环境的真实流量作为测试案例的能力的话，就能够更好的帮我们覆盖到在测试环境中未能涉及的盲点。

Istio 就提供了这样的能力。在 [Istio 0.5.0](https://istio.io/blog/2018/traffic-mirroring.html) 中，加入了流量镜像功能，这一功能对测试工作很有帮助。可以使用路由规则来实现流量的复制：

~~~yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: mirror-traffic-to-httbin-v2
spec:
  destination:
    name: httpbin
  precedence: 11
  route:
  - labels:
      version: v1
    weight: 100
  - labels: 
      version: v2
    weight: 0
  mirror:
    name: httpbin
    labels:
      version: v2
~~~

这里有几点需要注意的：

- 只有非关键流量被镜像到其他服务（主要指的是 Istio 相关的跟踪 Tag）。
- 目标服务对于镜像流量的响应会被忽略；流量的镜像过程是一种发过即弃的过程。
- 要创建一条权重为 0 的 Istio 路由规则，让 Envoy 集群进行初始化，[未来版本会改善这一问题](https://github.com/istio/istio/issues/3270)。

可以阅读 [Mirror Task](../../docs/tasks/traffic-management/mirroring.html)，并在[我的博客](https://blog.christianposta.com/microservices/traffic-shadowing-with-istio-reduce-the-risk-of-code-release/)上做一些这方面的深入了解。