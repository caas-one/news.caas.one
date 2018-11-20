--- 
title: "Understanding-resources-limits-in-kubernetes-cpu" 
date: 2018-10-15T15:03:04+08:00
categories: [ "translation"]
draft: false
---
深入理解Kubernetes资源限制：CPU
-----------

在上一篇关于Kubernetes资源限制的文章（链接）我们讨论了如何通过ResourceRequirements给Pod里的容器设置内存限制，以及容器运行时是如何利用Linux Cgroups实现这些限制的。我也讲述了通知调度器Pod所需资源需求的requests和当宿主机遇到内存压力时帮助内核限制资源使用的limits的区别。在这篇文章中，我会继续深入探讨CPU时间的requests和limits。你是否阅读了第一篇文章并不影响本文的学习，但是我建议你两篇文章都读一读，得到工程师或者集群管理员视角的集群控制全景图。

![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/cpu.png)

### CPU时间
正如我在第一篇文章中提到的CPU时间要比内存限制更加复杂，原因如下。好消息是CPU限制也是根据我们前面所了解到的cgroups机制控制的，原理是通用的，我们只需要关注一些细节即可。我们从向前文的例子里添加CPU时间限制开始：
```yaml
resources:
  requests:
    memory: 50Mi
    cpu: 50m
  limits:
    memory: 100Mi
    cpu: 100m
```

单位后缀`m`表示“千分之一核心”，所以这个资源对象定义了容器进程需要50/1000的核心（5%），并且最多使用100/1000的核心（10%）。类似的，2000m表示2颗完整的核心，当然也可以用`2`或者`2.0`来表示。让我们创建一个只拥有CPU requests的Pod，然后看看Docker是如何配置cgroups的：
```shell
$ kubectl run limit-test --image=busybox --requests "cpu=50m" --command -- /bin/sh -c "while true; do sleep 2; done"
deployment.apps "limit-test" created
```

我们能够看到Kubernetes已经配置了50m的CPU requests：
```shell
$ kubectl get pods limit-test-5b4c495556-p2xkr -o=jsonpath='{.spec.containers[0].resources}'
map[requests:map[cpu:50m]]
```

我们也可以看到Docker配置了同样的limits:
```shell
$ docker ps | grep busy | cut -d' ' -f1
f2321226620e
$ docker inspect f2321226620e --format '{{.HostConfig.CpuShares}}'
51
```

为什么是51而不是50？CPU cgroup和Docker都把一个核心划分为1024份，而Kubernetes则划分为1000份。那么Docker如何把它应用到容器进程上？设置内存限制会让Docker来配置进程的`memory` cgroup，同样设置CPU限制会让它配置`cpu, cpuacct` cgroup。
```shell
$ ps ax | grep /bin/sh
   60554 ?      Ss     0:00 /bin/sh -c while true; do sleep 2; done
$ sudo cat /proc/60554/cgroup
...
4:cpu,cpuacct:/kubepods/burstable/pode12b33b1-db07-11e8-b1e1-42010a800070/3be263e7a8372b12d2f8f8f9b4251f110b79c2a3bb9e6857b2f1473e640e8e75
$ ls -l /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pode12b33b1-db07-11e8-b1e1-42010a800070/3be263e7a8372b12d2f8f8f9b4251f110b79c2a3bb9e6857b2f1473e640e8e75
total 0
drwxr-xr-x 2 root root 0 Oct 28 23:19 .
drwxr-xr-x 4 root root 0 Oct 28 23:19 ..
...
-rw-r--r-- 1 root root 0 Oct 28 23:19 cpu.shares
```

Docker的`HostConfig.CpuShares`容器属性映射到了cgroup的`cpu.shares`上，所以让我们看看：
```shell
$ sudo cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/podb5c03ddf-db10-11e8-b1e1-42010a800070/64b5f1b636dafe6635ddd321c5b36854a8add51931c7117025a694281fb11444/cpu.shares
51
```

你可能会惊奇地发现设置一个CPU请求会把这个值发送到cgroup去，而上篇文章中设置内存却并非如此。下面这行内核对内存软限制的行为对Kubernetes来说没什么用处，而设置了cpu.shares则是有用的。我下面会对此做出解释。那么当我们设置cpu限制时发生了什么？让我们一起找找看：
```shell
$ kubectl run limit-test --image=busybox --requests "cpu=50m" --limits "cpu=100m" --command -- /bin/sh -c "while true; do
sleep 2; done"
deployment.apps "limit-test" created
```

限制我们回过头来看看Kubernetes Pod资源对象的限制：
```shell
$ kubectl get pods limit-test-5b4fb64549-qpd4n -o=jsonpath='{.spec.containers[0].resources}'
map[limits:map[cpu:100m] requests:map[cpu:50m]]
```

在Docker容器配置里：
```shell
$ docker ps | grep busy | cut -d' ' -f1
f2321226620e
$ docker inspect 472abbce32a5 --format '{{.HostConfig.CpuShares}} {{.HostConfig.CpuQuota}} {{.HostConfig.CpuPeriod}}'
51 10000 100000
```

正如我们所见，CPU请求被存放在`HostConfig.CpuShares`属性里。CPU限制，尽管不是那么明显，它由`HostConfig.CpuPeriod`和`HostConfig.CpuQuota`两个值表示，这些Docker容器配置映射为进程的`cpu, cpuacct` cgroup的两个属性：`cpu.cfs_period_us`和`cpu.cfs_quota_us`。让我们仔细看看：
```shell
$ sudo cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pod2f1b50b6-db13-11e8-b1e1-42010a800070/f0845c65c3073e0b7b0b95ce0c1eb27f69d12b1fe2382b50096c4b59e78cdf71/cpu.cfs_period_us
100000
$ sudo cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pod2f1b50b6-db13-11e8-b1e1-42010a800070/f0845c65c3073e0b7b0b95ce0c1eb27f69d12b1fe2382b50096c4b59e78cdf71/cpu.cfs_quota_us
10000
```

如我们所料这两个配置会同样配置到Docker容器配置里。但是这些值是怎么从Pod的`100m` CPU限制里转换过来，并且是怎么实现的呢？真相是CPU requests和CPU limits是由两套不同的cgroup进行控制的。Requests使用CPU分片系统，是二者中较早的一个。Cpu分片是将每个核心划分为1024份，并且保证每个进程会接收到一定比例的CPU分片。如果只有1024片而这两个进程都设置`cpu.shares`为512，那么这两个进程会各自得到一般的CPU时间。CPU分片系统并不能指定上界，也就是说如果一个进程并没有使用它的这一份，其它进程是可以使用的。

在2010年左右Google和一些公司注意到了这个可能存在的问题（链接）。一个秒级响应和更加强大的系统被添加进去：CPU带宽控制。带宽控制系统定义了一个周期，通常是1/10秒，或者100000微秒，以及一个表示周期里一个进程可以使用的最大分片数配额。在这个例子里，我们为我们的Pod申请了`100m`CPU，它等价于100/1000的核心，或者10000/100000毫秒的CPU时间。所以我们的CPU requests被翻译为设置这个进程的`cpu,cpuacct`的配置为`cpu.cfs_period_us=100000`并且`cpu.cfs_quota_us=10000`。`cfs`表示完全公平调度，它是Linux默认的CPU调度器。同时还有一个响应quota值的实时调度器。

我们为Kubernetes设置CPU requests最终设置了`cpu.shares` cgroup属性，设置CPU limits通过配置`cpu.cfs_period_us`和`cpu.cfs_quota_us`触发了另一个不同子系统。就像内存requests对调度器一样，CPU requests会让调度器选择至少拥有那么多可用CPU分片的节点。不同于内存requests，设置CPU requests也会给cgroup设置相应的属性，帮助内核实际给进程分配一样数量的CPU核心分片。Limits的处理也与内存不一样。超出内存limits会让你的容器进程成为oom-kill的选项，但是你的进程基本上不可能超出设置的cpu配额，并且永远不会因为试着使用更多CPU而被驱逐。系统在调度器那里加强了配额的使用，所以进程在到达limits后只会被限流。

如果你并未为你的容器设置这些属性，或者给他们设置了不一定准确的值？作为内存，如果你设置了limits但并未指定requests，Kubernetes会默认让request指向limit。如果你的对你的应用需要多少CPU时间很清楚的话这是有用的。那么如果设置requests而不设置limits呢？在这个场景里Kubernetes可以精确地调度你的Pod，内核也会保证它能得到需要的最少资源配额。但是不会限制你的进程只能使用requested数量的资源，它可能会偷取别的进程的分片。不设置requests和limits是最坏的情况，调度器不知道容器需要多少资源，以及进程的CPU分片也是无限的，这可能会对节点带来不利影响。这引出了我想要说的最后一件事情：为每个namespace设置默认的的资源限制。

### 默认限制
在了解到我们前面所说的不给Pod配置资源限制的负面效应后，你可能会想到给它们设置默认值，所以每个提交到集群的Pod都会有一个默认设置。Kubernetes允许我们这么做，基于Namespace，使用v1版本的LimitRange API对象。你可以通过在你想限制的Namespace里创建LimitRange对象来简历默认限制。示例如下：
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limit
spec:
  limits:
  - default:
      memory: 100Mi
      cpu: 100m
    defaultRequest:
      memory: 50Mi
      cpu: 50m
  - max:
      memory: 512Mi
      cpu: 500m
  - min:
      memory: 50Mi
      cpu: 50m
    type: Container
```

这里的命名可能会有些迷惑，让我们把它拆分开看看。`limits`下的`default`键代表了每种资源的默认limits。在这个场景里这个Namespace里的任何没有配置内存限制的Pod都会被默认设置一个`100Mi`的限制，任何没有CPU限制的Pod会被默认设置一个`100m`的限制。`defaultRequest`键表示资源requests。如果创建了一个Pod没有指定内存requests的Pod，它会被自动分配默认的`50Mi`内存，以及如果没有指定CPU requests的话，会被默认分配`50m`的CPU。`max`和`min`键有些不同：基本上如果一个Pod的requests或limits超过了这两种规定的上下界，这个Pod就无法提交通过创建。我目前还没有找到这种用法的场景，但是你可能会用到，所以如果有的话请你留言告诉我们你用它解决了什么问题。

默认的LimitRange设置通过LimitRange插件应用到Pod上，这个插件存在于Kubernetes Admission Controller里。Admission Controller是可能会在对象被API接收之后，实际创建之前修改它的定义的插件集合。在LimitRange场景里，它会检查每个Pod，如果它没有指明requests和limits，并且Namespace设置里设置了默认值，它就会把这个默认值应用到Pod上。你会发现LimitRanger通过检查Pod metadata的annotations里设置默认值。下面是一个LimitRanger设置`100m`默认CPU requests的例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/limit-ranger: 'LimitRanger plugin set: cpu request for container
      limit-test'
  name: limit-test-859d78bc65-g6657
  namespace: default
spec:
  containers:
  - args:
    - /bin/sh
    - -c
    - while true; do sleep 2; done
    image: busybox
    imagePullPolicy: Always
    name: limit-test
    resources:
      requests:
        cpu: 100m
```

前面所说的这些合起来就构成了Kubernetes里对资源的限制。我希望这些对你有帮助。如果你对使用资源限制、默认值、Linux Cgroups或者内存管理感兴趣，我提供了一些关于这些主题更多细节的链接：

[Understanding Linux Container Scheduling](https://engineering.squarespace.com/blog/2017/understanding-linux-container-scheduling)
[Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)
[Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-memory)
[Chapter 1. Introduction to Control Groups](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/ch01)
[Configure Default Memory Requests and Limits for a Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)


### 原文链接
[原文作者: Mark Betz](https://medium.com/@betz.mark?source=post_header_lockup)
[原文链接: https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)
