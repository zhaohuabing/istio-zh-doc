# 安装 Istio sidecar

**备注**：以下需要 Istio 0.5.0 或更高。 0.4.0 及 以前版本参见 https://archive.istio.io/v0.4/docs/setup/kubernetes/sidecar-injection。

## Pod Spec 中需满足的条件

为了成为 Service Mesh 中的一部分，kubernetes 集群中的每个 Pod 都必须满足如下条件：

1. **Service 关联**：每个 pod 都必须只属于某**一个** [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/) （当前不支持一个 pod 同时属于多个 service）。
2. **命名的端口**：Service 的端口必须命名。端口的名字必须遵循如下格式 `<protocol>[-<suffix>]`，可以是_http_、_http2_、 _grpc_、 _mongo_、 或者 _redis_ 作为 `<protocol>` ，这样才能使用 Istio 的路由功能。例如`name: http2-foo` 和 `name: http` 都是有效的端口名称，而 `name: http2foo` 不是。如果端口的名称是不可识别的前缀或者未命名，那么该端口上的流量就会作为普通的 TCP 流量（除非使用 `Protocol: UDP` 明确声明使用 UDP 端口）。
3. **带有 app label 的 Deployment**：我们建议 kubernetes 的`Deploymenet` 资源的配置文件中为 Pod 明确指定 `app` label。每个Deployment 的配置中都需要有个不同的有意义的 `app` 标签。`app` label 用于在分布式坠重中添加上下文信息。
4. **Mesh 中的每个 pod 里都有一个 Sidecar**：最后，Mesh 中的每个 pod 都必须运行与 Istio 兼容的sidecar。以下部分介绍了将 sidecar 注入到 pod 中的两种方法：使用`istioctl` 命令行工具手动注入，或者使用 istio initializer 自动注入。注意 sidecar 不涉及到容器间的流量，因为他们都在同一个 pod 中。



## 注入

手动注入需要修改控制的配置文件，如 deployment。通过修改 deployment 文件中的 pod 模板规范可达到该deployment 下创建的所有 pod 都注入 sidecar。添加/更新/删除 sidecar 需要修改整个 deployment。

自动注入会在 pod 创建的时候注入，无需控制资源。sidecars 可通过以下方式被更新：有选择性地手工删除 pod或者有条理地进行 deployment 滚动更新。

手动或者自动注入都使用同样的模板配置。自动注入会从  `istio-system`  命名空间下获取`istio-inject`的 ConfigMap。手动注入的模板可以是本地文件或者 Configmap 。

两个变种的注入模板在默认安装中也会被提供：`istio-sidecar-injector-configmap-release.yaml` 和 `istio-sidecar-injector-configmap-debug.yaml`。调试的版本包含调试的代理镜像、附加的日志和core dump功能以用于调试 sidecar 代理。

### 手动注入 sidecar

手工注入使用默认的模板和动态从 istio ConfigMap中获取服务网格的配置文件。附件参数覆盖可以通过程序的flags设置（参见：`istioctl kube-inject —help`）

```bash
kubectl apply -f <(~istioctl kube-inject -f samples/sleep/sleep.yaml)
```

`kube-inject` 也可以在没有 kubernetes 集群的情况下运行。创建本地的注入和网格配置。

```bash
kubectl create -f install/kubernetes/istio-sidecar-injector-configmap-release.yaml \
    --dry-run \
    -o=jsonpath='{.data.config}' > inject-config.yaml
    
kubectl -n istio-system get configmap istio -o=jsonpath='{.data.mesh}' > mesh-config.yaml
```

在输入的文件上运行`kube-inject`：

```bash
istioctl kube-inject \
    --injectConfigFile inject-config.yaml \
    --meshConfigFile mesh-config.yaml \
    --filename samples/sleep/sleep.yaml \
    --output sleep-injected.yaml
```

部署注入后的 YAML 文件：

```bash
kubectl apply -f sleep-injected.yaml 
```

验证 sidecar 已经注入到 Deployment 中：

```bash
kubectl get deployment sleep -o wide
```
```bash
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS          IMAGES                             SELECTOR
sleep     1         1         1            1           2h        sleep,istio-proxy   tutum/curl,unknown/proxy:unknown   app=sleep
```



## 自动注入 sidecar

关于Webhook管理控制的整体介绍可以参见： [validatingadmissionwebhook-alpha-in-18-beta-in-19](https://kubernetes.io/docs/admin/admission-controllers/#validatingadmissionwebhook-alpha-in-18-beta-in-19)。

### 前置条件

kubernetes 1.9 的集群需要启动 `admissionregistration.k8s.io/v1beta1`。

```bash
kubectl api-versions | grep admissionregistration.k8s.io/v1beta1
```

```bash
Copyadmissionregistration.k8s.io/v1beta1
```

#### _GKE_

1.9.1 版本中已经具备非白名单早期用户访问的 alpha 集群  (参考：https://cloud.google.com/kubernetes-engine/release-notes#january-16-2018）。

```bash
gcloud container clusters create <cluster-name> \
    --enable-kubernetes-alpha 
    --cluster-version=1.9.1-gke.0 
    --zone=<zone>
    --project <project-name>
```

```bash
Copygcloud container clusters get-credentials <cluster-name> \
    --zone <zone> \
    --project <project-name>
```

```bash
Copykubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)
```
#### minikube
TODO(https://github.com/istio/istio.github.io/issues/885)

#### IBM Cloud Container Service
TODO(https://github.com/istio/istio.github.io/issues/887)

#### AWS with Kops
TODO(https://github.com/istio/istio.github.io/issues/886)


#### 安装 Webhook

安装基本的 Istio

```bash
kubectl apply -f install/kubernetes/istio.yaml
```

Webhook 需要签名的 cert/key 对。使用 `install/kubernetes/webhook-create-signed-cert.sh` 生成 kuberntes CA 签发的 cert/key 对，结果文件 cert/key 被 sidecar 注入的 webhook 当做 kuberntes 密钥去使用。

*备注：* Kubernetes CA 批准（approval）需要权限能够创建和验证 CSR。 更多信息参见：https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster 和 `install/kubernetes/webhook-create-signed-cert.sh` 。

```bash
./install/kubernetes/webhook-create-signed-cert.sh \
    --service istio-sidecar-injector \
    --namespace istio-system \
    --secret sidecar-injector-certs
```

安装 sidecar 注入的 configmap：

```bash
kubectl apply -f install/kubernetes/istio-sidecar-injector-configmap-release.yaml
```

设置安装 yaml 中 webhook 的 `caBundle`，以便 api-server能够用来调用 webhook。

```bash
cat install/kubernetes/istio-sidecar-injector.yaml | \
     ./install/kubernetes/webhook-patch-ca-bundle.sh > \
     install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```

安装 sidecar 注册器 webhook。

```bash
kubectl apply -f install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```

sidecar 注入的 webhook 应该已经运行。

```bash
kubectl -n istio-system get deployment -listio=sidecar-injector
```

```bash
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
istio-sidecar-injector   1         1         1            1           1d
```

NamespaceSelector 决定是否在一个对象上运行 webhook，取决于该对象的所在的 namespace 是否匹配选择器

 (参见： https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)。默认webhook 的配置使用 `istio-injection=enabled`.

采用标签 `istio-injection` 查看 namespae，用以确认  `default` 命名空间下没有被打标签。

```
Copykubectl get namespace -L istio-injection

CopyNAME           STATUS        AGE       ISTIO-INJECTION
default        Active        1h        
istio-system   Active        1h        
kube-public    Active        1h        
kube-system    Active        1h

```


#### 部署程序

部署 sleep 程序。验证 deployment 和 pod 都有一个单独的container。

```bash
kubectl apply -f samples/sleep/sleep.yaml 
```
```bash
kubectl get deployment -o wide
```
```bash
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES       SELECTOR
sleep     1         1         1            1           12m       sleep        tutum/curl   app=sleep
```
```bash
kubectl get pod
```
```
NAME                     READY     STATUS        RESTARTS   AGE
sleep-776b7bcdcd-7hpnk   1/1       Running       0          4
```

 为 `default` 命名空间下的设置标签 `istio-injection=enabled`

```
kubectl label namespace default istio-injection=enabled
```
```bash
kubectl get namespace -L istio-injection
```
```
NAME           STATUS    AGE       ISTIO-INJECTION
default        Active    1h        enabled
istio-system   Active    1h        
kube-public    Active    1h        
kube-system    Active    1h  
```

注入动作会在 pod 创建的时候发生。杀掉运行的 pod 并验新创建的 pod 已经被注入 sidecar。原来老的 pod 具有 1/1 就绪的的 containers 而被注入的新的 pod 具有 2/2 就绪的 containers。

```
kubectl delete pod sleep-776b7bcdcd-7hpnk 
```
```bash
kubectl get pod
```
```
NAME                     READY     STATUS        RESTARTS   AGE
sleep-776b7bcdcd-7hpnk   1/1       Terminating   0          1m
sleep-776b7bcdcd-bhn9m   2/2       Running       0          7s
```

在 `default`命名空间下禁用注入，验证新的 pod 中 sidecar 不再存在。

```
kubectl label namespace default istio-injection-
```
```
kubectl delete pod sleep-776b7bcdcd-bhn9m 
```
```
kubectl get pod
```
```
NAME                     READY     STATUS        RESTARTS   AGE
sleep-776b7bcdcd-bhn9m   2/2       Terminating   0          2m
sleep-776b7bcdcd-gmvnr   1/1       Running       0          2s
```




#### 了解发生什么

[admissionregistration.k8s.io/v1alpha1#MutatingWebhookConfiguration](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.9/#mutatingwebhookconfiguration-v1beta1-admissionregistration) 
配置 webhook 何时被 kubernetes 调用。Istio 默认选择 namespace 下被打了标签 `istio-injection=enabled`的Pod。这个行为可以通过修改
`install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml`中的 MutatingWebhookConfiguration 配置。

 `istio-system` 命名空间下的 `istio-inject` ConfigMap 保存了默认注入策略（policy）和 sidecar 注入模板（template）。

##### _**策略（policy）**_

`disabled` - sidecar 注入器默认不会注入到 pod 中。添加pod模板定义中的注解  `sidecar.istio.io/inject` 值为 `true`会启用注入功能。

`enabled` - sidecar 注入器默认会注入到 pod 中。添加pod模板定义中的注解  `sidecar.istio.io/inject` 值为 `false`会禁止注入功能。
​    
##### _**模板（template）**_

sidecar 注入模板使用 https://golang.org/pkg/text/template，当解析和执行后会被解码成以下的 struct 结构，包含了一系列要注入 pod 的 containers 和 volumes。

```golang
    type SidecarInjectionSpec struct {
	      InitContainers []v1.Container `yaml:"initContainers"`
	      Containers     []v1.Container `yaml:"containers"`
	      Volumes        []v1.Volume    `yaml:"volumes"`
    }
```

模板会在运行的过程中被应用成以下的数据 struct：    
```golang
type SidecarTemplateData struct {
    ObjectMeta  *metav1.ObjectMeta        
    Spec        *v1.PodSpec               
    ProxyConfig *meshconfig.ProxyConfig  // Defined by https://istio.io/docs/reference/config/service-mesh.html#proxyconfig
    MeshConfig  *meshconfig.MeshConfig   // Defined by https://istio.io/docs/reference/config/service-mesh.html#meshconfig 
}
```

`ObjectMeta` 和 `Spec` 来自于 pod。 `ProxyConfig` 和 `MeshConfig` 来自于namespace ` istio-system`下的`istio` ConfigMap。模板可以用于条件定义注入的 containers 和 volumes。

例如，以下的模板片段，来自于 install/kubernetes/istio-sidecar-injector-configmap-release.yaml

```yaml
containers:
- name: istio-proxy
  image: istio.io/proxy:0.5.0
  args:
  - proxy
  - sidecar
  - --configPath
  - {{ .ProxyConfig.ConfigPath }}
  - --binaryPath
  - {{ .ProxyConfig.BinaryPath }}
  - --serviceCluster
  {{ if ne "" (index .ObjectMeta.Labels "app") -}}
  - {{ index .ObjectMeta.Labels "app" }}
  {{ else -}}
  - "istio-proxy"
  {{ end -}}
```
当 `sample/sleep/sleep.yaml` 中的pod定义被应用后会被扩展成：

```
containers:
- name: istio-proxy
  image: istio.io/proxy:0.5.0
  args:
  - proxy
  - sidecar
  - --configPath
  - /etc/istio/proxy
  - --binaryPath
  - /usr/local/bin/envoy
  - --serviceCluster
  - sleep
```

### 卸载 webhook

```
kubectl delete -f install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```

上述命令并不移除已经已经注入到 Pod 中的 Sidecar。如有需要可以通过滚动升级（rolling update）或者简单删除 pods 使 部署（deployment）重新创建 Pod。

