# Jaeger在Kubernetes上的分布式追踪基础设施
![](https://miro.medium.com/max/875/1*MS3u5zT23BzNNyD_74shMw.jpeg)
人们不能夸大监控基础设施作为分布式系统(或任何系统)的一个组成部分的重要性。监控不仅要追踪二进制的“向上”和“向下”模式，而且要涉及复杂的系统行为。可以设置监控基础设施，以了解性能、系统健康状况以及它随时间变化的行为模式。本文介绍了监控基础设施的一个方面——分布式追踪。

#### 微服务体系结构中的可观测性
Kubernetes已经成为实际意义上的微服务基础设施和部署的协调器。它的生态系统非常丰富，是开源社区中增长最快的一个。Prometheus，ElasticSearch，Grafana，Envoy / Consul，Jaeger / Zipkin的监控基础架构为在整个堆栈中启用指标，日志记录，仪表板，服务发现和分布式跟踪奠定了坚实的基础。

#### 分布式跟踪
分布式跟踪能够捕获请求并构建从用户请求到数百个服务之间交互的整个调用链的视图。它还可以检测应用程序延迟（每个请求需要多长时间），追踪网络调用的生命周期（HTTP、RPC等），还可以通过了解bottlenecks来发现性能问题。

以下部分介绍了在Kubernetes设置中使用Jaeger对gRPC服务进行分布式追踪。Jaeger Github org专门为Kubernetes中Jaeger的各种部署配置提供repo。这些都是很好的示例，我将尝试分解每个Jaeger组件及其Kubernetes部署。

#### Jaeger组件
Jaeger是一个开源的分布式追踪系统，它实现了OpenTracing规范。Jaeger包含了用于存储、可视化和过滤跟踪的组件。

##### 总体结构

![](https://miro.medium.com/max/875/1*I-JUG4eJhOTyrcs7teQwLg.png)

#### Jaeger客户端

应用程序跟踪检测从Jaeger客户端开始。下面的示例使用Jaeger Go库从环境变量初始化追踪程序配置，并启用客户端度量标准。
```
package tracer

import (
    "io"
    
    "github.com/uber/jaeger-client-go/config"
    jprom "github.com/uber/jaeger-lib/metrics/prometheus"
)

func NewTracer() (opentracing.Tracer, io.Closer, error) {
    // load config from environment variables
    cfg, _ := jaegercfg.FromEnv()

    // create tracer from config
    return cfg.NewTracer(
        config.Metrics(jprom.New()),
    )
}
```

Go客户端使通过环境变量初始化Jaeger配置变得简单。要设置的一些重要环境变量包括`JAEGER_SERVICE_NAME`，`JAEGER_AGENT_HOST`和`JAEGER_AGENT_PORT`。这里列出了Jaeger Go客户端支持的环境变量的完整列表。

要向gRPC微服务添加跟踪，我们将使用gRPC中间件在gRPC服务器和客户端上启用追踪。`grpc-ecosystem / go-grpc-middleware`有很多拦截器，包括对OpenTracing提供者无关的服务器端和客户端拦截器的支持。

`grpc_opentracing`包公开了可以使用任何`opentracing.Tracer`实现初始化的opentracing拦截器。在这我们使用链式一元和流拦截器初始化gRPC服务器。启用此选项将创建根服务器Span，并且对于每个服务器端的gRPC请求，跟踪程序将向服务中定义的每个RPC调用附加一个`Span`。

```
package grpc_server

import (
	"github.com/opentracing/opentracing-go"
	"github.com/grpc-ecosystem/go-grpc-middleware/tracing/opentracing"
	"github.com/grpc-ecosystem/go-grpc-middleware"
	"google.golang.org/grpc"
  
  	"github.com/masroorhasan/myapp/tracer"		
)

func NewServer() (*grpc.Server, error) {
 	// initialize tracer
	tracer, closer, err := tracer.NewTracer()
	defer closer.Close()
	if err != nil {
		return &grpc.Server{}, err
	}
	opentracing.SetGlobalTracer(tracer)

	// initialize grpc server with chained interceptors
	s := grpc.NewServer(
		grpc.StreamInterceptor(grpc_middleware.ChainStreamServer(
			// add opentracing stream interceptor to chain
			grpc_opentracing.StreamServerInterceptor(grpc_opentracing.WithTracer(tracer)),
  		)),
	  	grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
			// add opentracing unary interceptor to chain
			grpc_opentracing.UnaryServerInterceptor(grpc_opentracing.WithTracer(tracer)),
		)),
	)
	return s, nil
}

```

为了能够追踪gRPC服务的上游和下游请求，还必须使用客户端`opentracing`拦截器初始化gRPC客户端，如以下示例所示：

```
package grpc_client

import (
    "github.com/opentracing/opentracing-go"
    "github.com/grpc-ecosystem/go-grpc-middleware/tracing/opentracing"
    "github.com/grpc-ecosystem/go-grpc-middleware"
    "google.golang.org/grpc"
    
    "github.com/masroorhasan/myapp/tracer"
)

func NewClientConn(address string) (*grpc.ClientConn, error) {
    // initialize tracer
    tracer, closer, err := tracer.NewTracer()
    defer closer.Close()
    if err != nil {
        return &grpc.ClientConn{}, err
    }

    // initialize client with tracing interceptor using grpc client side chaining
    return grpc.Dial(
        address,
        grpc.WithStreamInterceptor(grpc_middleware.ChainStreamClient(
            grpc_opentracing.StreamClientInterceptor(grpc_opentracing.WithTracer(tracer)),
        )),
        grpc.WithUnaryInterceptor(grpc_middleware.ChainUnaryClient(
            grpc_opentracing.UnaryClientInterceptor(grpc_opentracing.WithTracer(tracer)),
        )),
     )
}

```

由gRPC中间件创建的初始spans被注入到go `context` ，从而提供强大的跟踪支持。`opentracing` go客户端可用于将子spans附加到初始context以进行更精细的追踪，以及控制每个span的生命周期，向追踪添加自定义标记等。

#### Jaeger代理

Jaeger代理是一个daemon，它通过UDP接收来自Jaeger客户端的spans，对其进行分批处理并将其转发给收集器。代理充当一个从客户端提出一批进行处理和路由的缓冲区。

尽管该代理被作为daemon构建，但在kubernetes设置中，代理可以配置为在应用程序Pod中作为sidecar容器运行或作为独立的`DaemonSet`运行。

以下部分讨论了每种部署策略的优缺点。

#### Jaeger代理作为Sidecar

sidecar Jaeger代理是一个与应用程序容器位于同一个pod中的容器。该应用程序表示为Jaeger service `myapp`，将通过`localhost`向代理发送Jaeger spans到端口`6381`。如前文所说的，这些配置是通过客户端中的环境变量`JAEGER_SERVICE_NAME`，`JAEGER_AGENT_HOST`和`JAEGER_AGENT_PORT`设置的。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: masroorhasan/myapp
        ports:
        - containerPort: 80
        env:
        - name: JAEGER_SERVICE_NAME
          value: myapp
        - name: JAEGER_AGENT_HOST
          value: localhost  # default
        - name: JAEGER_AGENT_PORT
          value: "6831"
        resources:
          limits:
            memory: 500M
            cpu: 250m
          requests:
            memory: 500M
            cpu: 250m
      # sidecar agent
      - name: jaeger-agent
        image: jaegertracing/jaeger-agent:1.6.0
        ports:
        - containerPort: 5775
          protocol: UDP
        - containerPort: 5778
          protocol: TCP
        - containerPort: 6831
          protocol: UDP
        - containerPort: 6832
          protocol: UDP
        command:
          - "/go/bin/agent-linux"
          - "--collector.host-port=jaeger-collector.monitoring:14267"
        resources:
          limits:
            memory: 50M
            cpu: 100m
          requests:
            memory: 50M
            cpu: 100m

```

通过这种方法，每个代理（以及每个应用程序）都可以配置成将追踪发送到不同的收集器（因此，不同的后端存储）。

然而，这种方法的最大缺点之一是代理的生命周期与应用程序的紧密耦合。跟踪的目的在于为您的应用程序在其生命周期中提供见解。更可能出现的是，代理sidecar容器在主应用程序容器之前就被终止，应用程序服务关闭期间的任何/所有重要跟踪都将丢失。这些追踪的丢失对于理解复杂服务交互的应用程序生命周期行为可能非常重要。这个github问题验证了在shutdown期间正确处理sigterm的必要性。

#### Jaeger Agent作为Daemonset

另一种方法是通过Kubernetes中的`DaemonSet`工作负载，在集群的每个节点中将代理作为daemon运行。`DaemonSet`工作负载确保在缩放节点时，使用它来缩放DaemonSet Pod的副本。

在此方案中，每个代理daemon负责在其节点中安排来自所有正在运行的应用程序（配置了Jaeger客户端）的追踪。这是通过将客户端中的`JAEGER_AGENT_HOST`设置为指向节点中代理的IP来配置的。代理`DaemonSet`配置了`hostNetwork：true`和适当的DNS策略，以便Pod使用与主机相同的IP。由于代理端口`6831`暴露在UDP上接受jaeger.thrift消息，因此daemon的Pod配置端口也与`hostPort：6831`绑定。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: masroorhasan/myapp
        ports:
        - containerPort: 80
        env:
        - name: JAEGER_SERVICE_NAME
          value: myapp
        - name: JAEGER_AGENT_HOST   # NOTE: Point to the Agent daemon on the Node
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: JAEGER_AGENT_PORT
          value: "6831"
        resources:
          limits:
            memory: 500M
            cpu: 250m
          requests:
            memory: 500M
            cpu: 250m
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: jaeger-agent
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: agent-daemonset
spec:
  template:
    metadata:
      labels:
        app: jaeger
        jaeger-infra: agent-instance
    spec:
      hostNetwork: true     # NOTE: Agent is configured to have same IP as the host/node
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: agent-instance
        image: jaegertracing/jaeger-agent:1.6.0
        command:
          - "/go/bin/agent-linux"
          - "--collector.host-port=jaeger-collector.monitoring:14267"
          - "--processor.jaeger-binary.server-queue-size=2000"
          - "--discovery.conn-check-timeout=500ms"
        ports:
        - containerPort: 5775
          protocol: UDP
        - containerPort: 6831
          protocol: UDP
          hostPort: 6831
        - containerPort: 6832
          protocol: UDP
        - containerPort: 5778
          protocol: TCP
        resources:
          requests:
            memory: 200M
            cpu: 200m
          limits:
            memory: 200M

```

一个人可能会被诱惑（就像我一样），用Kubernetes服务来保护`DaemonSet`。其背后的想法是不把应用程序追踪绑定到当前节点中的单个代理程序。使用服务可以摊开所有代理之间分配工作负载（spans）。从理论上讲，这降低了在受影响节点的单个代理程序单元发生故障时从应用程序实例中丢失spans的可能性。

但是，这不能实现，因为您的应用程序会扩展而且高负载会导致需要处理的追踪数量出现大幅增加。使用Kubernetes服务意味着通过网络从客户端向代理发送追踪。很快，我开始注意到大量垮掉的spans。客户端通过UDP thrift协议向代理发送spans，并且大的峰值会导致超过UDP最大数据包的大小并因此丢弃数据包。

解决方案是适当地分配资源，以便Kubernetes在群集中更均匀地调度pod。可以增加客户端的队列大小（设置`JAEGER_REPORTER_MAX_QUEUE_SIZE`环境变量），以便在代理故障转移时为spans提供足够的缓冲区。增加代理的内部队列大小（设置`processor.jaeger-binary.server-queue-size`值）也是有益的，所以它们不太可能开始减少spans。

#### Jaeger收集器服务

Jaeger 收集器负责从Jaeger Agent接收批量spans，通过处理管道运行它们并将它们存储在指定的存储后端。Spans以Jaeger代理的`jaeger.thrift`格式通过端口`14267`上的`TChannel`（TCP）协议发送。

Jaeger 收集器是无状态的，可以根据需要扩展到任意数量的实例。因此，收集器可以由Kubernetes内部服务（ClusterIP）提供，该服务可以将代理的内部流量负载平衡到不同的收集器实例中。

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jaeger-collector
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: collector-deployment
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: jaeger
        jaeger-infra: collector-pod
    spec:
      containers:
      - image: jaegertracing/jaeger-collector:1.6.0
        name: jaeger-collector
        args: ["--config-file=/conf/collector.yaml"]
        ports:
        - containerPort: 14267
          protocol: TCP
        - containerPort: 14268
          protocol: TCP
        - containerPort: 9411
          protocol: TCP
        readinessProbe:
          httpGet:
            path: "/"
            port: 14269
        volumeMounts:
        - name: jaeger-configuration-volume
          mountPath: /conf
        env:
        - name: SPAN_STORAGE_TYPE
          valueFrom:
            configMapKeyRef:
              name: jaeger-configuration
              key: span-storage-type
      volumes:
        - configMap:
            name: jaeger-configuration
            items:
              - key: collector
                path: collector.yaml
          name: jaeger-configuration-volume
      resources:
        requests:
          memory: 300M
          cpu: 250m
        limits:
          memory: 300M
          cpu: 250m
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: collector-service
spec:
  ports:
  - name: jaeger-collector-tchannel
    port: 14267
    protocol: TCP
    targetPort: 14267
  selector:
    jaeger-infra: collector-pod
  type: ClusterIP
view rawjae

```

#### Jaeger查询服务

查询服务是Jaeger服务器，用于支持UI。它负责从存储中检索跟踪并将其格式化以在UI上显示。根据查询服务的使用情况，其资源占用可能非常小。

设置内部Jaeger UI的入口以指向后端查询服务。

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jaeger-query
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: query-deployment
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: jaeger
        jaeger-infra: query-pod
    spec:
      containers:
      - image: jaegertracing/jaeger-query:1.6.0
        name: jaeger-query
        args: ["--config-file=/conf/query.yaml"]
        ports:
        - containerPort: 16686
          protocol: TCP
        readinessProbe:
          httpGet:
            path: "/"
            port: 16687
        volumeMounts:
        - name: jaeger-configuration-volume
          mountPath: /conf
        env:
        - name: SPAN_STORAGE_TYPE
          valueFrom:
            configMapKeyRef:
              name: jaeger-configuration
              key: span-storage-type
        resources:
          requests:
            memory: 100M
            cpu: 100m
          limits:
            memory: 100M
            cpu: 100m
      volumes:
        - configMap:
            name: jaeger-configuration
            items:
              - key: query
                path: query.yaml
          name: jaeger-configuration-volume
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: query-service
spec:
  ports:
  - name: jaeger-query
    port: 16686
    targetPort: 16686
  selector:
    jaeger-infra: query-pod
  type: ClusterIP
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: jaeger-ui
 namespace: monitoring
 annotations:
   kubernetes.io/ingress.class: traefik # or nginx or whatever ingress controller
spec:
 rules:
 - host: jaeger.internal-host # your jaeger internal endpoint
   http:
     paths:
     - backend:
         serviceName: jaeger-query
         servicePort: 16686 
view rawjaeger-query.yaml hosted
```

#### 存储配置

Jaeger支持ElasticSearch和Cassandra作为存储后端。使用ElasticSearch进行存储可以实现强大的监控基础架构，将追踪和日志记录结合在一起。收集器处理管道的一部分是为其存储后端的追踪编制索引-这将使追踪显示在日志记录UI（例如Kibana）中，并将追踪ID绑定到结构化日志记录标签。您可以通过`SPAN_STORAGE_TYPE`上的环境变量将存储类型设置为ElasticSearch，并通过配置设定存储端点。

Kubernetes `ConfigMap`用于设置某些Jaeger组件的存储配置。 例如，Jaeger Collector和Query服务的存储后端类型和端点。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-configuration
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: configuration
data:
  span-storage-type: elasticsearch
  collector: |
    es:
      server-urls: http://elasticsearch:9200
    collector:
      zipkin:
        http-port: 9411
  query: |
    es:
      server-urls: http://elasticsearch:9200
```

#### 监控

如前文所述，追踪是监控基础架构的重要组成部分。这意味着，甚至追踪基础结构的组件也需要被监控。

Jaeger在每个组件的特定端口上公开了prometheus格式的指标。如果有prometheus节点导出器正在运行（应该是绝对的），它们正在搜索特定端口上的度量标准-那么就将Jaeger组件的度量标准端口映射到节点导出器正在抓取度量标准的端口。

这可以通过更新Jaeger服务（代理，收集器，查询）来将其度量端口（5778,14628或16686）映射到节点导出器期望获取度量的端口（例如8888/8080）。

需要追踪的一些重要指标：

+ 每个组件的运行状况 - 内存使用量：

`sum(rate(container_memory_usage_bytes{container_name=~”^jaeger-.+”}[1m])) by (pod_name)`

+ Jaeger 代理的批量故障：

`sum(rate(container_cpu_usage_seconds_total{container_name=~"^jaeger-.+"}[1m])) by (pod_name)`

+ 收集器丢弃的跨度：

`sum(rate(jaeger_agent_tc_reporter_jaeger_batches_failures[1m])) by (pod)`

+ 收集器的队列延迟（P95）：

`histogram_quantile(0.95, sum(rate(jaeger_collector_in_queue_latency_bucket[1m])) by (le, pod))`

这些指标对每个组件的运行方式提供了重要的见解，并且应该使用历史数据来优化设置。

这是一篇很长的文章，如果您看到了这里，那么我希望这篇文章对您追求可靠的跟踪基础设施有帮助：）。
我很想听听你的Jaeger / Zipkin追踪设置 - 随时在GitHub或Twitter上联系。 谢谢阅读！

原文作者：Masroor Hasan

原文地址：[Distributed Tracing Infrastructure with Jaeger on Kubernetes](https://medium.com/@masroor.hasan/tracing-infrastructure-with-jaeger-on-kubernetes-6800132a677)
