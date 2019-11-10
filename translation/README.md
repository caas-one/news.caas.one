---
title: "README" 
date: 2018-10-15T15:0s4+08:00
categories: [ "translation"]
draft: false
---
###  翻译计划&&格式约定

*本列表中文章来源于日常发现的好的英文文章或者日报中好的英文文章*
*翻译文章约定使用MarkDown格式写作*

#### 翻译仓库格式约定
translation/image ---- 路径下为文章中所需图片，图片命名格式为 XXX-日期（月日 例 1010）
translation ---- 路径下为翻译后文章，文章命名格式为 文章名.md

文章中图片插入格式约定 ---- 图片使用url形式插入
#### 翻译文章列表格式约定
例：Logging Architecture <https://kubernetes.io/docs/concepts/cluster-administration/logging/>

如已经翻译，请写明翻译人姓名，并且标注已翻译

XXX(文章名)<https://XXX（文章链接）>（已翻译）（windghoul）

#### 操作示例
以[Kubernetes vs Docker](https://www.sumologic.com/blog/kubernetes-vs-docker/)为例:

1. Fork当前仓库到自己的Github
![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/fork-it.jpg)
![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/forking-it.jpg)

2. Clone自己的仓库到个人电脑本地
![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/clone-it.jpg)
![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/cloning-it.jpg)

3. 打开原文，完成翻译，并将markdown格式的译文保存到本地目录
![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/translate-then-save.jpg)
![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/translating-and-save.jpg)

4. 推送到自己的远程仓库
![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/translating-then-save.jpg)

5. 提交合并请求
![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/submitting-merge-request.jpg)

6. 审核完毕，合并到主仓库
![](https://github.com/caas-one/news.caas.one/blob/master/translation/images/reviewing.jpg)

7. 发表到社区
![](publish-to-community.png)

#### 文章列表

1. [Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) 已翻译(@Bucky Barenes)
2. [Building Microservices with Event Sourcing CQRS in Go using gRPC, NATS Streaming and CockroachDB](https://medium.com/@shijuvar/building-microservices-with-event-sourcing-cqrs-in-go-using-grpc-nats-streaming-and-cockroachdb-98) 已翻译(@lth2015)
3. [Handling Client Requests Properly with Kubernetes](https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/) 已翻译(@maxwell92)
4. [Kubernetes NodePort vs LoadBalancer vs Ingress? When should I use what?](https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0)已翻译(@Whisper-Wang)
5. [Dynamically Expand Volume with CSI and Kubernetes](https://kubernetes.io/blog/2018/08/02/ally-expand-volume-with-csi-and-kubernetes/)已翻译(@SheriffAlan)
6. [Understanding Kubernetes networking: ingress](https://medium.com/google-cloud/understanding-kubernetes-networking-ingress-1bc341c84078) 已翻译(@hyw-1228)
7. [Use of Let's Encrypt wildcard certs in Kubernetes](https://rimusz.net/lets-encrypt-wildcard-certs-in-kubernetes/)已翻译(@ky11n)
8. [How to Run HA MySQL on Google Kubernetes Engine](https://portworx.com/run-ha-mysql-google-kubernetes-engine/)已翻译(@SheriffAlan)
9. [Hands On with Linkerd 2.0](https://kubernetes.io/blog/2018/09/18/hands-on-with-linkerd-2.0/)已翻译(@qiaoyvlei)
10. [Introducing Kustomize; Template-free Configuration Customization for Kubernetes](https://kubernetes.io/blog/2018/05/29/introducing-kustomize-template-free-configuration-customization-for-kubernetes/)已翻译(@qiaoyvlei)
11. [How to Tag #Docker Images without Pulling them](https://dille.name/blog/2018/09/20/how-to-tag-docker-images-without-pulling-them/)已翻译(@esterwang)
12. [Resizing Persistent Volumes using Kubernetes](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/) 已翻译(@esterwang)
13. [Extending Kubernetes with Custom Resources and Operator Frameworks](https://speakerdeck.com/ianlewis/extending-kubernetes-with-custom-resources-and-operator-frameworks)已翻译(@hyw-1228)
14. [How, and When, to Deploy Serverless](https://thenewstack.io/how-and-when-to-deploy-serverless/)已翻译(@gaoyixiang1)
15. [In-depth introduction to Kubernetes admission webhooks](https://banzaicloud.com/blog/k8s-admission-webhooks/)已翻译(@hyw-1228)
16. [Debug a Kubernetes Service Locally with Telepresence](https://articles.microservices.com/debug-a-kubernetes-service-locally-with-telepresence-675eb6e94b09)已翻译(@wxppp)
17. [Distributed Tracing Infrastructure with Jaeger on Kubernetes](https://medium.com/@masroor.hasan/tracing-infrastructure-with-jaeger-on-kubernetes-6800132a677)已翻译(@wxppp)
18. [Cortex: Stateful Prometheus Monitoring for Multiple Clients](https://thenewstack.io/cortex-stateful-prometheus-monitoring-for-multiple-clients/) 已翻译(@esterwang)
19. [Health checking gRPC servers on Kubernetes](https://kubernetes.io/blog/2018/10/01/health-checking-grpc-servers-on-kubernetes/) 已翻译(@SheriffAlan)
20. [Modernizing Applications for Kubernetes](https://www.digitalocean.com/community/tutorials/modernizing-applications-for-kubernetes) 已翻译(@ky11n)
21. [Using pod security policies with kubeadm](https://pmcgrath.net/using-pod-security-policies-with-kubeadm)已翻译(@gaoyixiang1)
22. [Unbabel migrated to Kubernetes and you won’t believe what happened next!](https://medium.com/unbabel/unbabel-migrated-to-kubernetes-and-you-wont-believe-what-happened-next-b39f082def1c) 
23. [Be as Serverless as You Can, but Not More than That](https://dzone.com/articles/be-as-serverless-as-you-can-but-not-more-than-that)已翻译(@gaoxuanya)
24. [Container Networking Interface aka CNI](https://medium.com/@vikram.fugro/container-networking-interface-aka-cni-bdfe23f865cf)已翻译(@Whisper-Wang)
25. [Why You Should Not Neglect Your Developer’s Kubernetes Clusters](https://itnext.io/why-you-should-not-neglect-your-developers-kubernetes-clusters-a658c8ca0e78)已翻译(@hyw-1228)
26. [We’re open-sourcing etcdadm! Here’s what it means for Kubernetes in production](https://platform9.com/blog/were-open-sourcing-etcdadm-heres-what-it-means-for-kubernetes-in-production/)
<<<<<<< HEAD
27. [Kubernetes: The Surprisingly Affordable Platform for Personal Projects](http://www.doxsey.net/blog/kubernetes--the-surprisingly-affordable-platform-for-personal-projects)已翻译（@sjh080815）
28. [Kubernetes Add-ons for more Efficient Computing](https://akomljen.com/kubernetes-add-ons-for-more-efficient-computing/)
=======
27. [Kubernetes: The Surprisingly Affordable Platform for Personal Projects](http://www.doxsey.net/blog/kubernetes--the-surprisingly-affordable-platform-for-personal-projects)
28. [Kubernetes Add-ons for more Efficient Computing](https://akomljen.com/kubernetes-add-ons-for-more-efficient-computing/)已翻译(@ky11n)
>>>>>>> 794eb73d33353217d3726758129a098adaa2fb4d
29. [Knative what’s that now?](https://medium.com/@grapesfrog/knative-whats-that-now-65041e585d3d)
30. [Kubernetes and external DNS services](https://banzaicloud.com/blog/k8s-external-dns-route53/)已翻译(@zcxcurry30)
31. [Segregating Jenkins Agents on Kubernetes](https://medium.com/@kmadel/segregating-jenkins-agents-on-kubernetes-b2fa9c471423)
32. [Application Safety and Correctness Cannot Be Offloaded to Istio or Any Service Mesh](http://blog.christianposta.com/microservices/application-safety-and-correctness-cannot-be-offloaded-to-istio-or-any-service-mesh/)已翻译(@wxppp)
33. [Kubernetes v1.12: Introducing RuntimeClass](https://kubernetes.io/blog/2018/10/10/kubernetes-v1.12-introducing-runtimeclass/)已翻译(@hyw-1228)
34. [Introducing Volume Snapshot Alpha for Kubernetes](https://kubernetes.io/blog/2018/10/09/introducing-volume-snapshot-alpha-for-kubernetes/)已翻译(@hyw-1228)
35. [Getting Started with Docker on Windows Server 2019](https://blog.sixeyed.com/getting-started-with-docker-on-windows-server-2019/)已翻译(@ky11n)
36. [Improving the multi-team Kubernetes ingress experience with Heptio Contour 0.6](https://blog.heptio.com/improving-the-multi-team-kubernetes-ingress-experience-with-heptio-contour-0-6-55ae0c0cadef)已翻译(@Whisper-Wang)
37. [Docker Authentication with Keycloak](https://developers.redhat.com/blog/2017/10/31/docker-authentication-keycloak/)已翻译(@ky11n)
38. [Kubernetes Persistent Volumes with Deployment and StatefulSet](https://akomljen.com/kubernetes-persistent-volumes-with-deployment-and-statefulset/)已翻译(@gaoyixiang1)
39. [Spring Boot with Vault on Kubernetes](https://banzaicloud.com/blog/vault-java-spotguide/)已翻译(@gaoyixiang1)
40. [Service mesh data plane vs. control plane](https://blog.envoyproxy.io/service-mesh-data-plane-vs-control-plane-2774e720f7fc)已翻译(@ky11n)
41. [How to scale Microservices with Message Queues, Spring Boot, and Kubernetes](https://medium.freecodecamp.org/how-to-scale-microservices-with-message-queues-spring-boot-and-kubernetes-f691b7ba3acf)已翻译(@hyw-1228)
42. [Kubernetes Infrastructure](https://docs.okd.io/latest/architecture/infrastructure_components/kubernetes_infrastructure.html)已翻译(@Whisper-Wang)
43. [Kubernetes vs. Docker: What Does It Really Mean?](https://www.sumologic.com/blog/devops/kubernetes-vs-docker/)已翻译(@hyw-1228)
44. [Deploy Your First Deep Learning Model On Kubernetes With Python, Keras, Flask, and Docker](https://medium.com/analytics-vidhya/deploy-your-first-deep-learning-model-on-kubernetes-with-python-keras-flask-and-docker-575dc07d9e76)已翻译(@hyw-1228)
45. [How to use Envoy as a Load Balancer in Kubernetes](https://blog.markvincze.com/how-to-use-envoy-as-a-load-balancer-in-kubernetes/)已翻译(@gaoyixiang1)
46. [Autoscaling Applications on Kubernetes - A Primer](https://blog.tomkerkhove.be/2018/10/08/autoscaling-applications-on-kubernetes-a-primer/)已翻译(@ky11n)
47. [Imperative vs Declarative](https://medium.com/@dominik.tornow/imperative-vs-declarative-8abc7dcae82e)已翻译(@wxppp)
48. [To Better Understand Detroit’s Revival, Look At Its Water And Sewerage Department](https://www.forbes.com/sites/oracle/2018/10/09/to-better-understand-detroits-revival-look-at-its-water-and-sewerage-department/#23dd6326d63e)已翻译(@wxppp)
49. [Topology-Aware Volume Provisioning in Kubernetes](https://kubernetes.io/blog/2018/10/11/topology-aware-volume-provisioning-in-kubernetes/)已翻译(@hyw-1228)
50. [3 simple tricks for smaller Docker images](https://medium.com/skills-matter/3-simple-tricks-for-smaller-docker-images-cf2760645621)已翻译(@gaoxuanya)
51. [Microk8s puts up its Istio and sails away](https://itnext.io/microk8s-puts-up-its-istio-and-sails-away-104c5a16c3c2)已翻译(@hyw-1228)
52. [How Does The Kubernetes Networking Work? : Part 1](https://www.level-up.one/kubernetes-networking-pods-levelup/)已翻译(@esterwang)
53. [How Does The Kubernetes Networking Work? : Part 2](https://www.level-up.one/kubernetes-networking-series-two/)已翻译(@esterwang)
54. [How Does The Kubernetes Networking Work? : Part 3](https://www.level-up.one/kubernetes-networking-3-level-up/)已翻译(@esterwang)
55. [Kafka on Kubernetes - using etcd](https://banzaicloud.com/blog/kafka-on-etcd/)已翻译(@wxppp)
56. [Kubernetes Monitoring with Prometheus -The ultimate guide (part 1).](https://sysdig.com/blog/kubernetes-monitoring-prometheus/)已翻译(@wxppp)
57. [An introduction to Ansible Operators in Kubernetes](https://opensource.com/article/18/10/ansible-operators-kubernetes)已翻译(@hyw-1228)
58. [An Introduction to the Kubernetes DNS Service](https://www.digitalocean.com/community/tutorials/an-introduction-to-the-kubernetes-dns-service)已翻译(@esterwang)
59. [This guest post was written by Andrey Belik, Dana Groce, and Jason Mimick of MongoDB](https://blog.openshift.com/mongodb-kubernetes-operator/#.W8Y7ugmdw4A.twitter)已翻译(@ky11n)
60. [Deploy the Blockchain network using Kubernetes APIs on IBM Cloud](https://github.com/IBM/blockchain-network-on-kubernetes)
61. [How to use Envoy as a Load Balancer in Kubernetes](https://blog.markvincze.com/how-to-use-envoy-as-a-load-balancer-in-kubernetes/)已翻译(@ky11n)
62. [KubeDirector: The easy way to run complex stateful applications on Kubernetes](https://kubernetes.io/blog/2018/10/03/kubedirector-the-easy-way-to-run-complex-stateful-applications-on-kubernetes/)
63. [Kubernetes best practices: Setting up health checks with readiness and liveness probes](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes)
64. [Kubernetes Namespaces, Resource Quota, and Limits for QoS in Cluster](https://blog.couchbase.com/kubernetes-namespaces-resource-quota-limits-qos-cluster/)
65. [Kubernetes Autoscaling 101: Cluster Autoscaler, Horizontal Pod Autoscaler, and Vertical Pod Autoscaler](https://medium.com/magalix/kubernetes-autoscaling-101-cluster-autoscaler-horizontal-pod-autoscaler-and-vertical-pod-2a441d9ad231)
66. [Tag Cleanup Daemon for VMware Harbor](https://github.com/HylandSoftware/Harbor.Tagd)
67. [Kubernetes disaster recovery with Pipeline](https://banzaicloud.com/blog/k8s-disaster-recovery/)已翻译(@gaoyixiang1)
68. [IPVS-Based In-Cluster Load Balancing Deep Dive](https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/)
69. [gRPC On HTTP/2: Engineering A Robust, High Performance Protocol](https://www.cncf.io/blog/2018/08/31/grpc-on-http-2-engineering-a-robust-high-performance-protocol/)已翻译(@sjh080815)
70. [Understanding the Container Storage Interface (CSI)](https://medium.com/google-cloud/understanding-the-container-storage-interface-csi-ddbeb966a3b)
71. [Using go mod download to speed up Golang Docker builds](https://medium.com/@petomalina/using-go-mod-download-to-speed-up-golang-docker-builds-707591336888)
72. [Using Kubeless for Kubernetes Events](https://leebriggs.co.uk/blog/2018/10/16/using-kubeless-for-kubernetes-events.html)
73. [Best practices and anti-patterns for containerized deployments](https://techbeacon.com/best-practices-anti-patterns-containerized-deployments)
74. [Customizing Kubernetes Logging (Part 1)](https://medium.com/uptime-99/adopting-istio-in-your-kubernetes-clusters-a3e28ed6f4b7)
75. [Adopting Istio in Your Kubernetes Clusters](https://medium.com/uptime-99/adopting-istio-in-your-kubernetes-clusters-a3e28ed6f4b7)
76. [Rookout brings breakpoints back to Kubernetes](https://www.rookout.com/pr/rookout_brings_breakpoints_back_to-_kubernetes)已翻译（@sjh080815）
77. [Effectively Managing Kubernetes Resources with Cost Monitoring](https://medium.com/kubecost/effectively-managing-kubernetes-with-cost-monitoring-96b54464e419)
78. [Best practices for building Kubernetes Operators and stateful apps](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps)
79. [Unit Testing with the Kubernetes Client Library](https://matt-rickard.com/kubernetes-unit-testing/) 已翻译(@esterwang)
80. [Understanding resource limits in Kubernetes-memory](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-memory-6b41e9a955f9) 已翻译(@maxwell92)
81. [Understanding resource limits in Kubernetes-cpu](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-memory-6b41e9a955f://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b) 已翻译(@maxwell92)
82. [Testing Microservices, the sane way](https://medium.com/@copyconstruct/testing-microservices-the-sane-way-9bb31d158c16) 已翻译
83. [Diving Deep Into The Golang Channels](https://codeburst.io/diving-deep-into-the-golang-channels-549fd4ed21a8) 已翻译(@windghoul)
84. [Kubernetes-vs-Docker](https://www.sumologic.com/blog/kubernetes-vs-docker/) 已翻译（@ky11n、@wxppp、@zcxcurry30、@Bucky Barenes、@esterwang）
85. [Optimizing-Kubernetes-resource-allocation-in-production](https://opensource.com/article/18/12/optimizing-kubernetes-resource-allocation-production?utm_campaign=intrel) 已翻译(@maxwell92)
86. [Least Privilege in Kubernetes Using Impersonation](https://johnharris.io/2019/08/least-privilege-in-kubernetes-using-impersonation/) 正在翻译@maxwell92
87. [Operators and Controllers, What is the Difference?](https://octetz.com/posts/k8s-controllers-vs-operators)已翻译(@ky11n)
88. [Writing Kubernetes Custom Controllers](https://medium.com/@cloudark/kubernetes-custom-controllers-b6c7d0668fdf)已翻译(@sjh080815)
89. [Handling Client Requests Properly with Kubernetes](https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/?utm_sq=g6rhsr1fby)
90. [Kubernetes admission controllers for secure deployments](https://sysdig.com/blog/kubernetes-admission-controllers/?utm_sq=g6qty0qqim)已翻译(@sjh080815)
