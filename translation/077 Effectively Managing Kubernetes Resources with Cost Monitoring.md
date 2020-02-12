韦伯·布朗
2018年10月30日

通过成本监控有效管理Kubernetes资源

（2019年4月更新：我们现在启动了Kubecost项目，以为Kubernetes提供更准确的开销监控和分配。在[此处](https://medium.com/kubecost/introducing-kubecost-a-better-approach-to-kubernetes-cost-monitoring-b5450c3ae940)了解更多信息。）

我们于2014年在Google工作期间首次发现Kubernetes。我们最初的观点很大程度上取决于我们在启动内部Borg系统项目方面的经验。我们立即看到了新的开源项目的巨大潜力，但对较小的基础架构/DevOps团队可能面临的巨大复杂性感到不安。我们已经看到团队/公司努力管理这种复杂性的一个领域是有效管理集群资源，尤其是Kubernetes集群成本。结果，在获得有关资源优化的专家建议后，我们发现团队将云成本降低了100万美元以上。

在确定最佳实例大小，确定废弃的资源，确定合适的平台（GKE，EKS，AKS与其他平台）之间，要管理的影响成本的变量列表很长。采取行动优化这些参数通常始于更好地了解当前的资源使用/成本。因此，我们扩展了[Karl Stoney项目](https://karlstoney.com/2018/07/07/managing-your-costs-on-kubernetes/)，以简化监视和解释成本指标的过程。实际上，如果您已经安装了Helm，则设置过程仅需执行两个步骤。如果您已经在集群上安装了kube-state-metrics，Prometheus和Grafana，则也可以直接在[此处](https://grafana.com/grafana/dashboards/8670)和[此处](https://grafana.com/dashboards/8673)获取Grafana仪表板。

## 仪表板概述
![集群级仪表板的视图](https://img-blog.csdnimg.cn/20191117231103504.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjk5NjU5NQ==,size_16,color_FFFFFF,t_70)
											
这些仪表板旨在根据当前维持的计算、内存、网络和存储资源消耗水平，提供每月的成本估算。成本指标度量在群集，节点和名称空间各级别均可使用。集群级别的指标可以帮助识别高层成本趋势，并跟踪跨开发集群和生产集群的支出	。如果您使用不同的计算机类型运行节点池，则节点级指标可帮助您比较不同的硬件成本。最后，命名空间级别的成本度量可以帮助您在不同的应用程序和部门之间分配和比较成本。预计新的仪表板指标（例如GPU支出）即将面世。

这些仪表板已在Google Kubernetes Engine（GKE）和Amazon Kubernetes弹性容器服务（EKS）集群上使用/测试过。默认成本输入可在仪表板UI界面中轻松配置（更多信息参见下文）。该项目全部使用开源技术构建，包括kube-state-metrics，Prometheus和Grafana。
## 集群仪表板指标
主仪表板提供了集群级别的关键成本输入的视图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117231251194.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjk5NjU5NQ==,size_16,color_FFFFFF,t_70)
上面屏幕截图中的指标展示了我们经常遇到的一个场景。平均CPU使用率远低于群集容量和容器请求。通常，这是由工程团队响应达到资源请求阈值而拆分新节点引起的。此仪表板为您提供了一个起始点，通过帮助您知道在何处更仔细地分析影响集群成本的工作负载，资源约束和流量模式，从而确定如何降低成本。在这种情况下，可能的优化可能是利用[垂直pod自动缩放](https://github.com/kubernetes/enhancements/issues/21)或将一部分计算卸载到preemptible/spot实例以及其他选项。

以下是有关如何计算仪表板关键成本指标的更多详细信息。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117231322892.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjk5NjU5NQ==,size_16,color_FFFFFF,t_70)
总成本等于CPU，内存，存储和网络成本的总和。

在此仪表板上，还有许多其他更详细的指标和图形可用于管理集群资源。请按照下一部分中的步骤查看您自己的集群上的数据。

## 安装之前
 - 您需要一个Kubernetes集群，并且需要将kubectl命令行工具配置为与您的集群相通信。
- 应该安装Git。如果需要，请参阅此[安装页面](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)。

## 安装步骤
以下步骤将使用Helm安装所需的一切，以启动和运行开销仪表板。

1.从Github下载kubecost源代码并应用heml.yaml。这将创建Helm所需的ServiceAccount。
```bash
git clone https://github.com/AjayTripathy/kubecost-quickstart
 cd kubecost-quickstart 
kubectl apply -f helm.yaml
```

2.安装Helm，一个Kubernetes软件包管理器，即适合Kubernetes。使用以下命令之一完成安装。
```bash
MacOS with Homebrew:        brew install kubernetes-helm
Linux with Snap:            sudo snap install helm
Windows with Chocolatey:    choco install kubernetes-helm
```
有关其他选项，请参见[Helm安装页面](https://docs.helm.sh/using_helm/#installing-helm)，包括如何直接下载二进制文件。

3.要开始使用Helm，请运行'helm init'命令。这会为您的Kubernetes集群安装Helm服务器（Tiller），并设置您的本地配置。
```bash
helm init --service-account helm
```

4.使用Helm安装kubecost及其依赖项。
```bash
helm install cost-analyzer --name cost-analyzer --namespace monitoring
```
**注意：** EKS用户必须为您的集群和Prometheus服务器定义一个默认存储类以供使用。请参阅关于配置存储类的这篇[AWS文章](https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html)。

5.开始将Grafana端口转发到本地端口3000。

```bash
kubectl port-forward --namespace monitoring deployment/cost-analyzer-grafana 3000
```

就是这样！现在，您应该可以通过访问http://localhost:3000来查看集群的仪表板。您可能需要访问左上角的Grafana的主页选项卡，才能看到新创建的仪表板。

## 成本输入
资源价格可以根据GKE当前的定价（us-central1）使用默认值进行编辑。请注意，价格可能因地区而异，并随时间而变化。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117232229686.png)
- CPU表示按需vCPU的每月费用。该数值未包含任何折扣。
- PE CPU是preemptible/spot vCPU的每月费用。该数值未包含任何折扣。注意：需要适当的标签。
- RAM是按需GB的每月费用。该数值未包含任何折扣。
- PE RAM是preemptible/spot GB的每月费用。该数值未包含任何折扣。注意：需要适当的标签。
- Storage是1GB标准配置空间的每月费用。
- SSD是1GB SSD配置空间的每月费用。
- Egress是用于传输1GB的互联网出口速率。
- Discount表示计算和内存资源的持续使用或承诺的折扣。数值以百分比表示。

## 结论
您可以进行多种优化以降低Kubernetes的成本。我们建议或参与的大多数团队可以通过各种措施将其基础设施成本降低30％以上。监控通常是最好的起始点，以便确定不同成本削减措施的投资回报率。本文介绍了一组简单的仪表板，以使您能够直观了解关键的成本参数（CPU，内存，网络和存储）。请继续关注有关详细分析和有针对性的操作的其他文章，这些文章可以帮助您降低监控后的Kubernetes成本。
通过开销监控有效管理Kubernetes资源
韦伯·布朗
2018年10月30日

（2019年4月更新：我们现在启动了Kubecost项目，以为Kubernetes提供更准确的开销监控和分配。在此处了解更多信息。）

我们于2014年在Google工作期间首次发现Kubernetes。我们最初的观点很大程度上取决于我们在启动内部Borg系统项目方面的经验。我们立即看到了新的开源项目的巨大潜力，但对较小的基础架构/DevOps团队可能面临的巨大复杂性感到不安。我们已经看到团队/公司努力管理这种复杂性的一个领域是有效管理集群资源，尤其是Kubernetes集群成本。结果，在获得有关资源优化的专家建议后，我们发现团队将云成本降低了100万美元以上。

在确定最佳实例大小，确定废弃的资源，确定合适的平台（GKE，EKS，AKS与其他平台）之间，要管理的影响成本的变量列表很长。采取行动优化这些参数通常始于更好地了解当前的资源使用/成本。因此，我们扩展了Karl Stoney项目，以简化监视和解释成本指标的过程。实际上，如果您已经安装了Helm，则设置过程仅需执行两个步骤。如果您已经在集群上安装了kube-state-metrics，Prometheus和Grafana，则也可以直接在此处和此处获取Grafana仪表板。

仪表板概述
集群级仪表板的视图

这些仪表板旨在根据当前维持的计算、内存、网络和存储资源消耗水平，提供每月的成本估算。成本指标度量在群集，节点和名称空间各级别均可使用。集群级别的指标可以帮助识别高层成本趋势，并跟踪跨开发集群和生产集群的支出	。如果您使用不同的计算机类型运行节点池，则节点级指标可帮助您比较不同的硬件成本。最后，命名空间级别的成本度量可以帮助您在不同的应用程序和部门之间分配和比较成本。预计新的仪表板指标（例如GPU支出）即将面世。

这些仪表板已在Google Kubernetes Engine（GKE）和Amazon Kubernetes弹性容器服务（EKS）集群上使用/测试过。默认成本输入可在仪表板UI界面中轻松配置（更多信息参见下文）。该项目全部使用开源技术构建，包括kube-state-metrics，Prometheus和Grafana。

集群仪表板指标
主仪表板提供了集群级别的关键成本输入的视图。
在这里插入图片描述
上面屏幕截图中的指标展示了我们经常遇到的一个场景。平均CPU使用率远低于群集容量和容器请求。通常，这是由工程团队响应达到资源请求阈值而拆分新节点引起的。此仪表板为您提供了一个起始点，通过帮助您知道在何处更仔细地分析影响集群成本的工作负载，资源约束和流量模式，从而确定如何降低成本。在这种情况下，可能的优化可能是利用垂直pod自动缩放或将一部分计算卸载到preemptible/spot实例以及其他选项。

以下是有关如何计算仪表板关键成本指标的更多详细信息。
在这里插入图片描述
总成本等于CPU，内存，存储和网络成本的总和。

在此仪表板上，还有许多其他更详细的指标和图形可用于管理集群资源。请按照下一部分中的步骤查看您自己的集群上的数据。

安装之前
您需要一个Kubernetes集群，并且需要将kubectl命令行工具配置为与您的集群相通信。
应该安装Git。如果需要，请参阅此安装页面。
安装步骤
以下步骤将使用Helm安装所需的一切，以启动和运行开销仪表板。

1.从Github下载kubecost源代码并应用heml.yaml。这将创建Helm所需的ServiceAccount。

git clone https://github.com/AjayTripathy/kubecost-quickstart
 cd kubecost-quickstart 
kubectl apply -f helm.yaml
2.安装Helm，一个Kubernetes软件包管理器，即适合Kubernetes。使用以下命令之一完成安装。

MacOS with Homebrew:        brew install kubernetes-helm
Linux with Snap:            sudo snap install helm
Windows with Chocolatey:    choco install kubernetes-helm
有关其他选项，请参见Helm安装页面，包括如何直接下载二进制文件。

3.要开始使用Helm，请运行’helm init’命令。这会为您的Kubernetes集群安装Helm服务器（Tiller），并设置您的本地配置。

helm init --service-account helm
4.使用Helm安装kubecost及其依赖项。

helm install cost-analyzer --name cost-analyzer --namespace monitoring
注意： EKS用户必须为您的集群和Prometheus服务器定义一个默认存储类以供使用。请参阅关于配置存储类的这篇AWS文章。

5.开始将Grafana端口转发到本地端口3000。

kubectl port-forward --namespace monitoring deployment/cost-analyzer-grafana 3000
就是这样！现在，您应该可以通过访问http://localhost:3000来查看集群的仪表板。您可能需要访问左上角的Grafana的主页选项卡，才能看到新创建的仪表板。

成本输入
资源价格可以根据GKE当前的定价（us-central1）使用默认值进行编辑。请注意，价格可能因地区而异，并随时间而变化。
在这里插入图片描述

CPU表示按需vCPU的每月费用。该数值未包含任何折扣。
PE CPU是preemptible/spot vCPU的每月费用。该数值未包含任何折扣。注意：需要适当的标签。
RAM是按需GB的每月费用。该数值未包含任何折扣。
PE RAM是preemptible/spot GB的每月费用。该数值未包含任何折扣。注意：需要适当的标签。
Storage是1GB标准配置空间的每月费用。
SSD是1GB SSD配置空间的每月费用。
Egress是用于传输1GB的互联网出口速率。
Discount表示计算和内存资源的持续使用或承诺的折扣。数值以百分比表示。
结论
您可以进行多种优化以降低Kubernetes的成本。我们建议或参与的大多数团队可以通过各种措施将其基础设施成本降低30％以上。监控通常是最好的起始点，以便确定不同成本削减措施的投资回报率。本文介绍了一组简单的仪表板，以使您能够直观了解关键的成本参数（CPU，内存，网络和存储）。请继续关注有关详细分析和有针对性的操作的其他文章，这些文章可以帮助您降低监控后的Kubernetes成本。

Markdown 3998 字数 83 行数 当前行 21, 当前列 0 文章已保存21:56:47 HTML 2705 字数 49 段落