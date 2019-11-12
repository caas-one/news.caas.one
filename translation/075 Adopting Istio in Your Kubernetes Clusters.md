作者:Dave Konopka
时间:2018年10月16日

# 在你的Kubernetes集群中使用Istio

最近，Istio引起了业界的广泛关注。 该服务网格平台最近达到了1.0版本的里程碑，随时可能投入生产实践。 Google一直在嘲笑Google Cloud上一个关于Istio的托管项。 时刻都有很多资源在Kubernetes集群上运行。

您学习Istio至今，可能经过了多个阶段的技术迭代更新的周期

1.什么是服务网格？
2.服务网格有点复杂啊，我真的需要一个吗？
3.我可以看到Istio的一些应用场合了。 那么，通过这个“15分钟，Istio进阶与实践”应用示例，我需要花多少天才能掌握它，从而实际运行呢？

现在，您即将开始为已有的Kubernetes托管程序实际配置Istio。 这将花费15分钟多一些的时间。 这篇文章将清晰明确地告诉您提供将Istio引入Kubernetes集群所需要做的事情。

# 什么是Istio？
没什么意外的话，您可能已经听说过Istio是服务网格。 “服务网格”是一个优秀的项目，我们使用它来处理一组互联的服务之间的常规通信问题。

实际上，Istio的功能是建立在Kubernetes的基础之上的，它增加了：

- 服务之间相互使用TLS，来进行标识，授权和加密通信。
- 具有选择性白名单的出站流量限制。
- 动态流量分配模式，例如应用多版本分发和 "canary" 发布这两种方式。
- 通过断路器，重试处理，故障转移 和支持 "Chaos Monkey" 故障注入 提高系统的弹性。
- 还有大量的其他指标可以说明其通信模式和性能。

Istio通过在每个Pod中运行一个单独的代理随航容器来完成所有这些工作。 一组核心服务在您的集群中运行，并与这些代理端工具进行通信以启用上述功能。

# 计算机：为我安装一个Istio
为Kubernetes安装Istio的首选方法是Helm图表。 请注意，Istio的Helm图表是内嵌于Istio Github仓库中的。 通过官方社区仓库的Helm图表是不可用的。

该图表安装了Istio正常运行所需的核心服务的集合。 它通过图表的值提供了广泛的自定义。 而且它还支持将来升级到新的Istio版本。

在开始安装Istio之前，请继续阅读以了解一些特定的基于在Kubernetes上安装并运行的优势。

## 您想如何在所有Pod中安装随航代理？
您的所有pod都需要一个随航代理容器。Istio可以将此代理容器自动注入到新的pod中，而无需对部署进行任何更改。 在默认情况下即启用此功能，但它需要Kubernetes 1.9或更高版本。

您需要标识命名空间以启用随航注入：

```bash
$ kubectl create namespace my-app
$ kubectl label namespace my-app istio-injection=enabled
```
已有的pod将不受Istio图表安装的影响。 加载在被标识的命名空间中的Pod会接收到随航，而在其他的命名空间中加载的Pod则不会接收到。 这为安装Istio的核心服务以及选择性地将Pod过渡到网格中提供了一条途径。

## 您如何将服务过渡到双向TLS？
相互进行TLS加密是一种备受关注的安全机制，因为它限制了哪些服务之间是可以相互通信的，并对其通信进行了加密。 尽管在过渡服务时它们可能被证明是绊脚石，并且它们需要与网格外部的资源进行通信。

为了简化这个过渡的过程，请考虑启用PERMISSIVE(信任)模式的身份验证策略。 许可模式可为服务启用纯文本HTTP和mTLS通信。 该策略可以起作用于整个集群范围内，作为网格策略，每个命名空间的默认策略，或者是针对于单个服务的更精细的策略。当您的应用程序完全迁移到Istio后，您可以禁用许可模式并强制执行双向TLS加密。

还要注意的另一件事是，典型的Kubernetes HTTP活动行和就绪性探测无法在仅mTLS模式下工作，因为这种探测来自服务网格外部。 这些检查要在启用PERMISSIVE模式的情况下才工作。 一旦您移除PERMISSIVE模式后，您就需要将探测方式转换为Pod内部对于健康检测的运行结果进行投票的EXEC检查，或者在一个禁用了mTLS的端口上，连接并建立健康检查终端。

您可以按照以下方法创建网状策略，在PERMISSIVE模式下启用mTLS，以这种方式启动整个集群：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: “authentication.istio.io/v1alpha1”
kind: “MeshPolicy”
metadata:
  name: “default”
spec:
  peers:
  — mtls:
      mode: PERMISSIVE
 EOF
```
## 您要启动服务并开启出口过滤吗？
Istio默认情况下限制pod的出站流量。这意味着，如果您的Pod与网格外部的服务（例如云提供商API，第三方API或托管数据库）进行通信，则流量传输将被阻止。 不过，您可以使用ServiceEntries选择性地启用对外部服务的访问。
 
从表面上看，出口过滤看起来像是一种吸引人的安全功能。 但是，它的实现方式并没有提供真正的安全优势。 正如某一篇Istio文章中所述，没有机制可以确保过滤的IP与名称匹配。 这意味着可以通过简单地在HTTP请求中设置主机标头来绕过过滤。

此外，启用出口过滤的终端可能很难作为已有应用程序的起始点。或者，您可以通过设置全局参数includeIPRanges来在集群层面禁用出口过滤。 通过将其设置为集群的内部ip空间，您将绕过所有外部流量对Istio代理的访问。 一旦您在Istio中运行了服务，您就可以识别外部服务并在开启出口过滤之前建立所需的ServiceEntries。

以下Helm安装将完全禁用所有pod的出口过滤。 您应该将内部集群IP CIDR添加到includeIPRanges设置中，以通过Istio将流量从Pod路由到内部资源，同时完全绕过外部终端的访问。使用这些配置之一，您将不需要定义任何ServiceEntries即可访问外部终端。

```bash
$ helm upgrade install/kubernetes/helm/istio — install \
  --name istio — namespace istio-system \
  --set global.tag=”1.1.0.snapshot.1" \
  --set global.proxy.includeIPRanges=””
```
## Istio的设计并非完全是Kubernetes原生的
毫无疑问，Istio提供了大量资源可供Kubernetes部署。 有一种Helm图表，其中包含子图表，可用于通过相关CRD安装大量服务。 有大量的文档和示例图表配置。 尽管如此，Istio的目标仍是独立于平台。 考虑到这一点，在Kubernetes上部署支持Istio的应用程序有一些明显的不同。

影响最大的差异之一在于会对Istio网格外部暴露内部的服务。 典型的Kubernetes应用程序会使用带有控制器的入口来暴露对外部的接口。 而Istio却不是依靠网关对象(例如端口和TLS)来定义协议设置。 网关对象与虚拟服务对象配对以控制路由的详细信息。

这可能会影响您可能已经为入口对象建立的模式，例如域名和TLS证书管理。 external-dns项目最近引入了对Istio网关对象的支持，使得从入口对象的转换更加容易，从而实现了自动域名管理。

关于网关的TLS证书仍然比较复杂。 Istio的IngressGateway不支持与证书管理器兼容的多种证书。一种比较流行的方式是自动配置和安装来自Kubernetes中加密的TLS证书。 证书更新需要滚动IngressGateway pod节点，这会使事情变得更加复杂。 如果不按计划手动滚动pod节点，则很难使用简易的加密证书。 即使使用静态证书，网关也会在启动时挂载所有证书，因此为其他服务添加或删除证书也需要重新启动。 另外，由于所有的服务共享同一网关，从而允许访问其他服务的证书。

现在，启用在同一网关下具有与外部服务TLS加密的Istio的最简单的选项就是：

 1. 在某些情况下，在网格网络外部终止TLS。例如使用Amazon弹性负载均衡器并使用Amazon证书管理器管理证书。
 2. 避免使用“Let’s Encrypt”这类的动态证书提供商，并在Istio IngressGateway上安装多域或通配TLS证书。

## 找到属于你自己的方式来实现企业级Istio
现在，您已经知道了Istio基于Kubernetes之上的一些优势，您可以继续在Kubernetes集群中安装Istio。 这篇文章远远没有达到提供实际生产时的配置。 但是，这将使您少走一些弯路，从而发现自己的实际生产配置的实现方法。
