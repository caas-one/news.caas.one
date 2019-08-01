# 在kubernetes中使用Let's Encrypt 通配符证书 

通配符证书可以保护基域（例如，.*example.com）的任意数量的子域。这允许对一个域及其所有子域使用一个证书和密钥对，可以明显简化https部署。

[Let's Encrypt](https://letsencrypt.org/) 通配符证书支持2018年3月上线。 

在本博客文章中，您将学习如何设置[Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 控制器 [Heptio Contour](https://github.com/heptio/contour)，使用[Jetstack Cert-Manager](https://github.com/jetstack/cert-manager) 自动管理和颁发通配符TLS证书，并使用[AppsCode Kubed](https://appscode.com/products/kubed/0.7.0/guides/config-syncer/).同步不同命名空间中的TLS证书。 



## 先决条件

您需要拥有[Kubernetes](https://kubernetes.io/)集群，如果您没有，请按照下面的文档之一设置： 

  · [Google GKE](https://cloud.google.com/kubernetes-engine/)

  · [Azure AKS](https://azure.microsoft.com/en-gb/services/kubernetes-service/)

  · [AWS EKS](https://aws.amazon.com/eks/)

  · [Mesosphere DC/OS Kubernetes-as-a-Service](https://github.com/mesosphere/dcos-kubernetes-quickstart)

[Helm](https://docs.helm.sh/)安装在您的Kubernetes集群中



## 输入控制器 

好的，首先我们需要安装Kubernetes入口控制器。

由于没有Heptio Contour的Helm图表，我编写了[图表](https://github.com/rimusz/charts/tree/master/stable/contour)并将其存储在我的Helm存储库中。为了能够使用它，您需要将我的图表仓库添加到Helm：

```go
$ helm repo add rimusz https://helm-charts.rimusz.net
$ helm repo up
```

让我们安装contour图表：

```go
helm install --name contour rimusz/contour
```

检查入口控制器是否正在运行： 

```go
$ kubectl --namespace=heptio-contour get pods -l "app=contour"
NAME                       READY     STATUS    RESTARTS   AGE
contour-5d7f6fc8bd-nv474   2/2       Running   0          20s
```

LoadBalancer IP可能需要几分钟时间才能提供。您可以通过以下方式查看状态：

```go
$ kubectl get svc --namespace heptio-contour -w contour
NAME     TYPE          CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
contour  LoadBalancer  10.64.12.134   <pending>     80:30321/TCP,443:32400/TCP   34s
contour  LoadBalancer  10.64.12.134   35.238.152.138   80:30321/TCP,443:32400/TCP   46s
```

现在请使用外部IP更新您的域DNS记录。              

很好，我们安装了入口控制器。 



## TLS证书管理器 

![1563863333413](https://rimusz.net/static/89d9975dd182f8667ee0d2fed7e2f58e/17788/high-level-overview.png)

我们将从Jetstack repo安装cert manager： 

```go
$ helm repo add jetstack https://charts.jetstack.io
$ kubectl apply \
    -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml
$ helm install --name cert-manager --namespace cert-manager stable/cert-manager
```

检查是否运行：

```go
$ kubectl get pods -n cert-manager
NAME                                   READY   STATUS      RESTARTS   AGE
cert-manager-5f8db6f6c4-jjzjc          1/1     Running     0          22s
cert-manager-webhook-85dd96d87-jsfjk   1/1     Running     0          22s
cert-manager-webhook-ca-sync-vwt57     0/1     Completed   0          22s
```

棒极了。



## 使用Cert-Manager生成通配符证书 

对于这篇博文，我使用CloudFlare（抱歉，我是他们服务的忠实粉丝）作为DNS01 Challenge Provider，请在[这里](https://cert-manager.readthedocs.io/en/latest/reference/issuers/acme/dns01.html)查看其他支持的内容。 

现在我们需要创建一个具有[CloudFlare Global API Key](https://support.cloudflare.com/hc/en-us/articles/200167836-Where-do-I-find-my-Cloudflare-API-key-)的secret，  带有DNS1 Challenge Provider的Cert-Manager发行人。他将使用这个secret和Cert-Manager证书（这将保存*.example.com的通配符证书）机密示例com tls： 

```go
$ cat <<EOF | kubectl create -f -  
---
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-key
  namespace: cert-manager
type: Opaque
data:
  api-key.txt: <copy here base64 encoded CloudFlare API Key>

---
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: cert-manager
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: user@example.com

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging

    # ACME DNS-01 provider configurations
    dns01:
      # Here we define a list of DNS-01 providers that can solve DNS challenges
      providers:
        - name: cf-dns
          cloudflare:
            email: user@example.com
            # A secretKeyRef to a cloudflare api key
            apiKeySecretRef:
              name: cloudflare-api-key
              key: api-key.txt

---
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com
  namespace: cert-manager
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-staging
  commonName: '*.example.com'
  acme:
    config:
    - dns01:
        provider: cf-dns
      domains:
      - '*.example.com'

EOF
```

注意：请使用您的CloudFlare API密钥、电子邮件地址和域名更新上面的示例模板。 注意：请使用您的CloudFlare API密钥、电子邮件地址和域名更新上面的示例模板。 

如果一切顺利，您应该看到您的通配符证书示例COM TLS: 

```go
$ kubectl -n cert-manager get secret
NAME                                    TYPE                                  DATA      AGE
cert-manager-cert-manager-token-whctf   kubernetes.io/service-account-token   3         4h
cloudflare-api-key                      Opaque                                1         17m
default-token-cfmln                     kubernetes.io/service-account-token   3         4h
letsencrypt-staging                     Opaque                                1         37m
example-com-tls                         kubernetes.io/tls                     2         12m
```

棒极了。



## 在Kubernetes命名空间之间同步通配符证书 

Kubernetes最佳实践建议是在每个命名空间安装应用程序，但我们的通配符证书example-com-tls存储在cert-manager命名空间中，因此当重新发布证书或添加新名称空间中的新应用程序时，我们需要跨名称空间同步它。

为此，我们将使用AppScode Kubed提供的一个很好的工具。检查他们的其他工具，你可以找到更有用的工具。 

Kubed可以通过Helm安装，使用AppScode[图表库](https://github.com/appscode/charts)中的[图表](https://github.com/appscode/kubed/tree/master/chart/kubed )： 

```go
$ helm repo add appscode https://charts.appscode.com/stable/
$ helm repo update
$ helm install appscode/kubed --name kubed --namespace kube-system \
    --set apiserver.enabled=false \
    --set config.clusterName=my_staging_cluster
```

注意：设置集群名称为对您有意义的东西，例如prod、prod-us-east、qa等，这也启用kubed的配置同步器功能。 

要验证kubed是否已启动，请运行： 

```go
$  kubectl --namespace=kube-system get deployments -l "release=kubed, app=kubed"
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubed-kubed   1         1         1            1           15s
```

稳健。

好的，现在让我们创建几个名称空间，我们将把通配符证书同步到： 

```go
$ kubectl create namespace demo1
$ kubectl create namespace demo2
```

现在，我们需要根据docs将注释放在通配符机密中，这样kubed知道什么可以在命名空间之间同步：

```go
$ kubectl annotate secret example-com-tls -n cert-manager kubed.appscode.com/sync="app=kubed"
secret "example-com-tls" annotated
```

从现在起kubed将开始将机密 example-com-tls同步到只有标签选择器app=kubed的命名空间。 

让我们注释命名空间demo1：

```go
$ kubectl label namespace demo1 app=kubed
```

现在我们应该能够在名称空间demo1中看到机密： 

```go
$ kubectl -n demo1 get secret example-com-tls
NAME              TYPE                DATA      AGE
example-com-tls   kubernetes.io/tls   2         2h
```

如果我们检查名称空间demo2，我们将看不到 example-com-tls ：

```go
$ kubectl -n demo2 get secret example-com-tls
Error from server (NotFound): secrets "example-com-tls" not found
```

将注释放在demo2命名空间上也将触发即时机密同步。

现在，您可以将Web应用程序安装到demo1和demo2名称空间中，并且应用程序可以使用相同的通配符证书作为其入口规则。 

## 下一步该做什么

按照下面的示例，继续生产发布并从let's encrypt接收真实的TLS证书，添加新的颁发者和证书： 

```go
$ cat <<EOF | kubectl create -f -  
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: user@example.com

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod

    # ACME DNS-01 provider configurations
    dns01:
      # Here we define a list of DNS-01 providers that can solve DNS challenges
      providers:
        - name: cf-dns-prod
          cloudflare:
            email: user@example.com
            # A secretKeyRef to a cloudflare api key
            apiKeySecretRef:
              name: cloudflare-api-key
              key: api-key.txt

---
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com-prod
  namespace: cert-manager
spec:
  secretName: example-com-tls-prod
  issuerRef:
    name: letsencrypt-prod
  commonName: '*.example.com'
  acme:
    config:
    - dns01:
        provider: cf-dns-prod
      domains:
      - '*.example.com'

EOF
```

快乐安全网络内容为您服务:)



## 总结

在这篇博客文章中，我们学习了如何安装Heptio Contour Ingress Controller，安装Jetstack Cert-Manager和设置它以发布通配符证书Let's Encrypt，并借助于AppsCode Kubed在标记的命名空间中同步通配符证书。




原文作者：Rimas Mocevicius

原文链接：https://rimusz.net/lets-encrypt-wildcard-certs-in-kubernetes
