![](https://miro.medium.com/max/500/1*EA--ho-HrsVwpjqkBpopzA@2x.png)
# 用Telepresence在本地调试Kubernetes服务
### 使用Homebrew / apt / dnf安装Telepresence
您的计算机上需要以下功能：

+ `kubectl`命令行工具（这里是[安装说明](https://kubernetes.io/docs/tasks/tools/install-kubectl/)）。

+ 使用计算机上的本地凭据访问您的Kubernetes集群。您可以通过运行`kubectl get pod`来测试这一点 - 如果这个工作你都准备好了。

+ 安装[Telepresence](https://www.telepresence.io/)，请按照[此处](https://www.telepresence.io/reference/install)的说明进行操作。

### 使用Telepresence在本地调试服务

假设您有一个在临时集群中运行的服务，并且有人报告了针对它的bug。为了找出您想要在本地运行服务的问题......但是服务依赖于集群中的其他服务，也许还依赖于像数据库这样的云资源。

在本教程中，您将了解Telepresence如何允许您在本地调试服务。我们将使用`telepresence`命令行工具把在临时集群中运行的版本换成在本地计算机上运行的由你控制的调试版本。然后，Telepresence将把来自Kubernetes的流量转发到本地进程。

您应该开始`部署`并公开`服务`如下所示：


```
$ kubectl run hello-world --image=datawire/hello-world --port=8000
$ kubectl expose deployment hello-world --type=LoadBalancer --name=hello-world
```


**如果您的集群位于云端**,您可以找到生成的`服务`的地址,如下所示：


```
$ kubectl get service hello-world 
NAME          CLUSTER-IP     EXTERNAL-IP       PORT(S)          AGE 
hello-world   10.3.242.226   104.197.103.123   8000:30022/TCP   5d
```


如果您在EXTERNAL-IP下看到`<pending>`，请等待几秒钟再试一次。 在这种情况下，`服务`被公开在`http://104.197.103.123:8000/`。

**在`minikube`上，你应该这样做**来找到URL：


```
$ minikube service --url hello-world 
http://192.168.99.100:12345/
```


一旦你知道地址，你就可以存储它的值（别忘了用真实地址替换它！）：


```
$ export HELLOWORLD=http://104.197.103.13:8000
```


然后发送一个查询，它将由你的集群中运行的代码提供服务：


```
$ curl $HELLOWORLD/
Hello, world!
```


### 使用Telepresence交换部署
**重要提示：** 首次启动`Telepresence`可能需要一段时间，因为Kubernetes需要下载服务器端的镜像。

此时，您想要切换到本地开发服务，将集群上运行的版本替换为笔记本上运行的自定义版本。为了简化示例，我们将使用一个在笔记本电脑上本地运行的简单HTTP服务器：


```
$ mkdir /tmp/telepresence-test
$ cd /tmp/telepresence-test
$ echo "hello from your laptop" > file.txt
$ python3 -m http.server 8001 &
[1] 2324
$ curl http://localhost:8001/file.txt
hello from your laptop
$ kill %1
```


我们希望公开这个本地进程,以便从Kubernetes获取流量，替换现有的`hello-world`部署。

重要提示：您将要在网上公开您笔记本电脑上的Web服务器。这很酷，但也非常危险！确保当前目录中没有您不希望与整个世界分享的文件。

以下是您应该如何运行`telepresence`（您应该确保您仍然在上面创建的`/ tmp / telepresence-test`目录中）：


```
$ cd /tmp/telepresence-test
$ telepresence --swap-deployment hello-world --expose 8000 \
               --run python3 -m http.server 8000 &
```


这有三件事：

+ 启动一个类似VPN的进程，将查询发送到集群的相应DNS和IP范围。

+ `--swap-deployment`命令Telepresence将现有的`hello-world` pod替换为运行Telepresence代理的pod。 退出时，原先的pod将恢复。

+ `--run`命令Telepresence运行本地Web服务器并将其连接到网络代理。

只要您将HTTP服务器留在`telepresence`中运行，就可以从Kubernetes集群中访问它。您从这儿离开......


![](https://miro.medium.com/max/875/1*j-w7Bln7tpGs6s9tEUW1jg.png)


......到这儿：


![](https://miro.medium.com/max/875/1*P7vkn2o76emb8ccr8KdTLg.png)


我们现在可以通过我们创建的`服务`的公共地址发送查询，并且它们将访问在您的笔记本电脑上运行的Web服务器，而不是之前运行的原始代码。等待几秒钟，Telepresence代理启动; 您可以通过以下方式检查其状态：


```
$ kubectl get pod | grep hello-world
hello-world-2169952455-874dd   1/1       Running       0          1m
hello-world-3842688117-0bzzv   1/1       Terminating   0          4m
```


一旦您看到新pod处于运行状态，您就可以使用新代理连接到笔记本电脑上的Web服务器：


```
$ curl $HELLOWORLD/file.txt
hello from your laptop
```


最后，让我们在本地终止Telepresence，这样您就不必担心其他人通过将其带到后台并按下Ctrl-C来访问本地Web服务器：


```
$ fg
telepresence --swap-deployment hello-world --expose 8000 --run python3 -m http.server 8000
^C
Keyboard interrupt received, exiting.
```


现在，如果我们等待几秒钟，旧代码将被重新交换。同样，您可以通过运行来检查交换状态：


```
$ kubectl get pod | grep hello-world
```


当新pod返回`运行`状态时，您可以看到一切都恢复正常：


```
$ curl $HELLOWORLD/file.txt
Hello, world!
```


**学到了什么:** Telepresence允许您使用代理替换现有部署，该代理将流量重新路由到计算机上的本地进程。这使您可以通过在本地运行代码轻松调试问题，同时仍然允许本地进程完全访问您的临时或测试集群。

现在是时候清理服务了：


```
$ kubectl delete deployment,service hello-world
```


[Telepresence](https://www.telepresence.io/)可以做的远不止这些：有关详细信息，请参阅文档的[参考部分](https://www.telepresence.io/discussion/why-telepresence)。

**还有问题？请在我们的[Gitter聊天室](https://gitter.im/datawire/telepresence)中询问或[在GitHub上提出问题](https://github.com/telepresenceio/telepresence/issues/new)。**


原文作者：Datawire

原文地址：[Debug a Kubernetes Service Locally with Telepresence](https://articles.microservices.com/debug-a-kubernetes-service-locally-with-telepresence-675eb6e94b09)
