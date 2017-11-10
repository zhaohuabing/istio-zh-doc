# 安装

> 注意：在 Nomad 上安装还没有测试过。

在非 kubernetes 环境中使用 Istio 有几个关键问题：

1. 使用 Istio API server 创建 Istio 控制平面
2. 向服务的每个实例中注入 Istio sidecar
3. 确认通过 sidecar 来路由请求

## 安装控制平面

Istio 控制平面由四个主要服务组成：Pilot、Mixer、CA 和 API server。

### API Server

Istio API server（基于 Kubernetes API server）提供了诸如配置管理和基于角色的访问控制（RBAC）等功能。API server 需要一个 [etcd 集群](https://kubernetes.io/docs/getting-started-guides/scratch/#etcd) 作为持久化存储。可以在 [这里](https://kubernetes.io/docs/getting-started-guides/scratch/#apiserver-controller-manager-and-scheduler) 找到关于设置 API server 的详细说明。 

关于设置 Kubernetes API server 的选项的说明请参阅 [这里](https://kubernetes.io/docs/admin/kube-apiserver/)。

#### 本地安装

处于 POC（概念验证） 的目的，可以使用 Docker-compose 文件来启用一个简单的单容器 API server：

```yaml
version: '2'
services:
  etcd:
    image: quay.io/coreos/etcd:latest
    networks:
      istiomesh:
        aliases:
          - etcd
    ports:
      - "4001:4001"
      - "2380:2380"
      - "2379:2379"
    environment:
      - SERVICE_IGNORE=1
    command: [
              "/usr/local/bin/etcd",
              "-advertise-client-urls=http://0.0.0.0:2379",
              "-listen-client-urls=http://0.0.0.0:2379"
             ]

  istio-apiserver:
    image: gcr.io/google_containers/kube-apiserver-amd64:v1.7.3
    networks:
      istiomesh:
        ipv4_address: 172.28.0.13
        aliases:
          - apiserver
    ports:
      - "8080:8080"
    privileged: true
    environment:
      - SERVICE_IGNORE=1
    command: [
               "kube-apiserver", "--etcd-servers", "http://etcd:2379", 
               "--service-cluster-ip-range", "10.99.0.0/16", 
               "--insecure-port", "8080", 
               "-v", "2", 
               "--insecure-bind-address", "0.0.0.0"
             ]
```


### 其他 Istio 组件

Istio Pilot、Mixer 和 CA  的 Debian 软件包可以通过 Isito 的发行版获得。或者可以以 docker 容器的方式运行（docker.io/istio/pilot、docker.io/istio/mixer、docker.io/istio/istio-ca）。请注意，这些组件是无状态的，可以水平缩放。这些组件都依赖于 Istio API server，而 Istio API server 由依赖 etcd 集群来实现持久化。为了实现高可用，每个控制平面服务都可以在 Nomad 中以 [job](https://www.nomadproject.io/docs/job-specification/index.html) 的方式运行，其中 [service stanza](https://www.nomadproject.io/docs/job-specification/service.html) 可以用来描述控制平面服务的期望属性。


## 向服务实例中添加 Sidecar

应用程序的每个实例中都必须伴有一个 Istio sidecar。根据您的安装单位（Docker 容器、虚拟机或者是裸机），Istio sidecar 都需要安装到这些组件中。例如，如果您的基础架构使用的是虚拟机，则必须在每台虚拟机上运行一个 Istio sidecar 进程才能将这台迅即添加到 service mesh 中。

有种将 sidecar 打包到机遇 Nomad 的部署中去的方式，就是将 Istio sidecar 进程作为 [任务组](https://www.nomadproject.io/docs/job-specification/group.html) 中的一个进程。任务组是一个或多个相关任务的集合，这些任务保证驻留在同一台主机上。但是，与 kuberntes pod 不同的是，统一组中的任务不共享相同的网络命名空间。因此，当使用 `iptables` 规则透明地重新路由所有网络流量时，必须注意确保每台主机上只运行一个任务组。当 Istio 支持非透明代理（应用程序明确的与 sidecar 通信）时，将不会再有此限制。

## 通过 Istio Sidecar 路由流量

部分 Sidecar 安装时会设置适当的 IP Table 规则，以通过 Istio sidecar 透明地路由应用程序的网络流量。在 [这里](https://github.com/istio/istio/blob/master/pilot/docker/prepare_proxy.sh) 可以找到设置这种转发规则的 IP Table 脚本。

> 注意：该脚本必须再启动应用程序和 sidecar 之前执行。
