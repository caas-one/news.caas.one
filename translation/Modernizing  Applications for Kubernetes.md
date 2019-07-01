## 简化Kubernetes的应用程序

 

#### 介绍



现代无状态的应用程序的构建与设计是在类似于Docker这样的软件容器中运行的，并且由Kubernetes这样的容器集群管理。它们使用云原生（Cloud Native）和Twelve Factor原则和模式开发，最大限度的减少人工干预并同时提高可移植性和冗余性。将虚拟机或者基于裸机的应用程序移到容器中（将此称为“容器化”），同时将它们部署在集群里，通常需要对这些应用程序的构建，包装以及交付方式上作出重大改变。

在为Kubernetes构建应用程序的基础上，我们将在本应用指南中讨论实现应用程序现代化的高级步骤。最终的目的是在Kubernetes集群中运行和管理它们。尽管您可以在Kubernutes中运行类似于数据库这种有状态的应用程序，但本指南的重点在于迁移和现代化无状态应用程序，并将持久数据卸载到外部数据储存。Kubernetes为高效管理和扩展无状态应用程序提供了高级功能。我们将探讨在Kubernetes上运行可伸缩、可观察和可移植应用程序所需的应用程序和基础结构更改。

 

### 准备迁移应用程序

在容器化应用程序或编写kubernetes pod和部署配置文件之前，您应该实现应用程序级别的更改，以最大限度地提高应用程序在kubernetes中的可移植性和可观察性。Kubernetes是一个高度自动化的环境，可以自动部署和重新启动出现故障的应用程序容器，因此，构建适当的应用程序逻辑，以便与容器编排工具（这里只是Kubernetes）通信，并允许它根据需要自动扩展应用程序十分重要。

 

#### 提取配置数据

要实现的第一个应用程序级更改之一是从应用程序代码中提取应用程序配置。配置包含在部署和环境中不同的任何信息，如服务端点、数据库地址、凭据以及各种参数和选项。例如，如果您有两个环境，比如Staging和Production，并且每个环境都包含一个单独的数据库，那么您的应用程序不应该在代码中显式声明数据库端点和凭据，而是将它们存储在一个单独的位置，作为运行环境、本地文件或外部键值存储中的变量，从中将这些值读取到应用程序中。

将这些参数强行编码到代码中会带来安全风险，因为此配置数据通常包含敏感信息，这些信息您之后会签入版本控制系统。同时它还增加了复杂性，因为您现在必须维护应用程序的多个版本，每个版本由相同的核心应用程序逻辑组成，但配置略有不同。随着应用程序及其配置数据的增长，将配置硬编码到应用程序代码中很快变得难处理。

通过从应用程序代码中提取配置值，而不是从正在运行的环境或本地文件中摄取这些值，您的应用程序将成为一个通用的可移植包，只要您向其提供附带的配置数据，它可以部署到任何环境中。容器软件（如Docker）和集群软件（如Kubernetes）都是围绕这个例子设计的，它们构建了用于管理配置数据并将其注入应用程序容器的特性。这些特性在“Containerizing（集装箱化）”和“Kubernetes”部分中将作详细介绍。

下面将演示如何从一个简单的python flask应用程序的代码中外部化两个配置值：db_host和db_user的快速示例。我们将在应用程序的运行环境中将它们作为env vars提供，应用程序将从中读取它们：

```python
hardcoded_config.py

from flask import Flask

 

DB_HOST = 'mydb.mycloud.com'

DB_USER = 'sammy'

 

app = Flask(__name__)

 

@app.route('/')

def print_config():

    output = 'DB_HOST: {} -- DB_USER: {}'.format(DB_HOST, DB_USER)
	return output


```







运行这个简单的应用程序（参考Flask Quickstart了解如何操作），并访问它的Web端点，将显示一个包含这两个配置值的页面。

下面是与应用程序运行环境外部化的配置值相同的示例：

```go
                                env_config.py
import os

from flask import Flask

DB_HOST = os.environ.get('APP_DB_HOST')
DB_USER = os.environ.get('APP_DB_USER')

app = Flask(__name__)

@app.route('/')
def print_config():
    output = 'DB_HOST: {} -- DB_USER: {}'.format(DB_HOST, DB_USER)
    return output


```



 



在运行应用程序之前，我们在本地环境中设置了必要的配置变量：

```go
export APP_DB_HOST=mydb.mycloud.com
export APP_DB_USER=sammy
flask run
```



显示的网页应包含与第一个示例中相同的文本，但现在可以独立于应用程序代码修改应用程序的配置。可以使用类似的方法从本地文件中读取配置参数。

 在下一节中，我们将讨论如何将应用程序状态移出容器。

 

#### 卸载应用程序状态

云本地应用程序在容器中运行，并由集群软件（如Kubernetes和Docker Swarm）动态协调。给定的应用程序或服务可以跨多个副本进行负载平衡，并且任何单个应用程序容器都应该会失败，对客户机的服务最小中断或不中断。为了实现这种水平的、冗余的扩展，应用程序必须以无状态的方式设计。这意味着它们响应客户机的请求而不在本地存储持久的客户机和应用程序数据，并且在任何时间点上，如果正在运行的应用程序容器被破坏或重新启动，关键数据不会丢失。

例如，如果您正在运行通讯簿应用程序，并且您的应用程序从通讯簿中添加、删除和修改联系人，则通讯簿数据存储应该是外部数据库或其他数据存储，而且保存在容器内存中的唯一数据应该是短期的，并且是一次性的，不会造成严重的信息丢失。跨用户访问（如会话）持续存在的数据也应移动到外部数据存储（如Redis）。您应该将应用程序中的任何状态尽可能地卸载到托管数据库或缓存等服务中。 

#### 实施健康检查

在Kubernetes模型中，可以依靠集群控制平面来修复损坏的应用程序或服务。它通过检查应用程序Pod的运行状况，重新启动或重新安排不健康或无响应的容器来实现。默认情况下，如果您的应用程序容器正在运行，Kubernetes会将Pod视为“健康的”。许多情况下，这是一个正在运行的应用程序的健康状况的可靠指标。但是，如果您的应用程序锁死，并且没有执行任何有意义的工作，那么应用程序进程和容器将继续无限期地运行，并且默认情况下，kubernetes将会使停止的容器保持活动状态。

要将应用程序运行状况正确的传递给kubernetes控制平面，您应该实现定制的应用程序运行状况检查，以指示应用程序何时运行并准备接收流量。第一种类型的健康检查称为就绪探针，它让Kubernetes知道应用程序何时准备好接收流量。第二种类型的检查称为存活探测，它让Kubernetes知道应用程序何时正常运行。Kubelet节点代理可以使用3种不同的方法在运行的pods上执行这些探测：

· HTTP：Kubelet探针对端点（如/health）执行HTTP GET请求，如果响应状态在200到399之间，则成功

· 容器命令：Kubelet探针在正在运行的容器内执行命令。如果退出代码为0，则探测成功。

· TCP：Kubelet探针尝试连接到指定端口上的容器。如果它可以建立TCP连接，则探测成功。

您应该根据正在运行的应用程序、编程语言和框架选择适当的方法。准备就绪和活动性探测器都可以使用相同的探测方法并执行相同的检查，但包含就绪探测将确保POD在探测器开始成功之前不会接收流量。 

在计划和考虑将应用程序装入容器并在kubernetes上运行时，应该分配计划时间来定义特定应用程序的“健康”和“就绪”的含义，以及实现和测试端点和（或）检查命令的开发时间。

这是上面引用的Flask示例的最小健康端点：

```go
                                  env_config.py
. . .  
@app.route('/')
def print_config():
    output = 'DB_HOST: {} -- DB_USER: {}'.format(DB_HOST, DB_USER)
    return output

@app.route('/health')
def return_ok():
    return 'Ok!', 200
```





检查此路径的kubernetes liveness探针将如下所示：

```go
                                   pod_spec.yaml
. . .
  livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 2
```



 

该initialDelaySeconds字段指定Kubernetes（特别是Node Kubelet）应在等待5秒后探测/health端点，periodseconds通知kubelet每隔2秒探测/health。

要了解有关活跃度和准备情况探测的更多信息，请参阅[Kubernetes文档](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)。

#### 用于记录和监控的仪器代码

在Kubernetes这样的环境中运行容器化应用程序时，发布遥测和日志数据以监视和调试应用程序的性能是很重要的。构建功能以发布性能指标，例如响应持续时间和错误率，将帮助您监视应用程序，并在应用程序不正常时向您发出警报。

[Prometheus](https://prometheus.io/)是一个可以用来监视服务的工具，一个开源系统监控和警报工具包，由CNCF托管。Prometheus提供了几个客户端库，用于使用各种度量类型对代码进行检测，以计算事件及其持续时间。例如，如果您使用flask python框架，则可以使用prometheus [python客户端](https://github.com/prometheus/client_python)向请求处理函数添加装饰器，以跟踪处理请求所花费的时间。然后，普罗米修斯可以在HTTP端点（如/metrics）上删除这些指标。

RED方法是设计应用程序仪器时使用的一种有效方法。它由以下三个关键请求指标组成：

​    · 速率：您的应用程序收到的请求数

​    · 错误：应用程序发出的错误数

​    · 持续时间：应用程序提供响应所需的时间

这个最小的度量标准应该为您提供足够的数据，以便在应用程序性能下降时发出警报。实现此工具以及上面讨论的健康检查将允许您快速检测出故障应用程序并从中恢复。

要了解有关监控应用程序时要测量的信号的更多信息，请参阅Google站点可靠性工程手册中的[监控分布式系统](#xref_monitoring_golden-signals)。

除了考虑和设计用于发布遥测数据的功能外，您还应该计划应用程序如何登录到基于分布式集群的环境中。理想情况下，您应该删除对本地日志文件和日志目录的硬编码配置引用，而不是直接登录到stdout和stderr。您应该将日志视为连续的事件流或时间顺序事件序列。然后，这个输出流将被封装应用程序的容器捕获，从中它可以被转发到日志记录层，如EFK（ElasticSearch、Fluentd和Kibana）堆栈。Kubernetes在设计日志体系结构时提供了很大的灵活性，我们将在下面详细探讨。

#### 将管理逻辑构建到API中

一旦您的应用程序被容器化，并在像kubernetes这样的集群环境中运行，您可能就不能再使用shell访问运行您的应用程序的容器了。如果您已经实现了足够的运行状况检查、日志记录和监视，那么您可以很快收到警报，并调试生产问题，但是除了重新启动和重新部署容器之外，采取措施可能很困难。对于快速的操作和维护修复，如刷新队列或清除缓存，您应该实现适当的API端点，这样您就可以执行这些操作，而无需重新启动容器或exec到正在运行的容器中并执行一系列命令。容器应被视为不可变的对象，并且应避免在生产环境中手动管理。如果必须执行一次性管理任务（如清除缓存），则应通过API公开此功能。 

#### 摘要

在这些部分中，我们讨论了您可能希望在将应用程序装入容器并将其移动到Kubernetes之前实现的应用程序级更改。有关构建云本地应用程序的更深入的演练，请参考Kubernetes的“[架构应用程序](https://www.digitalocean.com/community/tutorials/architecting-applications-for-kubernetes)”。              

我们现在将讨论在为应用程序构建容器时要记住的一些注意事项。

### 容纳您的应用程序 

现在，您已经实现了应用程序逻辑，以便在基于云的环境中最大限度地提高其可移植性和可观察性，现在是将应用程序打包到容器中的时候了。在本指南中，我们将使用Docker容器，但是您应该使用最适合您生产需要的容器实现。

#### 明确声明依赖关系

在为应用程序创建dockerfile之前，第一步是评估应用程序正确运行所需的软件和操作系统依赖关系。Dockerfiles允许您显式地版本化安装到image中的每一个软件，您应该通过明确声明父映像、软件库和编程语言版本来利用这个特性。

尽可能避免使用最新的标签和未版本化的软件包，因为它们可能会移动，从而可能破坏您的应用程序。您可能希望创建公共Registry的私有Registry或私有image，以对image版本控制施加更多控制，并防止上游更改无意中破坏您的image生成。

要了解有关设置私有 image registry的更多信息，请参阅Docker官方文档和下面的 Registries部分中的[部署 Registry服务器](https://docs.docker.com/registry/deploying/)。 

#### 保持图像尺寸尽量小

在部署和提取容器image时，大型image会显著降低速度并增加带宽成本。将一组最小的工具和应用程序文件打包到一个映像中有几个好处： 

​    · 缩小图像尺寸

​    · 加快image构建速度

​    · 减少容器开始滞后

​    · 加快image传输时间

​    · 通过减少攻击面来提高安全性

构建image时可以考虑的一些步骤：

​    · 使用最小的基本操作系统image，例如alpine，scratch而不是像ubunte这样的全功能操作系统

​    · 安装软件后清理不必要的文件和工件

​    · 使用单独的“构建”和“运行时”容器来保持生产应用程序容器的小型化

​    · 在大型目录中复制时，忽略不必要的构建工件和文件

有关优化Docker容器的完整指南，包括许多示例，请参阅“[为Kubernetes构建优化容器。 ](https://www.digitalocean.com/community/tutorials/building-optimized-containers-for-kubernetes)”

#### 注入配置

Docker提供了几个有用的特性，可以将配置数据注入到应用程序的运行环境中。              执行此操作一个选项是使用env语句在Dockerfile中指定环境变量及其值，以便配置数据内置到图像中：

```go
Dockerfile

...

ENV MYSQL_USER=my_db_user

...
```



然后，您的应用程序可以从其运行环境中分析这些值，并相应地配置其设置。              

当使用docker run和-e标志启动容器时，还可以将环境变量作为参数传入：

```go
docker run -e MYSQL_USER='my_db_user' IMAGE[:TAG]
```



最后，可以使用env文件，其中包含环境变量及其值的列表。为此，请创建该文件并使用--env file参数将其传递给命令：

```go
docker run --env-file var_list IMAGE[:TAG]
```



如果您正在使用类似kubernetes的集群管理器对应用程序进行现代化以运行它，那么您应该进一步从映像中外部化配置，并使用kubernetes的内置[configmap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)和[secrets](https://kubernetes.io/docs/concepts/configuration/secret/)对象管理配置。这允许您将配置与映像清单分开，以便您可以从应用程序单独管理和版本它。要了解如何使用configmap和secrets外部化配置，请参阅下面的configmap和secrets部分。

#### 将图像发布到注册表

一旦您构建了应用程序映像，为了使它们对Kubernetes可用，您应该将它们上载到容器映像注册表中。像[docker hub](https://hub.docker.com/)这样的公共注册中心为[node.js](https://hub.docker.com/_/node/)和[nginx](https://hub.docker.com/_/nginx/)等流行的开源项目托管最新的docker映像。私人注册中心允许您发布内部应用程序映像，使它们对开发人员和基础设施可用，但不适用于更广泛的领域。

您可以使用现有的基础设施（例如，在云对象存储之上）部署私有注册表，也可以选择使用[quay.io](https://quay.io/)或付费Docker Hub计划等多种Docker注册表产品之一。这些注册表可以与托管版本控制服务（如Github）集成，以便在更新和推送Dockerfile时，注册表服务将自动拉入新的Dockerfile，构建容器映像，并使更新后的映像对您的服务可用。 

要对容器图像及其标记和发布的构建和测试施加更多控制，可以实现持续集成（CI）管道。

 

#### 实现构建管道

手动构建、测试、发布和将您的图像部署到生产环境中可能容易出错，并且不能很好地扩展。要管理生成并持续发布包含对映像注册表的最新代码更改的容器，应使用生成管道。              

大多数构建管道执行以下核心功能：

​    · 观察源代码存储库的变化

​    · 对修改后的代码运行冒烟和单元测试

​    · 构建包含修改代码的容器图像

​    · 使用构建的容器映像运行进一步的集成测

​    · 如果测试通过，则将图像标记并发布到注册表

​    · （可选，在持续部署设置中）更新Kubernetes部署并将映像部署到登台/生产集群

有许多付费的持续集成产品与流行的版本控制服务（如GitHub）和图像注册（如Docker Hub）进行了内置集成。这些产品的一个替代品是[Jenkins](https://jenkins.io/zh/)，一个免费的开源构建自动化服务器，可以配置为执行上面描述的所有功能。要了解如何设置Jenkins持续集成管道，请咨询[如何在Ubuntu 16.04上的Jenkins中设置持续集成管道。 ](https://www.digitalocean.com/community/tutorials/how-to-set-up-continuous-integration-pipelines-in-jenkins-on-ubuntu-16-04)

#### 实施容器记录和监视

  在处理容器时，考虑将用于管理和存储所有正在运行和停止的容器的日志的日志基础结构非常重要。有多个容器级别的模式可以用于日志记录，还可以使用多个kubernetes级别的模式。

在kubernetes中，默认情况下，容器使用json文件docker[日志记录驱动程序](https://docs.docker.com/config/containers/logging/configure/)，该驱动程序捕获stdout和stderr流，并将其写入运行容器的节点上的json文件。有时直接登录到stderr和stdout可能不足以满足您的应用程序容器，您可能需要将应用程序容器与kubernetes pod中的logging sidecar容器配对。然后，这个SideCar容器可以从文件系统、本地套接字或SystemD日志中提取日志，这比简单使用stderr和stdout流更为灵活。此容器还可以进行一些处理，然后将增强的日志流式传输到stdout/stderr，或者直接传输到日志后端。要了解更多关于kubernetes日志记录模式的信息，请参考本教程的kubernetes日志记录和监视部分。

应用程序如何在容器级别登录取决于其复杂性。对于简单的、单用途的微服务，建议直接登录到stdout/stderr并让kubernetes拾取这些流，因为您可以利用kubectl logs_命令从kubernetes部署的容器访问日志流

 

与日志记录类似，您应该开始考虑在基于容器和集群的环境中进行监视。docker提供了有用的docker stats命令，用于获取在主机上运行容器的CPU和内存使用等标准度量，并通过[Remote REST API](https://docs.docker.com/develop/sdk/)公开更多度量。此外，开源工具[cadvisor](https://github.com/google/cadvisor)（默认安装在kubernetes节点上）提供了更高级的功能，如历史度量收集、度量数据导出和用于排序数据的有用Web UI。

 

但是，在多节点、多容器的生产环境中，更复杂的度量堆栈（如[prometheus](https://prometheus.io/)和[grafana](https://grafana.com/)）可能有助于组织和监控容器的性能数据。

 

#### 总结

在这些部分中，我们简要讨论了一些构建容器、设置CI/CD管道和图像注册表的最佳实践，以及一些提高容器可观测性的注意事项。

​     · 要了解有关优化Kubernetes容器的更多信息，请参阅为Kubernetes [构建优化容器](https://www.digitalocean.com/community/tutorials/building-optimized-containers-for-kubernetes)。

​    · 要了解更多关于CI / CD，请参考[持续集成，交付和部署的介绍](https://www.digitalocean.com/community/tutorials/an-introduction-to-continuous-integration-delivery-and-deployment)和[对CI / CD最佳做法的介绍](https://www.digitalocean.com/community/tutorials/an-introduction-to-ci-cd-best-practices)。

在下一节中，我们将探讨Kubernetes的特性，这些特性允许您在集群中运行和扩展容器化应用程序。 

 

## **在Kubernetes上部署**

此时，您已经将应用程序装箱并实现了逻辑，以最大限度地提高其在云本地环境中的可移植性和可观察性。现在我们将探讨Kubernetes的特性，这些特性提供了简单的界面来管理和扩展Kubernetes集群中的应用程序。

 

#### 编写部署和POD配置文件

 在将应用程序装入容器并发布到注册表中，就可以使用pod工作负载将其部署到kubernetes集群中。Kubernetes集群中最小的可部署单元不是容器而是POD。pod通常由应用程序容器（如容器化的flask web应用程序）或应用程序容器以及执行一些辅助功能（如监视或日志记录）的任何“sidecar”容器组成。pod中的容器共享存储资源、网络命名空间和端口空间。它们可以使用本地主机相互通信，并且可以使用装载的卷共享数据。另外，pod工作负载允许您在主应用程序容器开始运行之前定义运行安装脚本或实用程序的[Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)。

 

Pod通常使用Deployments推出，Deployments是由声明特定所需状态的YAML文件定义的控制器。例如，应用程序状态可以运行flask web应用程序容器的三个副本，并公开端口8080。一旦创建，控制平面会根据需要将容器调度到节点上，从而逐渐使集群的实际状态与部署中声明的所需状态相匹配。要将集群中运行的应用程序副本数量从3个扩展到5个，请更新部署配置文件的replicas字段，然后kubectl apply新配置文件。使用这些配置文件，可以使用现有的源代码管理服务和集成来跟踪和版本化缩放和部署操作。

 

以下是Flask应用程序的示例Kubernetes部署配置文件：

```go
                                  flask_deployment.yaml

apiVersion: apps/v1

kind: Deployment

metadata:

  name: flask-app

  labels:

     app: flask-app

spec:

  replicas: 3

  selector:

     matchLabels:

       app: flask-app

  template:

     metadata:

       labels:

         app: flask-app

     spec:

       containers:

       \- name: flask

         image: sammy/flask_app:1.0

         ports:

         \- containerPort: 8080
```



 

此部署使用sammy/flask AppImage（版本1.0），打开端口8080，启动运行名为flask的容器的3个POD。这种部署称为flask应用程序。

要了解有关Kubernetes Pods和Deployments的更多信息，请参阅官方Kubernetes文档的[Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 和 [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)部分。

#### 配置Pod存储

Kubernetes使用卷、持久卷（pvs）和持久卷声明（pvc）管理POD存储。卷是用于管理pod存储的kubernetes抽象，支持大多数云提供程序块存储产品，以及承载运行pod的节点上的本地存储。要查看支持的卷类型的完整列表，请参阅Kubernetes[文档](#types-of-volumes)。 

 

例如，如果您的pod包含两个需要在它们之间共享数据的nginx容器（第一个容器称为nginx为网页提供服务，第二个容器称为nginx sync从外部位置获取页面并更新nginx容器提供的页面），那么您的pod规范应该是这样的（这里我们使用emptydir体积类型）： 

```go
                                  pod_volume.yaml

apiVersion: v1

kind: Pod

metadata:

  name: nginx

spec:

  containers:

  \- name: nginx

     image: nginx

     volumeMounts:

     \- name: nginx-web

       mountPath: /usr/share/nginx/html

  \- name: nginx-sync

     image: nginx-sync

     volumeMounts:

     \- name: nginx-web

       mountPath: /web-data

  volumes:

  \- name: nginx-web

     emptyDir: {}
```



 

 

我们对每个容器使用volume mount，表示我们希望将nginx web卷/usr/share/nginx/html装入 nginx容器中，以及nginx同步容器中的/web数据。我们还定义了一个卷称为emptydir类型的nginx web。 

 

以类似的方式，您可以使用云块存储产品配置Pod存储，方法是将volume类型修改emptyDir为相关的云存储卷类型。

卷的生命周期与pod的生命周期有关，但与容器的生命周期无关。如果一个pod中的容器死了，那么这个卷将继续存在，并且新启动的容器将能够装载相同的卷并访问它的数据。当一个pod重新启动或停止运行时，它的卷也会重新启动或停止运行，尽管这些卷由云块存储组成，但是它们会被卸载，并且将来的pod仍然可以访问数据。

 

要在Pod重新启动和更新之间保留数据，必须使用PersistentVolume（PV）和PersistentVolumeClaim（PVC）对象。

 

Persistentvolumes表示持久存储片段的抽象，如云块存储卷或NFS存储。它们是独立于PersistentVolumeClaims创建的，PersistentVolumeClaims是开发人员对存储块的需求。在pod配置中，开发人员使用pvc请求持久存储，kubernetes与可用的pv卷匹配（如果使用云块存储，kubernetes可以在创建persistentvolumeclaims时动态创建persistentvolume）。

 

如果应用程序需要每个副本一个持久卷（许多数据库都是这样），则不应使用Deployments，而应使用Statefulset控制器，该控制器专为需要稳定网络标识符、稳定持久存储和订购保证的应用程序而设计。Deployments应用于无状态应用程序，如果定义了用于部署配置的PersistentVolume目标，则该pvc将由所有部署的副本共享。 

 

要了解有关Statefulset控制器的更多信息，请参阅Kubernetes[文档](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)。要了解有关Persistentvolume和Persistentvolume声明的更多信息，请参阅Kubernetes存储[文档](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)。 

 

#### 使用Kubernetes注入配置数据

 

 

与Docker类似，kubernetes提供了env和envfrom字段，用于在pod配置文件中设置环境变量。下面是pod配置文件中的一个示例片段，它将运行pod中的hostname环境变量设置为my hostname：

 

```go
                                     sample_pod.yaml

...

     spec:

       containers:

       \- name: nginx

         image: nginx:1.7.9

         ports:

         \- containerPort: 80

         env:

         \- name: HOSTNAME

           value: my_hostname

...
```



 

这允许您将配置移出dockerfiles，并移到pod和部署配置文件中。进一步从DockerFiles外部化配置的一个关键优势是，现在您可以从应用程序容器定义中分别修改这些kubernetes工作负载配置（例如，通过将hostname值更改为my_hostname_2）。修改pod配置文件后，可以使用其新环境重新部署pod，而底层容器映像（通过其dockerfile定义）不需要重新构建、测试并推送到存储库。您还可以将这些pod和部署配置与dockerfiles分开进行版本设置，从而使您能够快速检测破坏性的更改，并进一步将配置问题与应用程序错误分开。

Kubernetes提供了另一种构造，用于进一步外化和管理配置数据：ConfigMaps和Secrets。



#### ConfigMaps和Secrets。



configmap允许您将配置数据保存为对象，然后在pod和部署配置文件中引用这些对象，这样您就可以避免硬编码配置数据，并在pod和部署之间重用它。              

 

下面是一个例子，使用上面的pod配置。我们首先将hostname环境变量保存为configmap，然后在pod config中引用它：

 

```go
kubectl create configmap hostname --from-literal=HOSTNAME=my_host_name
```



 

要从pod配置文件引用它，我们使用valuefrom和configmapkeyref构造：

```go

sample_pod_configmap.yaml

...

     spec:

       containers:

       \- name: nginx

         image: nginx:1.7.9

         ports:

         \- containerPort: 80

         env:

         \- name: HOSTNAME

           valueFrom:

             configMapKeyRef:

               name: hostname

               key: HOSTNAME

...
```



 

因此，hostname环境变量的值已从配置文件中完全外部化。然后，我们可以在所有部署和引用它们的pods中更新这些变量，并重新启动pods以使更改生效。 

 

如果您的应用程序使用配置文件，configmap还允许您将这些文件存储为configmap对象（使用--来自文件标志），然后可以将其作为配置文件装入容器中。

 

Secrets提供与ConfigMaps相同的基本功能，但应该用于敏感数据，如数据库凭证，因为值是base64编码的。

要了解有关ConfigMaps和Secrets的更多信息，请参阅Kubernetes [文档](https://kubernetes.io/docs/concepts/configuration/)。

 

#### 创建服务

一旦您的应用程序在kubernetes中启动并运行，每个pod将被分配一个（内部）IP地址，由其容器共享。如果这些pods中的一个被删除或消失，新启动的pods将被分配不同的IP地址。

 

对于向内部和（或）外部客户机公开功能的长时间运行的服务，您可能希望对执行相同功能（或部署）的一组pods授予一个稳定的IP地址，该地址在其容器中负载平衡请求。您可以使用kubernetes服务来完成这项工作。

 

Kubernetes服务有4种类型，由服务配置文件中的类型字段指定：

 

​    · ClusterIP：这是默认类型，它为服务提供可从群集内的任何位置访问的稳定内部IP。

​    · NodePort：将在静态端口上的每个节点上公开您的服务，默认情况下在30000-32767之间。当请求以其节点IP地址和NodePort服务命中节点时，请求将进行负载平衡并路由到您的服务的应用程序容器。

​    · LoadBalancer：这将使用云提供商的负载平衡产品创建负载均衡器，NodePort并ClusterIP为将要路由外部请求的服务进行配置。

​    · ExternalName：此服务类型允许您将Kubernetes服务映射到DNS记录。它可以用于使用Kubernetes DNS从Pod访问外部服务。

请注意，为集群中运行的每个部署创建类型为LoadBalancer的服务将为每个服务创建一个新的云负载平衡器，这可能会变得昂贵。要使用单个负载平衡器管理将外部请求路由到多个服务，可以使用入口控制器。入口控制器超出了本文的范围，但是要了解更多关于它们的信息，您可以参考Kubernetes[文档](https://kubernetes.io/docs/concepts/services-networking/ingress/)。[nginx入口控制器](https://github.com/kubernetes/ingress-nginx)是一个流行的简单入口控制器。              

下面是本指南的pods和deployments部分中使用的Pod示例的简单服务配置文件： 

```go
                                  flask_app_svc.yaml

apiVersion: v1

kind: Service

metadata:

  name: flask-svc

spec:

  ports:

  \- port: 80

     targetPort: 8080

  selector:

     app: flask-app

  type: LoadBalancer
```



 

 

在这里，我们选择使用这个flask svc服务公开flask应用程序部署。我们创建了一个云负载均衡器，将流量从负载均衡器端口80路由到公开的容器端口8080。              

 

要了解更多关于kubernetes服务的信息，请参阅kubernetes[文档](https://kubernetes.io/docs/concepts/services-networking/service/)的服务部分。

 

 

#### 记录和监控 

 

使用kubectl logs和docker logs通过单个容器和pod日志进行解析会随着正在运行的应用程序数量的增加而变得单调乏味。为了帮助您调试应用程序或集群问题，您应该实现集中的日志记录。从高层来看，这包括运行在所有工作节点上的代理，这些工作节点处理pod日志文件和流，用元数据丰富它们，并将日志转发到后端，如elasticsearch。从那里，可以使用可视化工具（如kibana）对日志数据进行可视化、过滤和组织。 

 

在容器级日志记录部分，我们讨论了将容器中的应用程序记录到stdout/stderr流的推荐kubernetes方法。我们还简要讨论了在从应用程序登录时可以为您提供更大灵活性的SideCar容器的日志记录。您还可以直接在pods中运行日志记录代理，这些pods捕获本地日志数据并将其直接转发到日志记录后端。每种方法都有其优点和缺点，以及资源利用的权衡（例如，在每个pod内运行一个日志代理容器可能会变为资源密集型，并迅速压倒日志后端）。要进一步了解不同的日志体系结构及其权衡，请参考Kubernetes[文档](https://kubernetes.io/docs/concepts/cluster-administration/logging/)。

 

在标准设置中，每个节点都运行一个日志记录代理，如[filebeat](https://www.elastic.co/cn/products/beats/filebeat)或[fluentd](https://github.com/fluent/fluentd)，它收集由kubernetes创建的容器日志。回想一下，kubernetes为节点上的容器创建JSON日志文件（在大多数安装中，这些文件可以在/var/lib/docker/containers/找到）。这些应该使用类似logrotate的工具进行旋转。Node日志记录代理程序应作为[DaemonSet Controller](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)运行，这是一种Kubernetes工作负载，确保每个节点都运行守护程序pod的副本。在这种情况下，pod将包含日志代理及其配置，它处理装载到日志守护进程pod中的文件和目录中的日志。 

 

与使用kubectl日志调试容器问题的瓶颈类似，最终您可能需要考虑一个比简单使用kubectl top和kubernetes仪表板来监控集群上pod资源使用情况更强大的选项。可以使用Prometheus监控系统和时间序列数据库以及Grafana度量仪表盘设置集群和应用程序级监控。[Prometheus](https://prometheus.io/)使用“拉”模型，定期为度量数据刮取HTTP端点（如节点上的/metrics/cadvisor，或/metrics application rest api端点），然后对其进行处理和存储。然后可以使用Gravana仪表板分析和可视化这些数据。Prometheus和[Grafana](https://github.com/grafana/grafana)可以像其他任何部署和服务一样，进入Kubernetes集群。

 

为了增加弹性，您可能希望在单独的Kubernetes集群上运行日志记录和监视基础结构，或使用外部日志记录和度量标准服务。

 

 

### 结论

 

迁移和现代化应用程序，使其能够在kubernetes集群中高效运行，通常涉及大量软件和基础设施更改的规划和架构。一旦实现这些更改，服务所有者就可以继续部署他们的应用程序的新版本，并在必要时轻松地对其进行扩展，只需最少的手动干预。从应用程序中外部化配置、设置正确的日志记录和度量发布以及配置健康检查等步骤允许您充分利用Kubernetes设计的云原生典范。通过构建可移植容器并使用Kubernetes对象（如部署和服务）管理它们，您可以充分利用可用的计算基础设施和开发资源。



原文链接：https://www.digitalocean.com/community/tutorials/modernizing-applications-for-kubernetes

原文作者：Hanif Jetha         https://www.digitalocean.com/community/users/hjet
