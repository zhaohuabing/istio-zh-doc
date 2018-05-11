# 适配器

这是为 Mixer 适配器自动生成的文档。

- [Circonus](https://istio.io/docs/reference/config/adapters/circonus.html)：Adapter for circonus.com's monitoring solution.
- [Datadog](https://istio.io/docs/reference/config/adapters/datadog.html)：Adapter to deliver metrics to a dogstatsd agent for delivery to DataDog
- [Denier](https://istio.io/docs/reference/config/adapters/denier.html)：Adapter that always returns a precondition denial.
- [Fluentd](https://istio.io/docs/reference/config/adapters/fluentd.html)：Adapter that delivers logs to a fluentd daemon.
- [Kubernetes Env](https://istio.io/docs/reference/config/adapters/kubernetesenv.html)：Adapter that extracts information from a Kubernetes environment.
- [List](https://istio.io/docs/reference/config/adapters/list.html)：Adapter that performs whitelist or blacklist checks
- [Memory quota](https://istio.io/docs/reference/config/adapters/memquota.html)：Adapter for a simple in-memory quota management system.
- [OPA](https://istio.io/docs/reference/config/adapters/opa.html)：Adapter that implements an Open Policy Agent engine
- [Prometheus](https://istio.io/docs/reference/config/adapters/prometheus.html)：Adapter that exposes Istio metrics for ingestion by a Prometheus harvester.
- [RBAC](https://istio.io/docs/reference/config/adapters/rbac.html)：Adapter that exposes Istio's Role-Based Access Control model.
- [Redis Quota](https://istio.io/docs/reference/config/adapters/redisquota.html)：Adapter for a Redis-based quota management system.
- [Service Control](https://istio.io/docs/reference/config/adapters/servicecontrol.html)：Adapter that delivers logs and metrics to Google Service Control
- [SolarWinds](https://istio.io/docs/reference/config/adapters/solarwinds.html)：Adapter to deliver logs and metrics to Papertrail and AppOptics backends
- [Stackdriver](https://istio.io/docs/reference/config/adapters/stackdriver.html)：Adapter to deliver logs and metrics to Stackdriver
- [StatsD](https://istio.io/docs/reference/config/adapters/statsd.html)：Adapter to deliver metrics to a StatsD backend
- [Stdio](https://istio.io/docs/reference/config/adapters/stdio.html)：Adapter for outputting logs and metrics locally.

要为 Mixer 实现新的适配器，请参阅适[配器开发人员指南](https://github.com/istio/istio/blob/master/mixer/doc/adapters.md)。