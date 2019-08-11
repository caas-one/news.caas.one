# 应用程序安全性和准确性不能被移交给Istio或任何服务网格


原文作者：Christian Posta
原文地址：[Application Safety and Correctness Cannot Be Offloaded to Istio or Any Service Mesh](https://blog.christianposta.com/microservices/application-safety-and-correctness-cannot-be-offloaded-to-istio-or-any-service-mesh/)




我最近[开始谈论](https://www.slideshare.net/ceposta/evolution-of-integration-and-microservices-patterns-with-service-mesh-107786281)有关一体化的演变和服务网格的采用，尤其是Istio。自从我在2017年1月第一次听说[Isito](https://istio.io/zh/)以来，我一直对它感到兴奋；事实上，我对[帮助组织实现了微服务和cloud-native architectures 可能性的这一新的技术浪潮](https://blog.christianposta.com/microservices/microservices-2-0/)感到兴奋。也许您能跟我聊聊，因为我已经写了许多关于它的文章(follow along for the latest @christianposta:




+ [微服务最难的部分：调用服务](https://blog.christianposta.com/microservices/the-hardest-part-of-microservices-calling-your-services/)
+ [带有Envoy Sidecar代理的微服务模式：The serise](https://blog.christianposta.com/microservices/00-microservices-patterns-with-envoy-proxy-series/)
+ http://blog.christianposta.com/microservices/application-network-functions-with-esbs-api-management-and-now-service-mesh/
+ [Envoy和Istio Circuit Breaking与Netflix OSS Hystrix的比较](https://blog.christianposta.com/microservices/comparing-envoy-and-istio-circuit-breaking-with-netflix-hystrix/)
+ [使用Istio进行Traffic Shadowing：降低代码发布的风险](https://blog.christianposta.com/microservices/traffic-shadowing-with-istio-reduce-the-risk-of-code-release/)
+ [以服务网格实现微服务的高级Traffic shadowing模式](https://blog.christianposta.com/microservices/advanced-traffic-shadowing-patterns-for-microservices-with-istio-service-mesh/)
+ [如何用服务网格来帮助微服务安全](https://blog.christianposta.com/how-a-service-mesh-can-help-with-microservices-security/)




Istio建立在容器和kubernetes的一些目标之上：提供重要的分布式系统模式作为语言不可知论的惯用词。例如，Kubernetes通过执行诸如启动/停止、健康检查、缩放/自动缩放等操作来管理容器，而不管容器中实际运行的是什么。类似地，Istio通过透明地应用到程序的容器之外，从而解决可靠性、安全性、策略和流量方面的挑战。


随着[2018年7月31日Istio 1.0的发布](https://istio.io/blog/2018/announcing-1.0/)，我们看到了Istio的使用和采用大幅度增加。我一直看到的一个问题是，“如果Istio为我提供了可靠性，我还需要在我的应用程序中担心它吗？”


答案是：abso-freakin-lutely：）

[差不多在一年前](https://blog.christianposta.com/microservices/application-network-functions-with-esbs-api-management-and-now-service-mesh/)我写过一篇文章，其中包含了这个区别，但没能足够有力的说明它；这篇文章是我帮助纠正这一点的尝试，并且这篇文章建立在[前面提到的内容基础](https://www.slideshare.net/ceposta/evolution-of-integration-and-microservices-patterns-with-service-mesh-107786281)上。



所以将之前的一些内容与现在的联系起来：Istio提供应用程序网络“可靠性”功能，如




  + 自动重试
  + 重试配额/预算
  + 连接超时
  + 请求超时
  + 客户端负载均衡
  + 掉线
  + bulkheading



  在处理分布式系统时，这些功能是必不可少的。网络不可靠，破坏了许多良好的安全性
  假设/抽象一下我们是一个整体。我们被迫要么解决这些难题，要么受到不可预知的系统范围的中断。


## 退后一步

更大的难题实际上是让应用程序相互交流以解决某些业务功能。这就是为什么我们编写软件，最终——去提供某种商业价值。该软件使用来自业务领域的结构，如“客户”、“购物车”、“帐户”等。我们[从Domain Driven Design（领域驱动设计）中看到，每个服务对这些概念的理解可能都略有不同](https://blog.christianposta.com/microservices/the-hardest-part-about-microservices-data/)。



这些不明确的概念和更大的业务限制（也就是说，客户通过姓名和电子邮件进行唯一标识，或者客户只能拥有一种类型的支票账户等），以及不可靠的网络环境和整体不可预测的基础设施，使得正确地构建非常困难。


## End to End（端到端）的正确性和安全性


然而，不可否认，在构建正确和安全的应用程序方面，这样做的责任就变成了应用程序（以及所有支持它的人）的责任。为了性能或优化，我们可以尝试在系统组件中构建较低级别的可靠性，但总体责任仍在应用程序中。1984年，Saltzer、Reed和Clark在[“End-to-End Arguments in System Design（系统设计中的端到端论据）”](http://web.mit.edu/Saltzer/www/publications/endtoend/endtoend.pdf)一文中阐述了这一原则。
  具体来讲：


>只有在通信系统端点的应用程序的知识和帮助下，才能完全并正确地实施所讨论的功能。


在这里，“功能”是指应用程序的一项要求，如“预订”或“向购物车添加项目”。这种功能不能推广到通信系统或它的组件/基础设施中（这里的“通信系统”指的是网络、中介软件以及为应用程序提供基础设施以完成其工作的任何东西）：


>因此，提供受质疑的功能作为通信系统本身的一个特征是不可能的。


但是，我们可以对通信系统做一些事情，使它的一部分可靠，并且通常有助于实现更高阶的应用需求。我们做这些事情是为了优化一个区域，这样应用程序就不必“太担心”它了，但它不是应用程序可以忽略的：


>有时，通信系统提供的功能的不完整版本可能有助于提高性能。


例如，在Saltzer的论文中，他们使用了将文件从应用程序A传输到应用程序B的示例：


![](https://blog.christianposta.com/images/end-to-end/file-transfer.png)


我们需要做什么(安全性)来确保文件得以传输，用tact（准确性）?在图中的任意点，传输都可能会失败：1）存储机制可能存在区域故障/换位/损坏，因此当应用程序A读取该文件时，它读取的是一个有故障的文件；2）应用程序可能有一个错误，将文件读取到内存中或发送出去；3）网络可能会混淆字节顺序、文件的重复部分等。我们可以进行一些优化，例如使用更可靠的传输（如TCP或message queue（消息队列）），但TCP不知道“正确传递文件”的含义，因此我们所希望的是，至少当我们把东西放到网络上时，它们将被可靠地传输。



![](https://blog.christianposta.com/images/end-to-end/tcp-reliability.png)



为了使end-to-end(端到端)全部正确，我们可能需要使用文件校验和之类的东西，它在文件初始写入时与文件一同存储，然后让应用程序B在收到文件时检验校验和。然而，我们选择检验传输是否正确地进行（实施细节），应用程序的责任在于找出解决方案并使其正确，而不是TCP或message queue（消息队列）。


## 突然出现的典型模式是什么


为了解决分布式应用程序中的应用程序正确性和安全性问题，我们可以使用突然出现的模式。在前文，我们提到了Isito提供给我们的一些可靠性模式，但这些模式并不是唯一的。通常，有两类模式会突然出现，我们可以使用它们来帮助准确、安全地构建应用程序，并且两者是相关的。我称这几类为“应用程序集成”和“应用程序网络”。它们都是应用程序的责任。让我们一起来看看：


## 应用程序集成


这些模式以以下形式出现：



+ 调用排序、多点传送和编排
+ 聚合响应、转换消息语义、拆分消息等
+ 原子性、一致性问题、saga模式
+ 反腐蚀层、适配器、边界转换
+ 信息复审、重复数据消除/幂等消息重新排序
+ 缓存
+ 消息级路由
+ 重试，超时
+ 后端/遗留系统集成



使用一个“向购物车添加项目”的简单示例，我们可以说明以下概念：



![](https://blog.christianposta.com/images/end-to-end/shopping-cart.png)



当一位用户点击“添加到购物车”，他们希望看到这个项目被添加到他们的购物车。在系统中，在我们实际调用该服务插入购物车之前，可能涉及到协调推荐引擎（hey，我们把这个加到购物车上了，想知道我们是否能计算出与之配套的推荐价。）、库存服务以及其他服务的调用/调用顺序。我们需要处理将消息转换为不同的后端，处理故障(并回滚我们发起的任何更改) ,并且在每一个服务中，我们需要能够处理重复项。如果由于某种原因，调用速度变慢，用户再次单击“添加到购物车”怎么办？即使再多可靠的基础设施可以保护我们免遭用户这样做，我们仍然需要在应用程序中检测并执行重复审查/幂等服务。


## 应用程序网络


这些模式有以下形式：


+ 自动重试
+ 重试配额/预算
+ 连接超时
+ 请求超时
+ 客户端负载均衡
+ 掉线
+ bulkheading



但在处理通过网络通信的应用程序时也会遇到其他复杂问题：




+ Canary rollout
+ 流量路由
+ 度量标准集合
+ 分布式跟踪
+ Traffic shadowing
+ 故障注入
+ 状况检查
+ 安全性
+ 机构组织政策



## 我们如何使用这些模式？


在过去，我们试图去混合这些应用程序的负责领域。我们会做一些事情，比如把所有东西都推到集中的基础设施中，这些基础设施基本上是100%可用的（应用程序网络+应用程序集成）。我们将应用程序问题放在这个集中的基础架构中（这本应该使我们更加敏捷），但在快速对应用程序进行更改时，却遇到了瓶颈和僵化。这些动态表现在我们执行企业服务总线的方式上：


![](https://blog.christianposta.com/images/end-to-end/esb-commingle.png)


另外，我相信big clouds（Netflix、Amazon、Twitter等）认识到了这些模式的“应用程序责任”方面，并将应用程序网络代码混合到应用程序中。想想Netflix OSS，我们有不同的库应对掉线、客户端负载平衡、服务发现等。



![](https://blog.christianposta.com/images/end-to-end/netflix-commingle.png)



正如你所知道的，围绕应用程序网络的Netflix OSS库是非常注重Java的。当组织开始采用Netflix OSS和Spring Cloud Netflix之类的衍生产品时，他们迎面遇到了这样一个事实，即一旦开始添加其他语言，就无法实现这样的架构。Netflix已经成熟并实现了自动化，其他组织都不是Netflix。当尝试实施应用程序库和框架时的一些问题（应用程序库和框架解决了应用程序网络问题的范围）：



+ 每种语言/框架都有自己这些关注点的实现
+ 实现方式不会完全相同；它们会变化、会不同，有时甚至是错误的。
+ 如何管理、更新和修补这些库？即生命周期管理
+ 这些库混淆了应用程序的逻辑
+ 信任开发人员能正确实现基础部分




Istio和服务网格的总体目标是解决应用网络类的问题。将这些问题的解决方案转移到服务网格是对可操作性的优化。这并不意味着它不再是应用程序所责任，它只是意味着这些功能的实现存在于进程之外，必须进行配置。


![](https://blog.christianposta.com/images/end-to-end/layers.png)




通过这样做，我们可以通过执行以下操作来优化可操作性：




+ 可以随处实现这些功能
+ 一致的功能
+ 正确的功能
+ 可由应用程序操作员和应用程序开发人员编程



Istio和服务网格不允许您将责任转移到基础设施，它们只是增加了一定程度的可靠性并优化了可操作性。就像在end-to-end（端到端）的参数中一样，TCP不允许您减轻应用程序的责任。




Istio帮助解决应用程序网络类的问题，但是应用程序集成类的问题是什么？幸运的是，对于开发人员来说，有许多框架可以帮助实现应用程序集成。我最喜欢的Java开发人员是 [Apache Camel](https://github.com/apache/camel) ，Apache Camel提供了许多编写正确和安全应用程序所需的部件，包括：



+ [调用排序、多点传送和编排](https://blog.christianposta.com/microservices/application-safety-and-correctness-cannot-be-offloaded-to-istio-or-any-service-mesh/)
+ [聚合响应、转换消息语义、拆分消息等](https://github.com/apache/camel/blob/master/camel-core/src/main/docs/eips/aggregate-eip.adoc)
+ [原子性、一致性问题、saga模式](https://github.com/apache/camel/blob/master/camel-core/src/main/docs/eips/saga-eip.adoc)
+ [反腐蚀层、适配器、边界转换](https://github.com/apache/camel/blob/master/components/readme.adoc)
+ [信息复审、重复数据消除/幂等消息重新排序](https://github.com/apache/camel/blob/master/camel-core/src/main/docs/eips/idempotentConsumer-eip.adoc)
+ 缓存
+ [消息级路由](https://github.com/apache/camel/blob/master/camel-core/src/main/docs/eips/content-based-router-eip.adoc)
+ 重试，超时
+ [后端/遗留系统集成](https://github.com/apache/camel/blob/master/components/readme.adoc)


![](https://blog.christianposta.com/images/end-to-end/layers-camel.png)


其他框架包括[Spring集成](https://spring.io/projects/spring-integration)，甚至还有一种来自WSO2的有趣的新编程语言[Ballerina](https://ballerina.io/)。记住，重复使用现有的模式和构造是非常不错的，尤其是当它们存在并且对于您选择的语言来说是成熟的时候，但是这些模式都不需要您使用框架。



## 智能终端和哑管道怎么样



对此，关于微服务，我的一个朋友提出了一个朗朗上口但却很简单的问题，关于微服务的“智能终端和哑管道”以及“使基础设施更智能”如何影响微服务的前提：


![](https://blog.christianposta.com/images/end-to-end/twitter.png)


我给出的答案是：


![](https://blog.christianposta.com/images/end-to-end/twitter2.png)


通道依旧是哑管道；我们没有通过使用服务网格将关于应用程序正确性和安全性的应用程序逻辑强制到基础结构中。我们只是让它更可靠，在操作方面优化并简化应用程序必须实现的功能（但不为此负责）。如果您不赞同或有其他想法，请随时在twitter[@christianbosta](https://twitter.com/christianposta)上发表评论或联系我们。


如果您想要进一步了解Istio，请查看 http://istio.io 或[我写的关于Istio的书](https://blog.christianposta.com/our-book-has-been-released-introducing-istio-service-mesh-for-microservices/)。
