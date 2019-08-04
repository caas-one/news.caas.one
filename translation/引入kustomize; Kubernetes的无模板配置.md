### [引入kustomize; Kubernetes的无模板配置](https://kubernetes.io/blog/2018/05/29/introducing-kustomize-template-free-configuration-customization-for-kubernetes/)

**作者：** Jeff Regan（谷歌），Phil Wittrock（谷歌）

如果您需要运行Kubernetes环境，您可能已经自定义了Kubernetes的配置 - 比如复制、编辑一些API对象的YAML文件，使之满足自己的一些特定需求。

但是这种方法存在缺点 - 很难将我们的改进回溯到源材料。今天，谷歌宣布推出[**kustomize**](https://github.com/kubernetes-sigs/kustomize)，这是一个使用[SIG-CLI ](https://github.com/kubernetes/community/tree/master/sig-cli)[子项目](https://github.com/kubernetes/community/blob/master/keps/sig-cli/0008-kustomize.md)的命令行工具。该工具为个性化的配置提供了一种*声明性*方法。这种方法新颖而纯粹，它遵循我们熟悉的经过精心设计的Kubernetes API。

对于这个场景您肯定不陌生，您可以在互联网上找到关于CMS的Kubernetes配置的内容。它是一组包含Kubernetes API对象的YAML规范的文件。然后，在您自己公司的某个角落，您会为该CMS找到一个合适的数据库的配置来支持它-您常用的数据库，因为您非常了解它。

您想要以某种方式一起使用它们。此外，您希望通过自定义文件，使得资源实例显示在集群中，并给这些资源实例加上一个标签，以便区别在同一群集中执行相同操作的同事的资源。您还需要为CPU，内存和副本计数设置适当的值。

另外，您还需要拥有整个配置的*多种变体*：一个专门用于测试和实验的小型变体（在计算资源方面），以及一个用于为生产中的外部用户提供服务的更大变体。同样，其他团队也需要拥有自己的变体。

这造成了各种各样的问题。您是否将配置复制到多个位置并单独编辑？如果您有几十个开发团队需要稍微不同的堆栈变化怎么办？您如何维护和升级团队共有的配置？使用**kustomize的**工作流程可以为您解答上述这些问题。

## 定制是复用

Kubernetes配置并非代码（是API对象的YAML规范，它们更严格地被视为数据），但其生命周期与代码生命周期有许多相似之处。

您应该在版本控制中保留配置。配置所有者不一定与配置用户相同。配置可以作为更大的整体的一部分。用户希望可以为了不同目的*复用*配置。

与代码复用一样，配置复用的一种方法是简单地将其全部复制并自定义副本。与代码一样，因为与源材料的连接的切断，使得配置复用难以从源材料的持续改进中受益。在许多团队或环境中采用这种方法，而每个团队或环境又都拥有自己的配置变体，这些迥然不同的配置变体使得原本简单的升级变得十分棘手。

另一种复用方法是将源材料表示为参数化的模板。使用工具处理模板执行嵌入式脚本并使用期望值替换这些参数 - 以生成配置。复用来自于使用相同模板的不同值集。这里的难处是模板和值文件不是Kubernetes API资源的规范。它们必然是一种新东西，一种新语言，它包含了Kubernetes API。是的，它们的确很强大，但造成学习和使用工具的额外成本。而且因为不同的团队需要不同的更改 - 因此几乎所有出现在YAML文件中的规范都会成为需要值的参数。值集变大，因为必须指定所有参数（没有可信默认值）才能进行替换。

## 配置自定义的新选项

将其与**kustomize**进行比较，其中工具的行为根据被称为kustomization.yaml的文件中表示的声明性规范来确定。

该**kustomize**程序读取该文件，它引用的Kubernetes API资源文件，然后发出完整的资源到标准输出。此文本输出可以由其他工具进一步处理，也可以直接流式传输到**kubectl**以应用于群集。

例如，如果一个文件叫`kustomization.yaml` 含有

```
   commonLabels:
     app: hello
   resources:
   - deployment.yaml
   - configMap.yaml
   - service.yaml
```

是在当前工作目录中，以及它提到的三个资源文件，然后运行

```
kustomize build
```

发出包含三个给定资源的YAML流，并`app: hello`为每个资源添加一个公共标签。

同样，您可以使用*commonAnnotations*字段向所有资源添加注释，并使用*namePrefix* 字段为所有资源名称添加公共前缀。这种琐碎而常见的定制只是一个开始。

更常见的用例是，您需要一组共同资源的多种变体，例如 *开发*，*测试环境*和*生产*变体。

为此，**kustomize**支持*叠加*和*基础*的想法 。两者都由kustomization文件表示。基础声明变体共享的内容（资源和这些资源的常见自定义），并且叠加声明了差异。

这是一个文件系统布局，用于管理给定集群应用程序的*登台*和 *生产*变体：

```
   someapp/
   ├── base/
   │   ├── kustomization.yaml
   │   ├── deployment.yaml
   │   ├── configMap.yaml
   │   └── service.yaml
   └── overlays/
      ├── production/
      │   └── kustomization.yaml
      │   ├── replica_count.yaml
      └── staging/
          ├── kustomization.yaml
          └── cpu_count.yaml
```

该文件`someapp/base/kustomization.yaml`指定了这些资源的公共资源和常见自定义（例如，它们都获得了一些标签，名称前缀和注释）。

内容 `someapp/overlays/production/kustomization.yaml`可能是

```
   commonLabels:
    env: production
   bases:
   - ../../base
   patches:
   - replica_count.yaml
```

这个kustomization指定一个*补丁*文件 `replica_count.yaml`，可以是：

```
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: the-deployment
   spec:
     replicas: 100
```

补丁是部分资源声明，在这种情况下是部署的补丁 `someapp/base/deployment.yaml`，仅修改 *副本*计数以处理生产流量。

该补丁作为部分部署规范，具有明确的上下文关系和用途，即使与其余配置隔离读取，也可以进行验证。它不仅仅是一个无关上下文*{parameter name，value}*的元组。

要为生产变量创建资源，请运行

```
kustomize build someapp/overlays/production
```

运行的结果作为一组完整资源打印到stdout，准备应用于集群。类似的命令定义了登台环境。

## 综上所述

使用**kustomize**，您可以仅使用Kubernetes API资源文件管理任意数量的拥有明确定制的Kubernetes配置。**kustomize**使用的每个工件都是纯YAML，用户可以对之进行验证和处理。kustomize鼓励fork / modify / rebase [工作流](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/workflows.md)。

如果你想要立刻开始，请尝试[hello world](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/helloWorld)示例。有关讨论和反馈，请加入[邮件列表](https://groups.google.com/forum/#!forum/kustomize)或 [打开问题](https://github.com/kubernetes-sigs/kustomize/issues/new)。欢迎提交请求。
