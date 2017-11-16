# 任务

任务展示如何用Istio系统实现一个单独特定的有目标的行为。

* [流量管理](traffic-management/index.md)

	* [配置请求路由](traffic-management/request-routing.md)。这个任务展示如何基于权重和HTTP header配置动态请求路由。

	* [故障注入](traffic-management/fault-injection.md)。这个任务展示如何注入延迟并测试应用的弹性。

	* [流量转移](traffic-management/traffic-shifting.md)。这个任务展示如何将服务的流量从旧版本转移到新版本。

	* [设置请求超时](traffic-management/request-timeouts.md)。这个任务展示如何使用Istio在Envoy中设置请求超时。

	* [Istio Ingress控制器](traffic-management/ingress.md)。描述如何在Kubernetes上配置Istio Ingress控制器。

	* [控制Egress流量](traffic-management/egress.md)。描述如何控制Isto来路由流量，从mesh中的服务到外部服务。

	* [FAQ](traffic-management/faq.md)。常见问题，已知局限和解决方案，还有这个话题上其他频繁被问的问题。

* [策略实施](policy-enforcement/index.md)

	* [开启限流](policy-enforcement/rate-limiting.md)。这个任务展示如何使用Istio来动态限制到服务的流量

	* [FAQ](policy-enforcement/faq.md)。常见问题，已知局限和解决方案，还有这个话题上其他频繁被问的问题。

* [Metrics，日志和跟踪](telemetry/index.md)

    * [分布式跟踪](telemetry/distributed-tracing.md)。如何配置代理，以便向Zipkin或Jaeger发送跟踪请求

    * [收集metrics和日志](telemetry/metrics-logs.md)。这个任务展示如何配置Istio来收集metrics和日志。

    * [收集TCP服务的Metrics](telemetry/tcp-metrics.md)。这个任务展示如何为TCP服务收集metrics和日志。

    * [从Prometheus中查询Metrics](telemetry/querying-metrics.md)。这个任务展示如何使用Prometheus查询metrics。

    * [使用Grafana可视化Metrics](telemetry/using-istio-dashboard.md)。这个任务展示如何安装并使用Istio的Dashboard来监控网格流量

    * [生成服务图](telemetry/servicegraph.md)。这个任务展示如何把Istio网格中的服务生成服务图

    * [FAQ](telemetry/faq.md)。常见问题，已知局限和解决方案，还有这个话题上其他频繁被问的问题。

* [安全](security/index.md)

    * [验证Istio双向TLS认证](security/mutual-tls.md)。这个任务展示如何验证并测试Istio的自动交互TLS认证。

    * [配置基础访问控制](security/basic-access-control.md)。这个任务展示如何使用Kubernetes标签控制对服务的访问。

    * [配置安全访问控制](security/secure-access-control.md)。这个任务展示如何使用服务账号来安全的控制对服务的访问。

    * [FAQ](security/faq.md)。常见问题，已知局限和解决方案，还有这个话题上其他频繁被问的问题。

