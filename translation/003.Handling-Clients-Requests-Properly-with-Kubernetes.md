--- 
title: "Handling-Clients-Requests-Properly-with-Kubernetes" 
date: 2018-10-28T22:25:27+08:00
categories: [ "translation"]
draft: false
---
>本文摘自Marko Luksa的《Kubernetes in Actions》
>无需多言，我们都希望能够更加优雅地处理HTTP请求。显然我们不想在Pod重启或停止的时候产生那些中断的连接。从Kubernetes自身来讲它并不做任何保证。你的应用根据一些规则进行改造来组织中断的连接。这篇文章就一起讨论一下这些规则。

#### 确保所有的请求都被优雅处理
我们先从访问Pod的客户端的视角来看看Pod的生命周期（这些客户端是Pod提供的服务的消费端）。我们想确定客户端请求能够被优雅地处理，否则的话当Pod启动或停止时连接发生了中断，那我们就遇到大麻烦了。Kubernetes自身并不对此做任何保证，所以我们来看看我们需要做些什么来阻止这种情况。

#### 当Pod启动时阻止连接中断
如果你理解services和service endpoints是如何工作的的话，那么保证每个连接在Pod启动时都被优雅地处理就很简单。当一个Pod在启动时，它会作为一个endpoints被添加到所有匹配该Pod标签的Services里。Pod也会发出信号告诉Kubernetes它已就绪。只有它变成一个service endpoints时才可以接收来自客户端的请求。

如果你没有在Pod Spec里指定readiness探针，Pod会一直被认为是就绪的。这意味着当它所在的节点上Kube-Proxy第一个更新iptables规则时，它就会立即开始接收试着连接这个services的客户端的请求。如果那个时候你的应用并没有做好接收请求的准备，那么客户端将会见到“connection refused"类型的错误。

你所需要做的就是保证你的readiness探针当且仅当你的应用可以正确处理收到的请求时才返回成功结果。所以添加一个HTTP GET readiness探针并让它指向你应用的基础URL会是一个很好的开始。在很多案例里，这可以让你省去实现一个特定的readiness端点的工作量。

#### 当Pod停止时阻止连接中断
现在让我们看看当一个Pod生命周期结束时发生了什么——当Pod被删除和它里面的容器被停止时。一旦Pod的容器接收到SIGTERM后它就会开始关停（或者是在那之前先执行prestop钩子），但这是否能保证所有的客户端请求可以被优雅地处理？

我们的应用在收到结束信号时应该如何响应？它是否应该继续接收请求？那么那些已经收到的请求但是还未完成的请求呢？那么那些正在发送请求的间隔中且仍然处理打开状态的持久HTTP连接（连接上没有活跃的请求）呢？在回答这些问题之前，我们需要深入了解一下当Pod结束时集群里发生的一系列事件。

#### 理解Pod删除时发生的一系列事件
你要一直记着Kubernetes的各个组件是独立运行在集群的节点上的。它们之间并不是一个巨大的单体应用。这些组件间同步状态会花费一点时间。让我们一起来看看当Pod删除时发生了什么。

当APIserver收到一个停止Pod的请求时，它首先修改了etcd里Pod的状态，并通知关注这个删除事件所有的watcher。这些watcher里包括Kubelet和Endpoint Controller。这两个序列的事件是并行发生的（标记为A和B），如图1所示。

![图1](https://github.com/caas-one/news.caas.one/blob/master/translation/images/handling-client-request-properly-1.png)

在A系列事件里，你会看到在Kubelet收到该Pod要停止的通知以后会尽快开始停止Pod的一系列操作（执行prestop钩子，发送SIGTERM信号，等待一段时间然后如果这个容器没有自动退出的话就强行杀掉这个容器）。如果应用响应了SIGTERM并迅速停止接收请求，那么任何尝试连接它的客户端都会收到一个Connection Refusd的错误。因为APIserver是直接向Kubelet发送的请求，那么从Pod被删除开始计算，Pod用于执行这段逻辑的时间相当短。

现在，让我们一起看看另外一系列事件都发生了什么——移除这个Pod相关的iptables规则（图中所示事件系列B）。当Endpoints Controller（运行在在Kubernetes控制平面里的Controller Manager里）收到这个Pod被删除的通知，然后它会把这个Pod从所有关联到这个Pod的Service里剔除。它通过向APIserver发送一个REST请求对Endpoint对象进行修改来实现。APIserver然后通知每个监视Endpoint对象的组件。在这些组件里包含了每个工作节点上运行的Kubeproxy。这些代理负责更新它所在节点上的iptables规则，这些规则可以用来阻止外面的请求被转发到要停止的Pod上。这里有个非常重要的细节，移除这些iptables规则对于已有的连接不会有任何影响——连接到这个Pod的客户端仍然可以通过已有连接向它发送请求。

这些请求都是并行发生的。更确切地，关停应用所需要的时间要比iptables更新花费所需的时间稍短一些。这是因为iptables修改的事件链看起来稍微长一些（见图2），因为这些事件需要先到达Endpoints Controller，然后它向APIServer发送一个新请求，接着在Proxy最终修改iptables规则之前，APIserver必须通知到每个KubeProxy。这意味着SIGTERM可能会在所有节点iptables规则更新前发送。

![图2](https://github.com/caas-one/news.caas.one/blob/master/translation/images/handling-client-requests-properly-2.png)

这个结果就是Pod在收到结束信号后可能仍然会收到来自客户端的请求。如果应用立即停止接收请求，它会导致客户端收到Connection Refused类型的错误（就像在没有定义Readiness探针时，Pod启动但无法对外提供服务时一样）。

#### 解决这个问题

Google查了这个问题的解决方案，似乎给Pod添加一个Readiness探针就可以解决这个问题。按照推测，你所需要做的就是让你的Readiness探针在应用收到SIGTERM信号后尽快失败。这可以让Pod从Service上移除。只有在Readiness探针连续失败几次后才会从Service上移除（这可以在Readiness探针里进行配置）。并且显然这个移除事件会在iptables规则更新前先到达Kubeproxy。

实际上，Readiness探针在整个过程中并未起到关键作用。一旦Endpoints Controller收到Pod删除事件后（当Pod配置里的deletionTimestamp域不再为空时），它会尽快从Service上移除这些Pod。从那时起，这已经就与Readiness探测结果不相关了。

那么什么是优雅的处理方法？我们如何保证所有的请求都可以被正确处理？

那么，即使在收到终止信号后，Pod显然会继续接收请求直到所有的Kubeproxy更新完iptables规则。那么，不止是Kubeproxy。可能有一些IngressController或者其他直接向Pod转发请求的负载均衡设备等不需要经过Service的。同样还有那些使用客户端负载均衡的客户端。为了保证这些客户端都不会遇到中断的连接，你需要等待它们不会再向这个Pod发送请求的通知。

但这是不可能的，因为这些组件都分布在不同的计算机上。即使你知道它们每个所在的位置或者等着它们每个都满足停止Pod的需求，如果它们其中之一没有响应你会该怎么办？你要等待多久？记着，在那段时间里，你需要暂停关闭进程。

你唯一可能的做事情是你可以等待足够长的时间直到所有的代理都完成了它们的工作。但是多长时间才算够？在大多数情况中，短暂的几秒就足够了，但是显然这并不能满足全部的情况。当APIserver或者Endpoints Controller过载时，它们可能需要更长时间来通知到每个代理。你需要你理解你没有完美的解决方案，但是即使是5秒或者10秒都可以显著提升用户体验。你可以设置更长的延迟，但是别太过分，因为这些延迟推迟了容器的停止时间，并且会导致容器在被关停后仍然被显示在列表里，这会对用户带来一定的困扰。

优雅地关停应用包含下列的步骤：
* 等待几秒，然后停止接收新连接
* 关闭所有未在请求中的keep-alived连接
* 等到所有活跃的请求关闭，然后
* 彻底关闭

为了理解在这个过程中连接和请求的情况，请仔细检查图3.

![图3](https://github.com/caas-one/news.caas.one/blob/master/translation/images/handling-client-requests-properly-3.png)

这与现有的收到停止信号就立即关停进程不是一样的简单，对吧？这一切是值得的吗？这要靠你自己来决定。但是至少你可以添加一个prestop钩子并等待几秒，就像下面所示：

```yaml
lifecycle:
  preStop:
    exec:
      command:
      - sh
      - -c
      - "sleep 5"

```

这样，你不需要修改你应用的代码。如果你的应用早已能保证进行中的请求被正确处理，这个preStop钩子就可以满足所有你的需要。

这就是全部内容。

[handling-client-requests-properly-with-kubernetes](https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/)
