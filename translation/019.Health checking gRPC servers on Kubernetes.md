# Kubernetes中的gRPC服务健康检查

作者：[Ahmet Alp Balkan](https://twitter.com/ahmetb) (Google)

gRPC正在成为云本地微服务之间通信的通用语言。如今，如果你正在Kubernetes上部署gRPC应用，你可能想知道对gRPC应用健康检查的最佳方案。在本文中，我们将讨论Kubernetes自身的方式来对gRPC应用健康检查的方式，即[grpc-health-probe](https://github.com/grpc-ecosystem/grpc-health-probe/)（gRPC健康探针）。

如果你对此不熟，这里简单介绍一下。Kubernetes健康检查（存活性和可读性探测）是用来在无人工干预的情况下，保证应用的可用性。健康检查会探测无响应pod，并将其标注为“不健康“，并使这些pod重启或重新调度。

Kubernetes本身并不支持gRPC健康检查。这使得gRPC开发人员在部署到Kubernetes时可以使用以下三种方法：

![gRPC_1](https://github.com/SheriffAlan/news.caas.one/blob/master/translation/images/gRPC_1.png?raw=true)

- **httpGet probe:** 不能和gRPC同时使用。你需要重构应用来同时为gRPC和HTTP/1.1协议同时提供服务（在不通端口）。
- **tcpSocket probe:**向gRPC服务器打开套接字没有意义，因为它不能读取响应体。
- **exec probe:**这周期性地调用容器生态系统中的程序。在gRPC的情况下，这意味着你自己实现一个健康RPC，然后编写客户机工具并随容器一起发布。

我们可以做得更好吗？答案当然是肯定的。

## 介绍 “gRPC健康探针”

要将“exec probe”方法标准化，我们需要：

- 标准的健康检查协议，供任意gRPC服务方便地实现。
- 标准的健康检查工具来方便地查询健康协议。

幸好gRPC有一个标准的健康检查协议。任意语言使用该协议都十分简便。几乎所有的gRPC语言实现都提供了生成的代码和设置健康状态的实用程序。如果在gRPC应用程序中实现此健康检查协议，那么可以使用标准/公共工具调用check()方法来确定服务状态。

接下来需要的就是“标准工具”，它就是“gRPC健康探针”。

![gRPC_2](https://github.com/SheriffAlan/news.caas.one/blob/master/translation/images/grpc_2.png?raw=true)

使用此工具，您可以在所有gRPC应用程序中使用相同的健康检查配置。这种方法有如下要求:

- 确定你最喜欢的语言中的gRPC“health”模块并开始使用(例如Go库)。
- 将grpc_health_probe二进制文件放到容器中。
- 配置Kubernetes“exec”探测来调用容器中的“grpc_health_probe”工具。

这样，执行“grpc_health_probe”将通过本地主机调用gRPC服务器，因为它们位于同一个pod中。

## Next:

grpc-health-probe项目还处于早期阶段，需要的的反馈。它支持各种特性，如与TLS服务器通信和可配置的连接/RPC超时。

如果您现在正在Kubernetes上运行gRPC服务器，请尝试使用gRPC Health协议，并在部署中尝试gRPC - Health探针，并提供反馈。