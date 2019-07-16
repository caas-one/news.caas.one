Kubernetes vs Docker
---------------------

“Kubernetes vs.Docker” 是一个随着Kubernetes作为容器编排解决方案越来越流行而使得我们听到越来越多的短语。
 
然而，“Kubernetes vs. Docker”也是一个有一定误导性的短语。当你把它拆分开，这三个单词并不是许多人想要它们表达的意思，这是因为Docker和Kubernetes并不是直接竞争对手。

这篇文章主要是为了澄清围绕Kubernetes和Docker的一些常见的困惑，并且解释当人们谈论起“Docker vs.kubernetes”时他们真正想表达的。

### Containerization和Docker的崛起

与传统虚拟化相比，containers和container platforms有更大的优势。不需要用户操作系统就能在内核级上完成隔离，所以containers更加高效，快速和轻量。考虑到封装在独立运行环境中的应用程序有很多优点，例如更快的部署、可扩展性以及开发环境之间更接近的对等性。

Docker是目前最流行的container platform。尽管隔离环境的想法可以追溯到很久以前，而且过去也有其他的container软件，但Docker在恰当的时机出现在了市场上，并且从一开始就开放源代码，这很可能导致它目前的市场支配地位。

Docker的特点是Docker Engine,它在运行时允许你去构建和运行容器，而且还包括着一个能存储和共享图像的Docker Hub。

### 对编排系统的需求

虽然Docker为包装和分配集装箱应用程序提供了一个开放的标准，但出现了一些新的问题。如何去协调和安排所有的容器？如何使不同的容器在你的应用程序中信息互通？容器实例如何被实现？

协调容器的解决方案很快就出现了，Kubernetes,Mesos,and Docker Swarm是一些能提供抽象概念去使一组机器变得像一台大型机器的更受欢迎的选择，这在大环境中极为重要。

当大多数人说到“Kubernetes vs. Docker”时，他们真正想说的是“Kubernetes vs. Docker Swarm。”后者是Docker自己的Docker容器本地集群解决方案，其优势在于可以与Docker生态系统紧密集成，并使用自己的API。与大多数调度程序一样，Docker Swarm提供了一种管理跨服务器群集的大量容器的方法。其过滤和调度系统可以选择群集中的最佳节点来部署容器。

Kubernetes是Google开发的容器协调器，已经过度给CNCF并且现在是开源的。它具有Google多年来在容器管理方面的专业知识的优势。它是一个全面的系统，用于自动化容器化应用程序的部署、调度和扩展，并支持许多容器化工具，例如Docker。

目前，Kubernetes是市场领导者，也是编排容器和部署分布式应用程序的标准化方法，Kubernetes可以在公共云服务或本地运行，是一个高度模块化、开源、且充满活力的社区。各种规模的公司都在投资它，许多云提供商都提供Kubernetes服务。Sumo Logic为所有编排技术提供支持，包括Kubernetes支持的应用程序。

### Kubernetes和Docker有什么关系？

Kubernetes和Docker都是全面并且切合实际的解决方案，它们可以提供强大的功能智能地管理容器化应用程序并且管理因此出现的一些混乱。“Kubernetes”现在有时被用作基于Kubernetes的整个容器环境的简写。实际上，它们具有不同的根源，处理不同的事件，不能直接比较。

Docker是一个用于构建、分发和运行Docker容器的平台和工具。它提供了自己的原生集群工具，可用于在机器集群上配置和调度容器。Kubernetes是Docker容器的容器配置系统，旨在以高效的方式在生产中按比例协调节点集群，比Docker  Swarm应用的更加广泛。它围绕pod的概念工作。pod是Kubernetes生态系统中的调度单元，可以包含一个或多个容器。多个pod分布在节点之间可以提供高可用性。人们可以在Kubernetes集群上轻松运行Docker构建，但Kubernetes本身并不是一个完整的解决方案，而只是包含了一些自定义插件。

Kubernetes和Docker从根本上是不同的技术，但是它们可以很好地协同工作，而且都可以促进分布式体系结构中容器的管理和部署。

### 原文链接
[原文作者：Daisy Tsang](https://www.sumologic.com/resource/blog/author/dtsang/)
[原文链接：https://www.sumologic.com/blog/kubernetes-vs-docker/](https://www.sumologic.com/blog/kubernetes-vs-docker/)