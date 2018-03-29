# 控制 Egress TCP 流量
如[控制Egress流量](http://istio.doczh.cn/docs/tasks/traffic-management/egress.html)告诉我们可以从服务网格内部应用访问外部(指在Kubernetes外的服务)  HTTP 和 HTTPS 服务。默认情况下，支持 istio 的应用程序无法直接访问集群外部的 URL 。要启用这种访问，必须先定义 [Egress 规则](http://istio.doczh.cn/docs/concepts/traffic-management/rules-configuration.html#EgressRule)或者配置[直接调用外部服务](http://istio.doczh.cn/docs/tasks/traffic-management/egress.html#直接调用外部服务
)规则。

此任务描述如何配置 Istio 内应用如何访问 Istio 外部的应用。

## 开始之前
* 遵循安装指南设置Istio
* 启动sleep示例，用于测试外部访问。
```
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
```
注意: 所有可以在容器中执行`exec`和`curl`的 pod 都是可以用于本任务的。

## 为外部TCP服务设置路由规则
在这个任务中，我们将通过由应用发起 HTTPS 请求访问 `wikipedia.org` 。这个任务将演示应用无法使用 HTTP 与 HTTPS 协议访问外部服务的用例。[控制Egress流量](http://istio.doczh.cn/docs/tasks/traffic-management/egress.html)中描述了访问 HTTP 与 HTTPS 协议的外部应用，在[控制Egress流量](http://istio.doczh.cn/docs/tasks/traffic-management/egress.html)任务中，`https://www.google.com`通过使用`http://www.google.com:443`来进行访问。

由应用发起的 HTTPS 流量将被视为不透明的 TCP 。为了启用这种流量，我们在端口443上定义一个 TCP Egress 规则。

在 TCP Egress 规则中，与基于 HTTP Egress 规则相反，目标服务地址由 IP 或者 IP 地址块(遵循[CIDR notation](https://tools.ietf.org/html/rfc2317))指定.

假设通过域名`wikipedia.org`访问该实例。这意味我们必须指定 `wikipedia.org` 对应的所有 IP 的 TCP Egress 规则。幸运的是，  `wikipedia.org`的所有 IP 地址在[这里](https://www.mediawiki.org/wiki/Wikipedia_Zero/IP_Addresses)发布。它是遵循[CIDR notation](https://tools.ietf.org/html/rfc2317)的一个 IP 列表: `91.198.174.192/27, 103.102.166.224/27 ...`，我们必须为每个IP块定义一个 TCP Egress 规则。
或者， 如果通过 IP 访问 `wikipedia.org` 的话， 则必须定义该 IP 的 Tcp Egress 规则。

## 创建出口规则
让我们创建出口规则来启用TCP访问 `wikipedia.org` ：
```
cat <<EOF | istioctl create -f -
kind: EgressRule
metadata:
  name: wikipedia-range1
spec:
  destination:
      service: 91.198.174.192/27
  ports:
      - port: 443
        protocol: tcp
---
kind: EgressRule
metadata:
  name: wikipedia-range2
spec:
  destination:
      service: 103.102.166.224/27
  ports:
      - port: 443
        protocol: tcp
---
kind: EgressRule
metadata:
  name: wikipedia-range3
spec:
  destination:
      service: 198.35.26.96/27
  ports:
      - port: 443
        protocol: tcp
---
kind: EgressRule
metadata:
  name: wikipedia-range4
spec:
  destination:
      service: 208.80.153.224/27
  ports:
      - port: 443
        protocol: tcp
---
kind: EgressRule
metadata:
  name: wikipedia-range5
spec:
  destination:
      service: 208.80.154.224/27
  ports:
      - port: 443
        protocol: tcp
EOF
```
这个命令将创建五个 TCP Egress 规则，`wikipedia.org`每个不同的 IP 地址块都有一个规则。请注意`---`（三个破折号）是 Egress 规则之间的分隔符。这是流中文档之间的YAML分隔符。它允许我们使用单个 istioctl 命令创建多个配置工件。

## 通过HTTPS访问wikipedia.org
1. `kubectl exec`进入 pod 并执行测试代码。如果您正在使用 [sleep](https://github.com/istio/istio/tree/master/samples/sleep) 应用程序，请运行以下命令：
```
kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c sleep bash
```
2. 发起请求并确认我们可以成功访问 `https://www.wikipedia.org` ：
```
curl -o /dev/null -s -w "%{http_code}\n" https://www.wikipedia.org
200
```
我们应该看到200打印输出，这是正确返回的 HTTP Code。
3. 现在让我们获取英文维基百科上拥有文章的当前数量：
```
curl -s https://en.wikipedia.org/wiki/Main_Page | grep articlecount | grep 'Special:Statistics'
```
输出应该类似于：
```
<div id="articlecount" style="font-size:85%;"><a href="/wiki/Special:Statistics" title="Special:Statistics">5,563,121</a> articles in <a  href="/wiki/English_language" title="English language">English</a></div>
```
这意味着在撰写此任务时，英文维基百科中有5,563,121篇文章。
## 清理
1. 删除我们创建的出口规则。
```
istioctl delete egressrule wikipedia-range1 wikipedia-range2 wikipedia-range3 wikipedia-range4 wikipedia-range5 -n default
```
2. 关闭 [sleep](https://github.com/istio/istio/tree/master/samples/sleep) 应用程序。
```
kubectl delete -f samples/sleep/sleep.yaml
```

## 下一步是什么
* 该[Egress 规则配置](http://istio.doczh.cn/docs/concepts/traffic-management/rules-configuration.html#EgressRule)的参考。
* 该[控制Egress流量](http://istio.doczh.cn/docs/tasks/traffic-management/egress.html)流量的任务，HTTP和HTTPS。
