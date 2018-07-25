# 适配器

Mixer适配器让Istio连接到各种基础设施后端，用于指标和日志等。

- [Circonus](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/circonus/). 用于 circonus.com 监控解决方案的适配器
- [CloudWatch](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/cloudwatch/). 用于 cloudwatch 度量的适配器
- [Datadog](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/datadog/). 将度量发送给 dogstatsd 代理以便发送给 DataDog 的适配器
- [Denier](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/denier/). 总是返回前置条件结果为拒绝的适配器
- [Fluentd](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/fluentd/). 发送日志到 fluentd 守护进程的适配器
- [Kubernetes Env](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/kubernetesenv/). 从 Kubernetes 环境中提取信息的适配器
- [List](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/list/). 执行白名单或者黑名单的适配器
- [Memory quota](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/memquota/). 提供简单内存内配额管理系统的适配器
- [OPA](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/opa/). 实现 Open Policy Agent 引擎的适配器
- [Prometheus](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/prometheus/). 暴露Istio指标，供Prometheus harvester 获取的适配器
- [RBAC](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/rbac/). 暴露Istio基于角色的访问控制(RBAC)模型的适配器。
- [Redis Quota](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/redisquota/). 用于基于 Redis 的配额管理系统的适配器
- [Service Control](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/servicecontrol/). 发送日志和度量到 Google Service Control 的适配器
- [SolarWinds](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/solarwinds/). 发送日志和度量到 Papertrail 和 AppOptics 后端的适配器
- [Stackdriver](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/stackdriver/). 发送日志和度量到 Stackdriver 的适配器
- [StatsD](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/statsd/). 发送度量到 StatsD 后端的适配器
- [Stdio](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/stdio/). 本地输出日志和指标的适配器

要为 Mixer 实现新的适配器，请参阅适[配器开发人员指南](https://github.com/istio/istio/blob/master/mixer/doc/adapters.md)。