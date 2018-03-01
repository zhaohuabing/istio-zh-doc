# 插入CA证书和密钥

本任务将演示将现有的证书和密钥添加到 Istio CA 中。

默认情况下，Istio CA 生成自签名 CA 证书和密钥，并使用它们签署工作负载证书。Istio CA 也可以使用我们指定的证书和密钥来签署工作负载证书。接下来我们将演示将证书和密钥插入 Istio CA 的示例。

## 开始之前

- 根据[安装指南](../../setup/kubernetes/quick-start.md)中的快速入门指南，在 Kubernetes 集群中安装 Istio。
- 在安装步骤的第四步应该启动身份验证

## 插入现有的证书和密钥

假设我们想让 Istio CA 使用已经存在的 ca-cert.pem 的证书和密钥。此外，证书 ca-cert.pem 由根证书 root-cert.pem 签名，我们希望使用使用 root-cert.pem 作为 Istio 工作负载的根证书。

在此示例中，由于 Istio CA 证书（ca-cert.pem）未设置为工作负载的根证书（root-cert.pem），因此工作负载无法直接从根证书验证工作负载证书。工作负载需要一个 cert-chain.pem 文件来指定信任链，该信任链应包括工作负载和根 CA 之间的所有中间 CA 的证书。在这个例子中，它只包含 Istio CA 证书，所以 cert-chain.pem 和 ca-cert.pem 是一样的。请注意，如果您的 ca-cert.pem 与 root-cert.pem 相同，您可以有一个空的 cert-chain.pem 文件。

将证书和密钥插入 Istio CA 分为以下几步：

1. 创建一个密钥 cacert 包含所有输入文件，例如：ca-cert.pem，ca-key.pem，root-cert.pem 和 cert-chain.pem

   ```bash
   kubectl create secret generic cacerts -n istio-system --from-file=install/kubernetes/ca-cert.pem --from-file=install/kubernetes/ca-key.pem \
   --from-file=install/kubernetes/root-cert.pem --from-file=install/kubernetes/cert-chain.pem
   ```

2. 重新部署 Istio CA，从自定义的安装文件读取证书和密钥

   ```bash
   kubectl apply -f install/kubernetes/istio-ca-plugin-certs.yaml
   ```

3. 为了确保工作负载及时获得新的证书，请删除 Istio CA 生成的密钥文件（以 istio.* 命名）。在本例中是 istio.default。Istio CA 将为工作负载颁发新的证书。

   ```bash
   kubectl delete secret istio.default
   ```

请注意，如果你使用不同的证书/密钥文件或密钥名称，则需要更改 istio-ca-plugin-certs.yaml 中的相应参数。

## 验证新的证书

在本节中，我们将验证新的工作负载证书和根证书是否生效，这需要你的机器上安装 OpenSSL。

1. 按照[说明](https://istio.io/docs/guides/bookinfo.html)部署 bookinfo 应用程序

2. 检索挂载的证书

   获取 pods：

   ```bash
   kubectl get pods
   ```

   列举 producees：

   ```
   NAME                                        READY     STATUS    RESTARTS   AGE
   details-v1-1520924117-48z17                 2/2       Running   0          6m
   productpage-v1-560495357-jk1lz              2/2       Running   0          6m
   ratings-v1-734492171-rnr5l                  2/2       Running   0          6m
   reviews-v1-874083890-f0qf0                  2/2       Running   0          6m
   reviews-v2-1343845940-b34q5                 2/2       Running   0          6m
   reviews-v3-1813607990-8ch52                 2/2       Running   0          6m
   ```

   接下来，我们以 pod ratings-v1-734492171-rnr5l 为例，并验证已安装的证书。运行以下命令以检索安装在代理上的证书。

   ```bash
   kubectl exec -it ratings-v1-734492171-rnr5l -c istio-proxy -- /bin/cat /etc/certs/root-cert.pem > /tmp/pod-root-cert.pem
   ```

   文件 /tmp/pod-root-cert.pem 应该包含由操作员指定的根证书。

   ```bash
   kubectl exec -it ratings-v1-734492171-rnr5l -c istio-proxy -- /bin/cat /etc/certs/cert-chain.pem > /tmp/pod-cert-chain.pem
   ```

   文件 /tmp/pod-cert-chain.pem 应该包含工作负载证书和CA证书。

3. 验证根证书是否与操作员指定的相同：

   ```bash
   openssl x509 -in /tmp/root-cert.pem -text -noout > /tmp/root-cert.crt.txt
   openssl x509 -in /tmp/pod-root-cert.pem -text -noout > /tmp/pod-root-cert.crt.txt
   diff /tmp/root-cert.crt.txt /tmp/pod-root-cert.crt.txt
   ```

4. 验证CA证书是否与操作员指定的相同：

   ```bash
   tail /tmp/pod-cert-chain.pem -n 22 > /tmp/pod-cert-chain-ca.pem
   openssl x509 -in /tmp/ca-cert.pem -text -noout > /tmp/ca-cert.crt.txt
   openssl x509 -in /tmp/pod-cert-chain-ca.pem -text -noout > /tmp/pod-cert-chain-ca.crt.txt
   diff /tmp/ca-cert.crt.txt /tmp/pod-cert-chain-ca.crt.txt
   ```

   预计输出为空。

5. 验证从根证书到工作负载证书的证书链：

   ```bash
   head /tmp/pod-cert-chain.pem -n 18 > /tmp/pod-cert-chain-workload.pem
   openssl verify -CAfile <(cat /tmp/ca-cert.pem /tmp/root-cert.pem) /tmp/pod-cert-chain-workload.pem
   ```

   预计输出为：

   ```
   /tmp/pod-cert-chain-workload.pem: OK
   ```

## 清理

- 删除密钥 cacerts。

https://istio.io/docs/tasks/security/plugin-ca-cert.html