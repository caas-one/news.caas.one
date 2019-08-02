# 使用pod安全策略与kubeadm

2018年9月13日

## 目的

说明使用[pod安全策略](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)和使用[kubernem](https://github.com/kubernetes/kubeadm)安装k8s

Pod安全策略是一种限制容器在k8s上运行时可以执行的操作的机制，例如阻止其作为特权容器的运行，与主机网络一起运行等。

阅读[文档](https://pmcgrath.net/using-pod-security-policies-with-kubeadm)，了解如何使用它来提高安全性

我尽可能的寻找有关引导kubeadm集群的所有信息，内容如下：

## TLDR

这个帖子很长，所以我做了一个完整的插图，也就是说你需要：

- 在主运行kubeadm init并启用PodSecurityPolicy许可控制器
- 使用RBAC配置添加一些pod安全策略 - 足以允许CNI和DNS等启动
  - 没有它，CNI守护进程将无法启动
- 应用您的CNI提供程序，该提供程序可以使用以前创建的一个pod安全策略
- 通过kubeadm join完成配置集群添加节点
- 在向群集添加更多工作负载时，请检查是否需要其他pod安全策略和RBAC配置

## 我们将要做什么

这是我在kubeadm安装上运行pod安全策略所采取的步骤

- 为主init 配置pod安全策略许可[控制器](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy)
- 为控制平面组件配置一些pod安全策略
- 配置CNI提供商 - 这里将使用flannel

然后将使用以下内容演示其他一些pod安全策略方案

- 安装一个具有一些特定要求的nginx-ingress控制器 - 这只是为了说明添加其他策略
- 安装没有特定pod安全策略要求的常规服务 - 基于[httpbin.org](https://hub.docker.com/r/kennethreitz/httpbin/)

## 环境

- 将仅说明单个节点而不是具有HA的多节点群集
- Ubuntu
  - 根据kubeadm的要求切换
  - 为UTC配置时区
- Docker
  - 假设当前用户在docker组中
- kubernetes 1.11.3 - 使用RBAC

## 精心准备

按照说明安装[kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)

让我们安装[jq](https://stedolan.github.io/jq/manual/)，我们将用它来进行一些json输出处理

```
sudo apt-get update
sudo apt-get install -y jq
```

验证kubeadm版本

```
sudo kubeadm version
```

接着我们将在下面创建的内容创建一个目录，在以下所有说明中我们假设您在此目录中
我们将在下面创建的内容里创建一个目录，下面的所有说明都假定您在这个目录中

```
mkdir ~/psp-inv
cd ~/psp-inv
```

## kubeadm配置文件

将创建这个文件并将其用于主服务器上的**kubeadm init**

使用此内容创建一个**kubeadm-config.yaml**文件 - 请注意我们必须指定podBubnet为10.244.0.0/16的flannel

请注意，此文件对于此演示是最小的，如果您使用更高版本的kubeadm，则可能需要更改apiVersion

```
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
apiServerExtraArgs:
  enable-admission-plugins: PodSecurityPolicy
controllerManagerExtraArgs:
  address: 0.0.0.0
kubernetesVersion: v1.11.3
networking:
  podSubnet: 10.244.0.0/16
schedulerExtraArgs:
  address: 0.0.0.0
```

## Master init

```
sudo kubeadm init --config kubeadm-config.yaml
```

按照上面命令输出中的说明获取您自己的kubeconfig文件副本

如果要将工作节点添加到群集，请记下加入消息

让我们检查主节点的状态

```
kubectl get nodes

NAME                    STATUS     ROLES     AGE       VERSION
pmcgrath-k8s-master     NotReady   master    1m        v1.11.3
```

这样说明节点还没有准备好，因为它正在等待CNI

让我们检查一下pods

```
kubectl get pods --all-namespaces
No resources found.
```

所以，如果我们没有启用pod安全策略准入控制，则通常看不到任何正在运行的pod

让我们检查 docker

```
docker container ls --format '{{ .Names }}'

k8s_kube-scheduler_kube-scheduler-pmcgrath-k8s-master_kube-system_a00c35e56ebd0bdfcd77d53674a5d2a1_0
k8s_kube-controller-manager_kube-controller-manager-pmcgrath-k8s-master_kube-system_fd832ada507cef85e01885d1e1980c37_0
k8s_etcd_etcd-pmcgrath-k8s-master_kube-system_16a8af6b4a79e9b0f81092f85eab37cf_0
k8s_kube-apiserver_kube-apiserver-pmcgrath-k8s-master_kube-system_db201a8ecaf8e99623b425502a6ba627_0
k8s_POD_kube-controller-manager-pmcgrath-k8s-master_kube-system_fd832ada507cef85e01885d1e1980c37_0
k8s_POD_kube-scheduler-pmcgrath-k8s-master_kube-system_a00c35e56ebd0bdfcd77d53674a5d2a1_0
k8s_POD_kube-apiserver-pmcgrath-k8s-master_kube-system_db201a8ecaf8e99623b425502a6ba627_0
k8s_POD_etcd-pmcgrath-k8s-master_kube-system_16a8af6b4a79e9b0f81092f85eab37cf_0
```

所以容器正在运行，但是没有显示kubectl

让我们检查事件

```
kubectl get events --namespace kube-system
```

会看到类似这样的错误：pods“kube-proxy-”被禁止：没有可用于验证pod请求的提供程序

## 配置pod安全策略

我已经配置了

- 任何工作负载都可以使用的默认pod安全策略，它没有什么特权，但应该适用于大多数工作负载
  - 将创建一个RBAC ClusterRole
  - 将为任何经过身份验证的用户创建RBAC ClusterRoleBinding
- 我授予节点和kube-system命名空间中的所有服务帐户访问权限的特权pod安全策略
  - 思考是限制访问此命名空间
  - 应该只在此命名空间中运行k8s组件
  - 将创建一个RBAC ClusterRole
  - 将在kube-system命名空间中创建RBAC RoleBinding

使用此内容创建**default-psp-with-rbac.yaml**文件

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  annotations:
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'docker/default'
  name: default
spec:
  allowedCapabilities: []  # default set of capabilities are implicitly allowed
  allowPrivilegeEscalation: false
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  hostIPC: false
  hostNetwork: false
  hostPID: false
  privileged: false
  readOnlyRootFilesystem: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsNonRoot'
  supplementalGroups:
    rule: 'RunAsNonRoot'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  volumes:
  - 'configMap'
  - 'downwardAPI'
  - 'emptyDir'
  - 'persistentVolumeClaim'
  - 'projected'
  - 'secret'
  hostNetwork: false
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'

---

# Cluster role which grants access to the default pod security policy
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: default-psp
rules:
- apiGroups:
  - policy
  resourceNames:
  - default
  resources:
  - podsecuritypolicies
  verbs:
  - use

---

# Cluster role binding for default pod security policy granting all authenticated users access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: default-psp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: default-psp
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
```

使用此内容创建**privileged-psp-with-rbac.yaml**文件

```
# Should grant access to very few pods, i.e. kube-system system pods and possibly CNI pods
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  annotations:
    # See https://kubernetes.io/docs/concepts/policy/pod-security-policy/#seccomp
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
  name: privileged
spec:
  allowedCapabilities:
  - '*'
  allowPrivilegeEscalation: true
  fsGroup:
    rule: 'RunAsAny'
  hostIPC: true
  hostNetwork: true
  hostPID: true
  hostPorts:
  - min: 0
    max: 65535
  privileged: true
  readOnlyRootFilesystem: false
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  volumes:
  - '*'

---

# Cluster role which grants access to the privileged pod security policy
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: privileged-psp
rules:
- apiGroups:
  - policy
  resourceNames:
  - privileged
  resources:
  - podsecuritypolicies
  verbs:
  - use

---

# Role binding for kube-system - allow nodes and kube-system service accounts - should take care of CNI i.e. flannel running in the kube-system namespace
# Assumes access to the kube-system is restricted
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kube-system-psp
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: privileged-psp
subjects:
# For the kubeadm kube-system nodes
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:nodes
# For all service accounts in the kube-system namespace
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts:kube-system
```

### 使用RBAC配置应用上述pod安全策略

```
kubectl apply -f default-psp-with-rbac.yaml
kubectl apply -f privileged-psp-with-rbac.yaml
```

### 再次校验

一段时间后，控制平面pods将处于运行状态，coredns pods将挂起-等待CNI

```
kubectl get pods --all-namespaces --output wide --watch
```

在配置CNI之前,控制平面pods将再次启动失败，因为节点仍未准备就绪

### 安装flannel

看到[这里](https://github.com/coreos/flannel)

只有现在存在**privileged** pod安全策略才能完成此操作，并且kube-system 中的flannel服务帐户将能够使用

如果使用不同的CNI提供程序，则应使用其安装说明，可能需要更改用于kubeadm init的kubeadm-config.yaml文件中的podSubnet

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 校验

```
kubectl get pods --all-namespaces --output wide --watch
```

所有pods最终将进入运行状态，包括coredns pod(s)

```
kubectl get nodes
```

节点现已准备就绪

### 允许主服务器上的工作负载

如果你想启动工作节点，你可以正常使用**kubeadm join**命令，利用kubeadm init输出，在这里跳过。

此工作节点上没有特别需要加入集群pod安全策略

允许主节点上的工作负载，因为我们只是尝试在单个节点群集上进行验证

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 服务器入口

将使用[此处](https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml)的清单

这将创建一个新的命名空间和单个实例入口控制器，足以说明其他pod安全策略

### 命名空间

由于命名空间尚不存在，因此我们可以创建，以便我们可以引用服务帐户并创建角色绑定

```
kubectl create namespace ingress-nginx
```

### 让我们创建一个pod安全策略

此pod安全策略基于部署[清单](https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml)

使用此内容创建文件**nginx-ingress-psp-with-rbac.yaml**

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  annotations:
    # Assumes apparmor available
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'docker/default'
  name: ingress-nginx
spec:
  # See nginx-ingress-controller deployment at https://github.com/kubernetes/ingress-nginx/blob/master/deploy/mandatory.yaml
  # See also https://github.com/kubernetes-incubator/kubespray/blob/master/roles/kubernetes-apps/ingress_controller/ingress_nginx/templates/psp-ingress-nginx.yml.j2
  allowedCapabilities:
  - NET_BIND_SERVICE
  allowPrivilegeEscalation: true
  fsGroup:
    rule: 'MustRunAs'
    ranges:
    - min: 1
      max: 65535
  hostIPC: false
  hostNetwork: false
  hostPID: false
  hostPorts:
  - min: 80
    max: 65535
  privileged: false
  readOnlyRootFilesystem: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
    ranges:
    - min: 33
      max: 65535
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
    # Forbid adding the root group.
    - min: 1
      max: 65535
  volumes:
  - 'configMap'
  - 'downwardAPI'
  - 'emptyDir'
  - 'projected'
  - 'secret'

---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ingress-nginx-psp
  namespace: ingress-nginx
rules:
- apiGroups:
  - policy
  resourceNames:
  - ingress-nginx
  resources:
  - podsecuritypolicies
  verbs:
  - use

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ingress-nginx-psp
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx-psp
subjects:
# Lets cover default and nginx-ingress-serviceaccount service accounts
# Could have altered default-http-backend deployment to use the same service acccount to avoid granting the default service account access
- kind: ServiceAccount
  name: default
- kind: ServiceAccount
  name: nginx-ingress-serviceaccount
```

让我们先申请

```
kubectl apply -f nginx-ingress-psp-with-rbac.yaml
```

### 创建nginx-ingress工作负载

- 将删除controller-publish-service arg，因为我们在这里不需要

```
curl -s https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml | sed '/--publish-service/d'  | kubectl apply -f -
```

检查pods

```
kubectl get pods --namespace ingress-nginx --watch
```

现在可以看到pod安全策略附加了注释

```
kubectl get pods --namespace ingress-nginx --selector app.kubernetes.io/name=ingress-nginx -o json | jq -r  '.items[0].metadata.annotations."kubernetes.io/psp"'
```

## Httpbin.org的工作负载

让我们在默认pod安全策略足够的情况下部署工作负载

使用此内容创建一个**httpbin.yaml**文件

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: httpbin
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: httpbin
  template:
    metadata:
      labels:
        app.kubernetes.io/name: httpbin
    spec:
      containers:
      - args: ["-b", "0.0.0.0:8080", "httpbin:app"]
        command: ["gunicorn"]
        image: docker.io/kennethreitz/httpbin:latest
        imagePullPolicy: Always
        name: httpbin
        ports:
        - containerPort: 8080
          name: http
      restartPolicy: Always

---

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
  labels:
    app.kubernetes.io/name: httpbin
  name: httpbin
spec:
  rules:
  - host: my.httpbin.com
    http:
      paths:
      - path:
        backend:
          serviceName: httpbin
          servicePort: 8080

---

apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: httpbin
  name: httpbin
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app.kubernetes.io/name: httpbin
```

创建命名空间并运行工作负载

```
kubectl create namespace demo
kubectl apply --namespace demo -f httpbin.yaml
```

让我们检查pod是否存在以及是否使用了默认策略

```
kubectl get pods --namespace demo

kubectl get pods --namespace demo --selector app.kubernetes.io/name=httpbin -o json | jq -r  '.items[0].metadata.annotations."kubernetes.io/psp"'
```

### 测试工作负载

将通过入口控制器pod实例进行调用 - 我没有这个演示的入口服务

```
# Get nginx ingress controller pod IP
nginx_ip=$(kubectl get pods --namespace ingress-nginx --selector app.kubernetes.io/name=ingress-nginx --output json | jq -r .items[0].status.podIP)

# Test ingress and out httpbin workload
curl -H 'Host: my.httpbin.com' http://$nginx_ip/get
```

## 重置

如果像我一样经常弄乱它，你可以重置并重启

```
# Note: Will loose PKI also which is fine here as kubeadm master init will re-create
sudo kubeadm reset

# Should flush iptable rules after a kubeadm reset, see https://blog.heptio.com/properly-resetting-your-kubeadm-bootstrapped-cluster-nodes-heptioprotip-473bd0b824aa
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```

## 链接

- [文档](https://kubernetes.io/docs/concepts/policy/pod-security-policy)
- [Github发表评论](https://github.com/kubernetes/kubernetes/issues/62566#issuecomment-381360838)
- [安全建议](https://github.com/freach/kubernetes-security-best-practice/tree/master/PSP)
- [GCE政策](https://github.com/kubernetes/kubernetes/tree/master/cluster/gce/addons/podsecuritypolicies)
