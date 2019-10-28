作者:Kurt Madel
时间:2018年7月9日
# 基于Kubernetes实现分离Jenkins代理
这是Kubernetes实现CI/CD的系列文章的第一部分。在这一部分中，我们将探索如何使用Kubernetes `Namespaces`和Kubernetes`PodNodeSelector` Admission Controller将Jenkins agent工作负载从Jenkins服务器（或主服务器）工作负载以及Kubernetes集群上的其他负载分离。在继续阅读本系列文章时，我们将了解为什么这将成为管理Jenkins agent相关功能（例如自动缩放，资源配额和安全约束）的Kubernetes配置的重要基础。

> 注意：在这个系列中使用的配置示例是部署在AWS的Kubernetes集群，使用Kubernetes功能或KOPS版本1.9并启用Kubernetes的RBAC模式（其是用于KOPS 1.9的默认授权模式）。因此，对于其他Kuberenetes平台，展示的配置的某些步骤和方面可能有所不同。

# 为什么要分离
Jenkins的最佳做法是拥有分布式工作负载，其中大部分CI / CD工作负载分布在不同的代理池中。许多CI / CD自动化工具具有与Jenkins类似的概念，可将大部分工作量分配给与CI / CD服务器或主服务器分开运行的代理; 因此，即使您不使用Jenkins来实现 CI / CD，这也很可能适用。当然，对于一个大型组织来说，拥有多个Jenkins主服务器也是一种很好的实现，但是对于任何规模的组织来说，拥有多个Jenkins代理对于长期实现CI / CD绝对至关重要。Jenkins主服务通常是一个寿命长且显示状态的应用程序——你可以将其视作宠物，而我们希望拥有动态且周期短暂的Jenkins代理程序——像牛。Kubernetes是管理您的代理服务集群的理想平台——特别是使用Jenkins Kubernetes插件，这样我们不仅有一个代理池，而且它还是一个动态的代理池，在该池中按需创建代理，并在使用完代理后销毁它们。但是，我们还是希望确保此动态代理负载不会对我们的Jenkins主服务产生负面影响。我们通过将两个工作负载分开来实现这些。Kubernetes提供了许多功能，这些功能将使我们得以实现这种分离。
# 命名空间和节点池
Kubernetes提供`Namespaces`来允许在同一集群中分离Kubernetes对象的功能。此外，许多Kubernetes解决方案都有一个概念`node pools`。一个`node pool`是一组具有相同特性的Kubernetes `nodes`——例如相同的实例类型。Kops将节点池称为InstanceGroups，对于aws，它们由aws自动缩放组（asg）支持，其中`InstanceGroups`中的所有节点除了具有其他定义的特征外，还具有相同的ec2实例类型。Kops提供了为集群创建尽可能多的节点池（或实例组）的能力，就像google kubernetes引擎（gke）一样。为了实现jenkins masters和agents之间的分离，我们将创建单独的命名空间和节点池，它们分别指定各个jenkins masters和agents来使用。这将导致所有jenkins代理`Pods`位于代理专用节点上，并与kubernetes集群上具有jenkins masters或其他工作负载的节点分离。

下图说明了Jenkins master和agent 的两个`namespaces`和两个`InstanceGroups`：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026214508409.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjk5NjU5NQ==,size_16,color_FFFFFF,t_70)
> 注意：截至本文发布之日，Azure Kubernetes服务（AKS）不支持多个节点池- 根据此GitHub，这项功能有待开发。

# Kubernetes许可控制器
为了将 Jenkins agent工作负载分离到特定的`node pool`/`InstanceGroup`(节点池/实例组)，我们转向Kubernetes Admission Controllers(许可控制器)。即使您不知道Kubernetes许可控制器是什么或做什么，但如果您曾经与Kubernetes集群进行交互，就有可能已经从中受益。例如，如果使用默认的`StorageClass`或`ServiceAccount`，而这就基于许可控制器。在这种情况下，我们将使用podnodeselector访问控制器。但是首先，我们必须为集群启用它。对于用于这个帖子中的KOPS的版本-`1.9.3`-默认情况下启用以下许可控制器（如您在kops 1.9的代码中看到的，该代码基于一组推荐的许可控制器）：
```bash
- Initializers
    - NamespaceLifecycle
    - LimitRanger
    - ServiceAccount
    - PersistentVolumeLabel
    - DefaultStorageClass
    - DefaultTolerationSeconds
    - MutatingAdmissionWebhook
    - ValidatingAdmissionWebhook
    - NodeRestriction
    - ResourceQuota
```
所以我们需要添加`PodNodeSelector`。对于**kops**，您可以使用`kops edit cluster`命令来完成添加。如果您没有为`kubeAPIServer`进行配置，则你应该首先要运行`kops get cluster --full -o=yaml`，然后可以复制该`kubeAPIServer`的配置部分，然后将其添加到现有配置中（当然，其中的某些值可能会有所不同）：
```bash
apiVersion: kops/v1alpha2
kind: Cluster
metadata:
  creationTimestamp: 2018-05-24T01:06:51Z
  name: k8s.kurtmadel.com
spec:
  api:
...
  iam:
    allowContainerRegistry: true
    legacy: false
  kubeAPIServer:
    address: 127.0.0.1
    admissionControl:
    - Initializers
    - NamespaceLifecycle
    - LimitRanger
    - ServiceAccount
    - PersistentVolumeLabel
    - DefaultStorageClass
    - DefaultTolerationSeconds
    - NodeRestriction
    - ResourceQuota
    - PodNodeSelector
    allowPrivileged: true
    anonymousAuth: false
    apiServerCount: 1
    authorizationMode: RBAC
    cloudProvider: aws
    etcdServers:
    - http://127.0.0.1:4001
    etcdServersOverrides:
    - /events#http://127.0.0.1:4002
    image: gcr.io/google_containers/kube-apiserver:v1.9.3
    insecurePort: 8080
    kubeletPreferredAddressTypes:
    - InternalIP
    - Hostname
    - ExternalIP
    logLevel: 2
    requestheaderAllowedNames:
    - aggregator
    requestheaderExtraHeaderPrefixes:
    - X-Remote-Extra-
    requestheaderGroupHeaders:
    - X-Remote-Group
    requestheaderUsernameHeaders:
    - X-Remote-User
    runtimeConfig:
      scheduling.k8s.io/v1alpha1: "true"
    securePort: 443
    serviceClusterIPRange: 100.64.0.0/13
    storageBackend: etcd2
  kubeScheduler:
  ...
```
如果您已有的配置条目，`kubeAPIServer`则只需将其添加`PodNodeSelector`到`admissionControl`列表的末尾。一旦你已经保存的更改，您将需要更新集群——使用命令`kops update cluster-yes`。如果这是一个现有的集群，那么您也必须执行滚动更新：`kops rolling-update cluster --yes`。

> 注意：如果要在新集群上启用其他许可控制器，则应在应用配置之前执行此操作，否则将需要滚动更新所有集群节点。

# 将这些完成
现在，我们已经启用了`PodNodeSelector`许可控制器，我们可以开始创建`node pools`和`namespaces`了。
## 节点池
我们将创建两个`node pools`-一个用于Jenkins master，另一个用于Jenkins agent。

Jenkins主节点池

使用`kops create ig jenkins-masters`命令为Jenkins masters创建node pool， 并添加`jenkinsType: master`作为附加的节点标签。完成这些后，看起来应该像下面这样：

```bash
apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  creationTimestamp: 2018-05-25T11:28:57Z
  labels:
    kops.k8s.io/cluster: k8s.kurtmadel.com
  name: agentnodes
spec:
  image: kope.io/k8s-1.8-debian-stretch-amd64-hvm-ebs-2018-02-08
  machineType: m5.xlarge
  maxSize: 2
  minSize: 2
  nodeLabels:
    jenkinsType: masters
    kops.k8s.io/instancegroup: jenkins-masters
  role: Node
  subnets:
  - us-east-1b
```
Jenkins代理节点池

接下来，我们将使用`kops create ig jenkins-agents`命令为Jenkins agents创建`node pool`并添加`jenkinsType: agent`作为其节点标签。完成这些后，看起来像下面这样：
```bash
apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  creationTimestamp: 2018-05-25T11:30:23Z
  labels:
    kops.k8s.io/cluster: k8s.kurtmadel.com
  name: agentnodes
spec:
  image: kope.io/k8s-1.8-debian-stretch-amd64-hvm-ebs-2018-02-08
  machineType: m5.xlarge
  maxSize: 4
  minSize: 4
  nodeLabels:
    jenkinsType: agents
    kops.k8s.io/instancegroup: jenkins-agents
  role: Node
  subnets:
  - us-east-1b
```
而且由于我们使用的是kops `InstanceGroups`，每当我们向其中任何一个节点池添加其他节点时，它将自动应用`jenkinsType`节点标签。

## 命名空间
接下来，我们将创建两个`namespaces`。然后，一个分配给Jenkins masters，另一个分配给 Jenkins agents。

首先，我们将为Jenkins masters创建一个`namespace`：
```bash
apiVersion: v1
kind: Namespace
metadata:
  annotations:
      scheduler.alpha.kubernetes.io/node-selector: jenkinsType=master
  name: jenkins-masters
```
接下来，我们将为Jenkins agents创建一个`namespace`：
```bash
apiVersion: v1
kind: Namespace
metadata:
  annotations:
      scheduler.alpha.kubernetes.io/node-selector: jenkinsType=agent
  name: jenkins-agents
```
`PodNodeSelector`许可控制器使用`scheduler.alpha.kubernetes.io/node-selector`，根据创建pod的命名空间分配默认的nodeselector。这将导致在jenkins agents名称空间中创建所有pod，这些pod位于jenkins agents节点池中的节点上，并与jenkins masters分离。

> 注意：podnodeselector还有一个基于文件的配置，它不仅允许您为特定名称空间指定默认nodeselector标签，还允许指定clusterdefaultnodeselector。但是，我还没有弄清楚如何将这种基于文件的配置用于kops。因此，如果您对此有任何想法，请在下面评论。跟踪此问题的GitHub问题。

# Jenkins Kubernetes插件配置
所以，现在，我们已经为Jenkins masters和agents 准备好了`node pools`和`namespaces`，我们需要配置Kubernetes插件来使用`jenkins-agents namespace`。刚开始，我们必须在`jenkins-agents namespace`中创建一个`ServiceAccount`，因为对新动态代理`pods`的请求将由运行在同一**Kubernetes**集群中的**Jenkins masters**发起。为了清楚地理解，我将在配置中指定`namespace`，并使用`ServiceAccount`来命名`jenkins`。

```bash
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins-masters
```

现在，我们需要创建一个Kubernetes用户，以允许根据Jenkins Kubernetes项目中提供的示例创建、使用和删除pods、执行pods中的命令以及访问jenkins代理名称空间中的pod日志。

```bash
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins-agents
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: jenkins-agents
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pods-all
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins-masters
```
上面链接中提供的示例与上面的配置之间的主要区别在于为`RoleBinding subjects`使用名称空间。

现在，您可以使用`ServiceAccount`和`jenkins-agents` `namespace`我们根据这些说明创建的为您的Jenkins主服务器配置Kubernetes插件，并确保您的分布式Jenkins代理不会对您的Jenkins主服务器或Kubernetes集群上运行的任何其他工作负载产生负面影响。

现在，您可以使用我们根据这些说明创建的`ServiceAccount`和`jenkins-agents namespace`来为你的Jenkins master配置kubernetes插件，并确保您的分布式Jenkins agents不会对jenkins masters或您在kubernetes集群上运行的任何其他工作负载产生负面影响。

在`通过Kubernetes实现CI/CD`系列的下一篇文章中，我们将探讨Jenkins agents工作负载的自动缩放。