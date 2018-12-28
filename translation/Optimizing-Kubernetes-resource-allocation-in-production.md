在生产环境里优化Kubernetes资源分配
-----------
通过测试资源限制和负载可以为你的系统增加更多预测性和弹性。

我第一次用Kubernetes工作包括对应用的Docker容器化和部署到生产集群里。我当时正在把Buffer（译者注：一款社交媒体管理平台）的一个最高吞吐量（低风险）的端点从我们的单体应用中迁移出来。这个特别的端点带来了很多问题并且偶尔会影响到其他端点，拥有更高的流量优先级。

在经过多次手动执行`curl`命令测试后，我们决定将流量打到部署在Kubernetes的这个服务上。在1%的时候，一切看起来都正常；然后10%，仍然不错；接着是50%，这个时候服务突然开始不停地Crash。我第一反应是将我们的服务副本数扩容到20个，这个操作多少起到了些作用，服务仍然可以处理流量，但是Pod还是会Crash。通过`kubectl describe`的检查后，我发现Kubelet正在不停地杀掉这些Pods，原因是`OOMKilled`，也就是说内存用超了。经过进一步研究，我意识到我从别的Deployment拷贝和粘贴YAML时，我把内存的限制设置的太过严格。这个经历让我开始思考如何有效地设置容器资源的Requests和Limits

### Requests vs. Limits
Kubernetes允许我们对像CPU、内存和本地持久卷（v1.12的beta特性）等资源的Requests和Limits字段进行配置。像CPU这样的资源是可压缩资源，意味着这个容器受到CPU管理策略的限制。另一些资源，像内存，由Kubelet进行监控，如果它们超过了配置的限制，Kubelet就会杀掉对应的容器。使用不同的Requests和Limits配置可以对不同的任务实现不同的QoS等级。

#### Limits
Limits是该任务可以消耗的资源上限。超过这个值会触发Kubelet杀掉该Pod。如果没有设置Limits，该任务可以消耗所在节点上任意多的资源。如果运行的多种任务都没有设置Limits，那么这种资源会按照”尽最大努力”策略进行分配。

#### Requests
Requests是被调度器用来调度任务的依据。任务可以使用Requests字段所设置的所有资源而无需经过Kubernetes干涉。如果任务没有设置Limits并且使用了超过了Requests阈值的资源，容器会最少会被分配Requests所设的资源。如果只设置了Limits而没有设置Requests，Requests字段会自动与Limits字段相同。

#### Quality of Service
通过设置Limits和Requests可以为任务分配不同的QoS等级，最好的QoS等级（Pods拥有可以完成任务所需的资源，每个节点运行了尽可能多的Pod）设置取决于任务的需求。
![]()

##### Guaranteed QoS
Guaranteed等级的QoS通过设置任务资源的Limits就可以实现。这表示一个容器可以使用调度器调度参考时指明的资源数量。这种策略有利于对CPU敏感的任务和相对可预测的任务。例如一个在处理请求的Web服务器。
![]()

##### Burstable QoS
Burstable等级的QoS通过同时设置任务资源的Limits和低于Limits值的Requests进行指定。这意味着容器至少能得到Requests设置的资源，同时在节点拥有足够资源的情况下，最多不能使用超过Limits设置的资源。这对于拥有较短资源使用周期，或者那些需要密集初始化过程的任务来说很有用。比如一个构建Docker容器的worker，或者一个运行了未优化JVM进程的容器。
![]()

##### Best-Effort QoS
Best-Effort等级的QoS不需要通过设置Requests或Limits来指定。也就是说容器可以使用所在机器上的任何可用资源。从调度器的角度看这是最低优先级的任务，同时这样的任务会在拥有Burstable等级QoS和Guaranteed等级QoS的任务之前被杀掉。这种QoS等级适用于可中断的和优先级较低的应用，比如迭代运行的幂等优化进程。
![]()

### 设置Requests和Limits
给任务设置合适的Requests和Limits的关键是找到单个Pod Crash的那个点。我们可以通过使用一些不同的性能测试技术，在应用部署到生产环境前，先理解它发生Crash的各种不同情况。几乎每个应用到达极限时都有自己的失败模式。

为了准备这个测试，先确保你的副本数为一，并从保守的资源Limits开始，比如
```
# limits might look something like
replicas: 1
...
cpu: 100m # ~1/10th of a core
memory: 50Mi # 50 Mebibytes
```
注意这个过程中使用Limits可以让我们清晰地看见测试的效果（限制CPU并且当超过内存阈值时杀掉Pod）。每一轮测试结束都会改变一次资源（CPU或内存）Limits配置。

#### 加速测试（Ramp-Up Test）
加速测试随着时间增加负载，直到服务失败或者测试完成
![]()

如果加速测试突然失败，这是一个很好的迹象表示着资源Limits限制太小。如果观察到一个突变，增加两倍资源Limits直到测试成功结束。
![]()

当资源Limits接近最优（至少对Web服务来说）时，性能会随着时间可预见地降低。
![]()

如果当负载继续增加时性能不再变化，可能说明我们给这个任务分配了过多的资源。

#### 持续测试（Duration Test）
在加速测试和调整Limits之后，是时候执行一个持续测试了。持续测试是在任务失败点之前持续运行施加一段恒定的负载（至少10分钟，但是越长越好）。

![]()

这个测试的目的是为了鉴别那些在短期加速测试中不易捕获的内存泄露或者隐藏的队列技术问题。如果在这个期间进行调整，他们应该足够小（>10%改变）。一个好的结果会展示在测试时间性能是比较稳定的。
![]()

#### 记录失败日志
当执行上述测试时，记录服务的性能表现以及失败的原因是很重要的。失败模式可以记录到各种文档里，后面会为生产环境中的问题提供参照。我们测试时发现的一些失败模式：
* 内存缓慢增长
* CPU使用率100%
* 500s
* 高响应时间
* 请求丢失
* 响应时间变化很大

记下这些问题以备不时之需，兴许哪天就可以帮助团队节省很多排查时间。

### 有用的工具
可以使用Apache Bench来进行压测，使用cAdvisor实现资源使用可视化，还有一些工具更适合于资源限制测试。

#### Loader.IO
Loader.IO是一个压力测试服务。它让我们可以配置加速测试和持续测试，并可视化应用运行时的性能和负载情况，和实现测试的快速启停。它存储了历史测试结果，所以很容易根据资源限制变化来比较测试结果。
![]()

#### Kubescope CLI
Kubescope CLI是一个运行在Kubernetes上（或本地）的工具，可以从Docker(通过插件）收集和可视化容器Metrics（译作指标）。它使用类似于cAdvisor或者其他集群指标采集服务，每秒都进行采集（而不是每隔10~15秒）。如果使用10~15秒的采集间隔，你可能会错过本来测试中可以发现的瓶颈。使用cAdvisor的情况下，你不得不在每次任务资源超过阈值和Kubernetes杀掉该Pod之后寻找新Pod。Kubescope CLI直接从Docker采集指标从而修复了这个问题（或者你自行设置采集间隔），并且使用正则表达式来选择和过滤你想要可视化的Pod。
![]()

### 结论
我发现阻碍服务成为生产就绪的一个最难的问题就是你不知道它什么时候和如何失败。我希望你可以从我的错误和这些关于在你的Deployments上设置资源Limits和Requests的技术中有所收获。这可以为你的系统增加更多的弹性和可预测性，同时也会让你的顾客满意，让你能睡得更香。


### 原文链接
[原文作者: Harrison Harnisch](https://opensource.com/users/hharnisc)
[原文链接: https://opensource.com/article/18/12/optimizing-kubernetes-resource-allocation-production?utm_campaign=intrel](https://opensource.com/article/18/12/optimizing-kubernetes-resource-allocation-production?utm_campaign=intrelo)
[Getting The Most Out Of Kubernetes](https://schd.ws/hosted_files/kccna18/17/Getting%20The%20Most%20Out%20Of%20Kubernetes.pdf)
[Getting The Most Out Of Kubernetes KubeCon NA'2018](https://www.youtube.com/watch?v=NuLFomXGUj4&list=PLj6h78yzYM2PZf9eA7bhWnIh_mK1vyOfU&index=109)

