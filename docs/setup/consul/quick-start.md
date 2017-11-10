# 使用 Docker 快速开始

使用  Docker Compose 快速安装 Istio service mesh 指南。


## 前置条件

* [Docker](https://docs.docker.com/engine/installation/#cloud)
* [Docker Compose](https://docs.docker.com/compose/install/)

## 安装步骤

1. 到 [Istio release](https://github.com/istio/istio/releases) 页面根据您自己的操作系统下载相应的安装文件。如果您使用的是 MacOS 或者 Linux 系统的话，可以直接执行下面的命令自动下载和安装：
   ```bash
   curl -L https://git.io/getLatestIstio | sh -
   ```

2. 解压下载好的文件，切换到文件在目录。安装文件目录下包含：
    * `install/` 目录下是 kubernetes 使用的 `.yaml` 安装文件
   * `samples/` 目录下是示例程序
   * `istioctl` 客户端二进制文件在 `bin` 目录下。`istioctl` 文件用户手动注入 Envoy sidecar 代理、创建路由和策略等。
   * `istio.VERSION` 配置文件

3. 将 `istioctl` 客户端二进制文件加到 PATH 中。
   例如，在 MacOS 或 Linux 系统上执行下面的命令：

   ```bash
   export PATH=$PWD/bin:$PATH
   ```

4. 对于 Linux 用户，配置 `DOCKER_GATEWAY` 环境变量：

   ```bash
   export DOCKER_GATEWAY=172.28.0.1:
   ```

5. 切换根目录到 Istio 的安装目录。

6. 启动 Istio 控制平面容器：

    ```bash
    docker-compose -f install/consul/istio.yaml up -d
    ```

7. 确认所有的 docker 容器都在运行：

   ```bash
   docker ps -a
   ```

   > 如果 Istio Pilot 容器终止了，确认你使用了 `istio context-create` 命令并重新运行上一步。

8. 配置 `istioctl` 使用 Istio API server 映射的本地端口：

    ```bash
    istioctl context-create --api-server http://localhost:8080
    ```

## 部署应用

现在您可以部署自己的应用了，也可以部署我们提供的示例应用，例如 [BookInfo]({{home}}/docs/guides/bookinfo.html).

> 注意 1：由于在安装 Docker 的时候没有 pod 的概念，sidecar 和应用都运行在同一个容器里。我们会使用 [Registrator](http://gliderlabs.github.io/registrator/latest/) 在 Consul 服务注册表中注册服务的实例。

> 注意 2：应用程序必须使用 HTTP/1.1 或 HTTP/2.0 协议进行 HTTP 请求，因为不支持 HTTP/1.0 的流量。

```bash
docker-compose -f <your-app-spec>.yaml up -d
```

## 卸载

1. 卸载 Istio 核心组件，只要删除 docker 容器即可：

```bash
docker-compose -f install/consul/istio.yaml down
```

## 下一步

* 参阅 [BookInfo](../../../docs/guides/bookinfo.html) 应用程序示例。
