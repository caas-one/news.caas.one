# Kubernetes Adds-ons——提供更高效的计算

我认为，“启动”一个Kubernetes集群是一项相对容易的工作。部署应用程序以在Kubernetes之上工作需要更多的努力，如果您不熟悉容器的话那就更是如此。对于使用Docker的人来说，这也是一项相对容易的工作，但是当然，你需要掌握一些类似于Hlem的新工具。然后，当你把所有的东西放在一起并且试图在生产中运行你的应用程序时，你会发现有很多丢失的部分。也许Kubernetes没什么作用，是这样吗？好吧，kubernetes是可扩展的，有一些插件或附加组件可以让你的生活更轻松。 

## 什么是Kubernetes附加组件？

简而言之，附加组件扩展了Kubernetes的功能。它们有很多，而且很可能你已经使用了一些。例如，网络插件或CNI类似于Calico、Flannel、Coredns（现在是默认的dns管理器）或著名的Kubernetes仪表板。我说“著名”是因为这可能是集群运行后第一件要尝试部署的事情:) 。上面列出的是一些核心组件，CNIS是必须具有的，同样，DNS必须具有集群功能。但是一旦开始部署应用程序，您就可以做更多的工作。输入kubernetes加载项以提高计算效率！ 

## **Cluster Autoscaler - CA.**

cluster autoscaler根据利用率缩放集群节点。如果您有待处理的pod，CA将缩放集群，如果节点利用率不高，则将缩放集群-默认设置为0.5，并可使用缩放利用率阈值进行配置。你肯定不想让pods处于挂起状态，同时，你也不想运行未充分利用的节点——这太浪费钱了！

用例：您的AWS集群中有两个实例组或自动缩放组。它们在两个可用性区域1和2中运行。您希望根据利用率来扩展集群，同时希望在这两个区域中拥有相似数量的节点。另外，您希望使用CA自动发现功能，这样就不需要定义CA中的最小和最大节点数，因为这些节点已经在自动缩放组中定义了。而您希望在主节点上部署CA。 

下面是通过helm安装CA以匹配上述用例的示例： 

```go
⚡ helm install --name autoscaler \
    --namespace kube-system \
    --set autoDiscovery.clusterName=k8s.test.akomljen.com \
    --set extraArgs.balance-similar-node-groups=true \
    --set awsRegion=eu-west-1 \
    --set rbac.create=true \
    --set rbac.pspEnabled=true \
    --set nodeSelector."node-role\.kubernetes\.io/master"="" \
    --set tolerations[0].effect=NoSchedule \
    --set tolerations[0].key=node-role.kubernetes.io/master \
    stable/cluster-autoscaler
```

您需要做一些额外的更改才能使其正常工作。有关更多详细信息，请查看本帖- [Kubernetes Cluster Autoscaling on AWS](https://akomljen.com/kubernetes-cluster-autoscaling-on-aws/)。

## **Horizontal Pod Autoscaler - HPA**

 [**Horizontal Pod Autoscaler**](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)**根据观察到的CPU利用率自动缩放复制控制器、部署或复制集中的pod数量。通过**[**自定义指标**](https://git.k8s.io/community/contributors/design-proposals/instrumentation/custom-metrics-api.md)**的支持，****还可以使用其他一些应用程序提供的指标****。** 

HPA在kubernetes的世界里并不是什么新鲜事，但是banzai cloud最近发布了[HPA Operator](https://github.com/banzaicloud/hpa-operator) ，它简化了HPA。您所需要做的就是为部署或状态集提供注释，剩下的工作由hpa来操作完成。查看[这里](https://github.com/banzaicloud/hpa-operator)支持的注释。 

使用Helm安装HPA操作符非常简单：

```go
⚡ helm repo add akomljen-charts https://raw.githubusercontent.com/komljen/helm-charts/master/charts/

⚡ helm install --name hpa \
    --namespace kube-system \
    akomljen-charts/hpa-operator

⚡ kubectl get po --selector=release=hpa -n kube-systemNAME                                  READY     STATUS    RESTARTS   AGE
hpa-hpa-operator-7c4d47dd4-9khpv      1/1       Running   0          1m
hpa-metrics-server-7766d7bc78-lnhn8   1/1       Running   0          1m
```

部署了Metrics Server后，您还可以使用kubectl top pods命令。监控CPU或Pods内存使用情况可能很有用！;)

HPA可以从一系列聚合的API中获取度量值（metrics.k8s.io、custom.metrics.k8s.io和external.metrics.k8s.io）。但是，通常情况下，HPA将使用Heapster提供的metrics.k8s.io api（在kubernetes 1.11中已弃用）或Metrics Server。

将注释添加到部署之后，您应该能够使用以下方法对其进行监视：

```go
⚡ kubectl get hpa
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGEtest-app   Deployment/test-app   0%/70%    1         4         1          10m
```

请记住，上面看到的CPU目标是基于为此特定POD定义的CPU请求，而不是节点上可用的总体CPU。

## **Addon Resizer**

[Addon resizer](https://github.com/kubernetes/autoscaler/tree/master/addon-resizer)是一个有趣的插件，可以在上面的场景中与**Metrics Server**一起使用。当您将更多的数据包部署到集群中时，最终度量服务器将需要更多的资源。Addon Resizer容器监视部署中的另一个容器（例如**Metrics Server**），并垂直地向上和向下缩放依赖容器。Addon Resizer可以基于节点数线性地缩放**Metrics Server**。有关更多详细信息，请查看官方文件。 

## **Vertical Pod Autoscaler - VPA**

您需要为将部署在kubernetes上的服务定义CPU并请求内存。如果不使用默认CPU请求，则将可用CPU的请求设置为100m或0,1。资源请求有助于kube scheduler决定运行特定pod的节点。但是，很难定义适合更多环境的“足够好”值。[**Vertical Pod Autoscaler**](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)根据POD使用的资源自动调整CPU和内存请求。它使用Metrics Server获取POD度量。请记住，您仍然需要手动定义资源限制。 

我不会在这里介绍细节，因为VPA确实需要一篇专门的博客文章，但是有一些事情你应该知道：

  · VPA仍然是一个早期项目，因此请注意 

  · 您的群集必须支持MutatingAdmissionWebhooks，这是自kubernetes 1.9以来默认启用的。 

  · 它不能与HPA一起使用

  · 当资源请求更新时，它将重新启动所有pod，这是预期的类型

## **Descheduler**

kube-scheduler是负责在kubernetes中调度的组件。但是，由于kubernetes的动态特性，有时候pods会以错误的节点结束。您可能正在编辑现有资源，以添加节点关联或（反）pod关联，或者您在某些服务器上有更多的负载，有些服务器几乎处于空闲状态。一旦pod运行kube scheduler将不再尝试重新安排。根据环境的不同，可能会有很多活动部件。 

[**Descheduler**](https://github.com/kubernetes-incubator/descheduler)**检查可以移动的pod，并根据定义的策略将其驱逐出去。**Descheduer不是默认的计划程序代替品，它取决于Descheduler。该项目目前在Kubernetes孵化器中，尚未准备好生产。但是，我发现它非常稳定，而且工作得很好。Descheller将作为cronjob在集群中运行。

我写了一篇专门的文章[Meet a Kubernetes Descheduler](https://akomljen.com/meet-a-kubernetes-descheduler/)，您可以查看更多细节。

## **k8s Spot Rescheduler**

我试图解决在AWS上管理多个自动缩放组的问题，其中一个组是按需实例，另一个组是spot实例。问题是，一旦你扩大了spot实例组，你就需要把pods从按需实例中移出，这样你才可以缩小它。[**k8s spot recheduler**](https://github.com/pusher/k8s-spot-rescheduler)**试图通过将pod驱逐到可用的点来减少点播实例的负载。** 实际上，可以使用rescheduler。从任何节点组中删除加载到另一组节点上。它们只需要贴上适当的标签。

我还创建了一个Helm图表，以便于部署：

```go
⚡ helm repo add akomljen-charts https://raw.githubusercontent.com/komljen/helm-charts/master/charts/

⚡ helm install --name spot-rescheduler \
    --namespace kube-system \
    --set image.tag=v0.2.0 \
    --set cmdOptions.delete-non-replicated-pods="true" \
    akomljen-charts/k8s-spot-rescheduler
```

有关cmdOptions检查的完整列表，请[点击此处](#flags)。

要使k8s spot recheduler正常工作，您需要标记您的节点：

  · 按需节点 - node-role.kubernetes.io/worker: "true"

  · spot 节点 - node-role.kubernetes.io/spot-worker: "true"

并在按需实例上添加prefernosschedule spot，以确保K8S spot重排程序在做出调度决策时更侧重于spot。

## **概要**

请记住，上面的一些附加组件不兼容，不能一起工作！另外，这里可能还有一些我错过的有趣的附加组件，请在评论中告诉我们。请继续关注接下来的文章。

原文作者：[ALEN KOMLJEN ](https://akomljen.com/author/alen/)

原文链接：https://akomljen.com/kubernetes-add-ons-for-more-efficient-computing/
