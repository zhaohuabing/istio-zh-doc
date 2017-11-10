# FAQ

* 我的应用程序无法运行，我应该在哪里调试？

  请确保所有必需的容器都在运行：包括 etcd、istio-apiserver、consul、registrator、pilot 等。如果有任何一个容器没有运行，您可以使用 `docker ps -a` 命令来查找 {containerID}，然后使用 `docker logs {contaienrID}` 来查看容器日志。

* 最终我该如何使用 istioctl 取消 context 的设置？

  在使用 `istio context-creat` 命令之后，您的 `kubectl` 就切换到了 istio 的上下文了。您可以使用 `kubectl config get-contexts` 命令来获取上下文列表，然后使用 `kubectl config use-context {希望使用的上下文}` 来切换到您想要使用的上下文环境。
