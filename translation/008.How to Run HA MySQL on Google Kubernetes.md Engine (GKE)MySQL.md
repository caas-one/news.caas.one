--- 
title: "How to Run HA MySQL on Google Kubernetes.md Engine (GKE)MySQL" 
date: 2018-12-03T16:48:03+08:00
categories: [ "translation"]
draft: false
---
# 如何在Google Kubernetes Engine上运行HA MySQL

![Social-Graphic](https://github.com/caas-one/news.caas.one/blob/master/translation/images/Twitter-Social-Graphic-85.png?raw=true)

本文是“在Kubernetes中运行MySQL”系列之一。我们已经发表了多篇关于具体平台和具体用例中在Kubernetes中运行MySQL的文章。如果你正在寻找关于在具体的Kubernetes平台，请查阅相关文章。

[*Running HA MySQL on Amazon Elastic Container Service for Kubernetes (EKS)*](https://portworx.com/ha-mysql-amazon-eks/)

[*Running HA MySQL on Azure Kubernetes Service (AKS)*](https://portworx.com/run-ha-mysql-azure-kubernetes-service/)

[*Running HA MySQL on Red Hat OpenShift*](https://portworx.com/ha-mysql-openshift/)

[*How to Backup and Restore MySQL on Red Hat OpenShift*](https://portworx.com/backup-restore-mysql-red-hat-openshift/)

现在进入正题...

谷歌[*Kubernetes Engine(GKE)*](https://cloud.google.com/kubernetes-engine/)是一个用于在谷歌云平台中部署容器化应用程序的托管、可生产的环境。GKE于2015年上线，是第一个基于谷歌在容器中运营Gmail、YouTube等服务12年多经验的托管容器平台之一。为了完全消除对安装，管理，和操作Kubernetes集群的需要，GKE允许客户快速安装和运行Kubernetes。

Portworx是一个用于运行部署在多种编排引擎（包括Kubernetes）的持久性工作负载的云本地存储平台。利用Portworx，用户可以根据其选择利用任意框架和任意容器和容器调度管理数据库。无论服务运行在哪里，Portworx都能为有状态服务提供独立的数据管理层。
本教程介绍了在GKE上部署和管理高可用MySQL数据库的步骤。
总之，在谷歌云平台运行HA MySQL，你需要：

1、部署一个GKE集群

2、在GKE上安装类似Portworx的云本地存储解决方案作为守护进程组

3、创建存储类来定义存储需求，例如复制因子、快照策略，性能概要

4、利用Kubernetes部署MySQL

5、通过杀死或禁止调度节点检测故障转移

#### 如何部署GKE集群

在部署GKE集群运行Portworx之前，你需要确认集群是建立在Ubuntu系统上。由于基于容器优化OS (COS)的GKE集群的某些限制，Portworx需要Ubuntu作为GKE节点的基本镜像。
以下命令在ap-south-1-a空间内配置了3个节点的GKE集群。你可以根据需要适当修改。

```
$ gcloud container clusters create "gke-px" \
--zone "asia-south1-a" \
--username "admin" \
--cluster-version "1.8.10-gke.0" \
--machine-type "n1-standard-4" \
--image-type "UBUNTU" \
--disk-type "pd-ssd" \
--disk-size "100" \
--num-nodes "3" \
--enable-cloud-logging \
--enable-cloud-monitoring \
--network "default" \
--addons HorizontalPodAutoscaling,HttpLoadBalancing,KubernetesDashboard
```

当集群已就绪，用以下命令配置kubectl CLI：

```
$ gcloud container clusters get-credentials gke-px --zone asia-south1-a
```

Portworx需要为用户建立ClusterRoleBinding。如果没有此配置，执行命令会报clusterroles.rbac.authorization.k8s.io "portworx-pvc-controller-role" is forbidden错误。

用以下命令创建ClusterRoleBinding：

```
$ kubectl create clusterrolebinding cluster-admin-binding \
--clusterrole cluster-admin \
--user $(gcloud config get-value account)
```

现在你应该已经在谷歌云平台搭建好了三个节点的Kubernetes集群。

```
$ kubectl get nodes
NAME                                    STATUS    ROLES     AGE       VERSION
gke-gke-px-default-pool-177a3f0b-nvxm   Ready         9d        v1.8.10-gke.0
gke-gke-px-default-pool-177a3f0b-slkb   Ready         9d        v1.8.10-gke.0
gke-gke-px-default-pool-177a3f0b-st0n   Ready         9d        v1.8.10-gke.0
```

![px-mysql-gke-0](https://github.com/caas-one/news.caas.one/blob/master/translation/images/px-mysql-gke-0-768x137.jpg?raw=true)

### 在GKE中安装Portworx

在Amazon GKE中安装Portworx和通过Kops安装Kubernetes几乎没什么不同。

[*Portworx GKE文档*](https://docs.portworx.com/cloud/gcp/gke.html)包含在部署在GCP中的Kubernetes环境中运行Portworx集群所涉及的步骤。

在执行下一步之前，我们需要在GKE上启动Portworx集群。 Kube-system命名空间应该有出于运行状态的Portoworx pods。

```
$ kubectl get pods -n=kube-system -l name=portworx
NAME             READY     STATUS    RESTARTS   AGE
portworx-g8sq5   1/1       Running   0          9d
portworx-gnjpx   1/1       Running   0          9d
portworx-tbrc6   1/1       Running   0          9d
```

![px-mysql-gke-1](https://github.com/caas-one/news.caas.one/blob/master/translation/images/px-mysql-gke-1.jpg?raw=true)

#### 为MySQL创建Kubernetes存储类

一旦GKE集群启动并运行，并且安装并配置了Portworx，我们将部署一个高可用的MySQL数据库。

通过Kubernetes存储类对象，管理员可以定义集群中提供的Portworx卷的不同类型。在动态提供判断时将会使用这些类。存储类定义复制因子、I/O配置文件(例如，用于数据库或CMS)和优先级(例如，SSD或HDD)。这些参数影响工作负载的可用性和吞吐量，我们可以为每个卷指定这些参数。这点非常重要，因为生产级数据库的要求不同于开发级的Jenkins集群。

在这个例子中，我们部署的存储类副本数为3，IO配置文件设置为"db"，优先级设置为"high"。这意味着存储将针对低延迟数据库工作负载(如MySQL)进行优化，同时将被放置到集群中可用的最高性能存储上。注意，我们还在存储类中涉及到了xfs文件系统。

```
$ cat > px-mysql-sc.yaml << EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
    name: px-ha-sc
provisioner: kubernetes.io/portworx-volume
parameters:
   repl: "3"
   io_profile: "db"
   io_priority: "high"
   fs: "xfs"
EOF
```

```
$ kubectl create -f px-mysql-sc.yaml
storageclass.storage.k8s.io "px-ha-sc" created

$ kubectl get sc
NAME                PROVISIONER                     AGE
px-ha-sc            kubernetes.io/portworx-volume   10s
stork-snapshot-sc   stork-snapshot                  3d
```

#### 在Kubernetes上创建MySQL PVC

现在，我们可以在存储类的基础上创建PVC。幸好有动态配置，这些声明将在没有显式配置持久卷(PV)的情况下创建。

```
$ cat > px-mysql-pvc.yaml << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
   name: px-mysql-pvc
   annotations:
     volume.beta.kubernetes.io/storage-class: px-ha-sc
spec:
   accessModes:
     - ReadWriteOnce
   resources:
     requests:
       storage: 1Gi
EOF

$ kubectl create -f px-mysql-pvc.yaml
persistentvolumeclaim "px-mysql-pvc" created

$ kubectl get pvc
NAME           STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
px-mysql-pvc   Bound     pvc-ebe875f5-99e8-11e8-9de8-42010aa00fdd   1Gi        RWO            px-ha-sc       6s
```

#### GKE上部署MySQL

最后，我们来创建MySQL实例来作为Kubernetes部署对象。简单起见，我们先只部署一个MySQL pod。因为Portworx为高可用性提供同步复制，所以单个MySQL实例可能是MySQL数据库的最佳部署选项。Portworx也为多节点MySQL集群提供备用盘卷，这个自由选择。

```
$ cat > px-mysql-app.yaml << EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mysql
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      schedulerName: stork
      containers:
      - name: mysql
        image: mysql:5.6
        imagePullPolicy: "Always"
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password        
        ports:
        - containerPort: 3306
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mysql-data
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: px-mysql-pvc
EOF
```

```
$ kubectl create -f px-mysql-app.yaml
deployment.extensions "mysql" created
```

上面定义的MySQL部署显式地与前面步骤中创建的PVC相关联。

该部署创建一个单独的pod用来运行Portworx支持的MySQL。

```
$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
mysql-dff54d66d-m9r6q   1/1       Running   0          6s
```

我们可以通过访问使用MySQL pod运行的pxctl工具来检查Portworx卷。

```
$ VOL=`kubectl get pvc | grep px-mysql-pvc | awk '{print $3}'`
$ PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
$ kubectl exec -it $PX_POD -n kube-system -- /opt/pwx/bin/pxctl volume inspect ${VOL}
Volume	:  549063659345266352
	Name            	 :  pvc-ebe875f5-99e8-11e8-9de8-42010aa00fdd
	Size            	 :  1.0 GiB
	Format          	 :  xfs
	HA              	 :  3
	IO Priority     	 :  LOW
	Creation time   	 :  Aug 7 02:23:50 UTC 2018
	Shared          	 :  no
	Status          	 :  up
	State           	 :  Attached: gke-gke-px-default-pool-177a3f0b-st0n (10.240.0.3)
	Device Path     	 :  /dev/pxd/pxd549063659345266352
	Labels          	 :  namespace=default,pvc=px-mysql-pvc
	Reads           	 :  77
	Reads MS        	 :  64
	Bytes Read      	 :  339968
	Writes          	 :  481
	Writes MS       	 :  10220
	Bytes Written   	 :  166514688
	IOs in progress 	 :  0
	Bytes used      	 :  126 MiB
	Replica sets on nodes:
		Set 0
		  Node 		 : 10.240.0.4 (Pool 0)
		  Node 		 : 10.240.0.2 (Pool 0)
		  Node 		 : 10.240.0.3 (Pool 0)
	Replication Status	 :  Up
	Volume consumers	 :
		- Name           : mysql-8776f4ff-p7b22 (2b1c0719-99e9-11e8-9de8-42010aa00fdd) (Pod)
		  Namespace      : default
		  Running on     : gke-gke-px-default-pool-177a3f0b-st0n
		  Controlled by  : mysql-8776f4ff (ReplicaSet)
```

![px-mysql-gke-2](https://github.com/caas-one/news.caas.one/blob/master/translation/images/px-mysql-gke-2-768x480.jpg?raw=true)

上面命令的输出对创建的MySQL数据库实例的卷进行确认。

### Kubernetes中MySQL pod执行失败

#### 填充样本数据

接下来在数据库中填充样本数据。

我们将首先找到访问shell的MySQL pod。

```
$ POD=`kubectl get pods -l app=mysql | grep Running | grep 1/1 | awk '{print $1}'`

$ kubectl exec -it $POD -- mysql -uroot -ppassword
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.6.40 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

现在我们进入shell中，可以填充数据库和表了。

```
mysql> CREATE DATABASE `classicmodels`;

mysql> USE `classicmodels`;

mysql> CREATE TABLE `offices` (
  `officeCode` varchar(10) NOT NULL,
  `city` varchar(50) NOT NULL,
  `phone` varchar(50) NOT NULL,
  `addressLine1` varchar(50) NOT NULL,
  `addressLine2` varchar(50) DEFAULT NULL,
  `state` varchar(50) DEFAULT NULL,
  `country` varchar(50) NOT NULL,
  `postalCode` varchar(15) NOT NULL,
  `territory` varchar(10) NOT NULL,
  PRIMARY KEY (`officeCode`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

mysql> insert  into `offices`(`officeCode`,`city`,`phone`,`addressLine1`,`addressLine2`,`state`,`country`,`postalCode`,`territory`) values 
('1','San Francisco','+1 650 219 4782','100 Market Street','Suite 300','CA','USA','94080','NA'),
('2','Boston','+1 215 837 0825','1550 Court Place','Suite 102','MA','USA','02107','NA'),
('3','NYC','+1 212 555 3000','523 East 53rd Street','apt. 5A','NY','USA','10022','NA'),
('4','Paris','+33 14 723 4404','43 Rue Jouffroy D\'abbans',NULL,NULL,'France','75017','EMEA'),
('5','Tokyo','+81 33 224 5000','4-1 Kioicho',NULL,'Chiyoda-Ku','Japan','102-8578','Japan'),
('6','Sydney','+61 2 9264 2451','5-11 Wentworth Avenue','Floor #2',NULL,'Australia','NSW 2010','APAC'),
('7','London','+44 20 7877 2041','25 Old Broad Street','Level 7',NULL,'UK','EC2N 1HN','EMEA');
```

现在执行一下查询语句：

```
mysql> select `officeCode`,`city`,`phone`,`addressLine1`,`city` from `offices`;
+------------+---------------+------------------+--------------------------+---------------+
| officeCode | city          | phone            | addressLine1             | city          |
+------------+---------------+------------------+--------------------------+---------------+
| 1          | San Francisco | +1 650 219 4782  | 100 Market Street        | San Francisco |
| 2          | Boston        | +1 215 837 0825  | 1550 Court Place         | Boston        |
| 3          | NYC           | +1 212 555 3000  | 523 East 53rd Street     | NYC           |
| 4          | Paris         | +33 14 723 4404  | 43 Rue Jouffroy D'abbans | Paris         |
| 5          | Tokyo         | +81 33 224 5000  | 4-1 Kioicho              | Tokyo         |
| 6          | Sydney        | +61 2 9264 2451  | 5-11 Wentworth Avenue    | Sydney        |
| 7          | London        | +44 20 7877 2041 | 25 Old Broad Street      | London        |
+------------+---------------+------------------+--------------------------+---------------+
7 rows in set (0.01 sec)
```

![px-mysql-gke-3](https://github.com/caas-one/news.caas.one/blob/master/translation/images/px-mysql-gke-3-768x306.jpg?raw=true)

例：查找USA中的所有办公室

```
mysql> select `officeCode`, `city`, `phone`  from `offices` where `country` = "USA";
+------------+---------------+-----------------+
| officeCode | city          | phone           |
+------------+---------------+-----------------+
| 1          | San Francisco | +1 650 219 4782 |
| 2          | Boston        | +1 215 837 0825 |
| 3          | NYC           | +1 212 555 3000 |
+------------+---------------+-----------------+
3 rows in set (0.00 sec)
```

退出MySQL shell返回主机。

#### 模拟节点故障

我们通过停止运行MySQL的节点调度来模拟节点故障。

```
$ NODE=`kubectl get pods -l app=mysql -o wide | grep -v NAME | awk '{print $7}'`

$ kubectl cordon ${NODE}
node "gke-gke-px-default-pool-177a3f0b-st0n" cordoned
```

上面的命令禁用了其中一个节点上的调度。

```
$ kubectl get nodes
NAME                                            STATUS                     ROLES     AGE       VERSION
gke-gke-px-default-pool-177a3f0b-nvxm   Ready                          9d        v1.8.10-gke.0
gke-gke-px-default-pool-177a3f0b-slkb   Ready                          9d        v1.8.10-gke.0
gke-gke-px-default-pool-177a3f0b-st0n   Ready,SchedulingDisabled       9d        v1.8.10-gke.0
```

现在继续，删除MySQL pod。

```
$ POD=`kubectl get pods -l app=mysql -o wide | grep -v NAME | awk '{print $1}'`
$ kubectl delete pod ${POD}
pod "mysql-dff54d66d-m9r6q" deleted
```

一旦删除pod，它就被重新定位到具有复制数据的节点上。利用Kubernetes存储协调器(STORK)， Portworx的自定义存储调度程序允许在存储数据的确切节点上共同定位pod。它确保为调度pod选择了合适的节点。

让我们通过运行下面的命令来验证这一点。我们将注意到在不同的节点中创建并调度了一个新的pod。

```
$ kubectl get pods -l app=mysql -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP               NODE
mysql-dff54d66d-tzvjw   1/1       Running   0          15s       192.168.86.169   ip-192-168-95-234.us-west-2.compute.internal
```

```
$ kubectl uncordon ${NODE}
node "ip-192-168-203-81.us-west-2.compute.internal" uncordoned
```

最后，让我们验证数据是否仍然可用。

#### 验证数据是否完整

接下来，找到pod名称并运行“exec”命令，然后访问MySQL shell。

```
kubectl exec -it $POD -- mysql -uroot -ppassword
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.6.40 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

查询数据库验证数据完整性。

```
mysql> USE `classicmodels`;
mysql> select `officeCode`, `city`, `phone`  from `offices` where `country` = "USA";
+------------+---------------+-----------------+
| officeCode | city          | phone           |
+------------+---------------+-----------------+
| 1          | San Francisco | +1 650 219 4782 |
| 2          | Boston        | +1 215 837 0825 |
| 3          | NYC           | +1 212 555 3000 |
+------------+---------------+-----------------+
3 rows in set (0.00 sec)
```

注意，数据库表存在，所有内容都完整。从客户端shell退出，返回到主机。

### 在MySQL上执行存储操作

在测试数据库的端到端故障转移之后，我们接下来在GKE集群上执行存储操作

#### 在不停机的情况下扩展Kubernetes卷

目前我们最初创建的Portworx卷大小为1Gib。我们现在将把它扩大到两倍的存储容量。

首先，我们获取卷名并通过pxctl工具检查。

如果有访问权限，请SSH进入其中一个节点并运行以下命令。

```
$ POD=`/opt/pwx/bin/pxctl volume list --label pvc=px-mysql-pvc | grep -v ID | awk '{print $1}'`
$ /opt/pwx/bin/pxctl v i $POD
Volume	:  150455926773027922
	Name            	 :  pvc-3a6788df-9274-11e8-8c5e-0253036635a0
	Size            	 :  1.0 GiB
	Format          	 :  xfs
	HA              	 :  3
	IO Priority     	 :  LOW
	Creation time   	 :  Jul 28 14:40:52 UTC 2018
	Shared          	 :  no
	Status          	 :  up
	State           	 :  Attached: ip-192-168-95-234.us-west-2.compute.internal
	Device Path     	 :  /dev/pxd/pxd150455926773027922
	Labels          	 :  namespace=default,pvc=px-mysql-pvc
	Reads           	 :  188
	Reads MS        	 :  104
	Bytes Read      	 :  8458240
	Writes          	 :  23
	Writes MS       	 :  128
	Bytes Written   	 :  2347008
	IOs in progress 	 :  0
	Bytes used      	 :  126 MiB
	Replica sets on nodes:
		Set  0
		  Node 		 :  192.168.95.234 (Pool 0)
		  Node 		 :  192.168.203.81 (Pool 0)
		  Node 		 :  192.168.185.157 (Pool 0)
	Replication Status	 :  Up
```

注意当前的Portworx卷。它是1GiB。我们把它扩展到2GiB。

```
$ /opt/pwx/bin/pxctl volume update $POD --size=2
Update Volume: Volume update successful for volume 150455926773027922
```

检查新卷大小。

```
$ /opt/pwx/bin/pxctl v i $POD
Volume	:  150455926773027922
	Name            	 :  pvc-3a6788df-9274-11e8-8c5e-0253036635a0
	Size            	 :  2.0 GiB
	Format          	 :  xfs
	HA              	 :  3
	IO Priority     	 :  LOW
	Creation time   	 :  Jul 28 14:40:52 UTC 2018
	Shared          	 :  no
	Status          	 :  up
	State           	 :  Attached: ip-192-168-95-234.us-west-2.compute.internal
	Device Path     	 :  /dev/pxd/pxd150455926773027922
	Labels          	 :  namespace=default,pvc=px-mysql-pvc
	Reads           	 :  200
	Reads MS        	 :  104
	Bytes Read      	 :  8507392
	Writes          	 :  60
	Writes MS       	 :  164
	Bytes Written   	 :  2498560
	IOs in progress 	 :  0
	Bytes used      	 :  126 MiB
	Replica sets on nodes:
		Set  0
		  Node 		 :  192.168.95.234 (Pool 0)
		  Node 		 :  192.168.203.81 (Pool 0)
		  Node 		 :  192.168.185.157 (Pool 0)
	Replication Status	 :  Up
```

![px-mysql-gke-4](https://github.com/caas-one/news.caas.one/blob/master/translation/images/px-mysql-gke-4-768x480.jpg?raw=true)

### 获取Kubernetes卷快照并恢复数据库

Portworx支持为Kubernetes PVC创建快照。

接下来我们为MySQL创建的Kubernetes PVC创建快照。

```
cat >  px-mysql-snap.yaml << EOF
apiVersion: volumesnapshot.external-storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: px-mysql-snapshot
  namespace: default
spec:
  persistentVolumeClaimName: px-mysql-pvc
EOF
```

```
$ kubectl create -f px-mysql-snap.yaml
volumesnapshot.volumesnapshot.external-storage.k8s.io "px-mysql-snapshot" created
```

验证卷快照的创建。

```
$ kubectl get volumesnapshot
NAME                AGE
px-mysql-snapshot   30s
```

```
$ kubectl get volumesnapshotdatas
NAME                                                       AGE
k8s-volume-snapshot-6ab731c7-9278-11e8-b018-e2f4b6cbb690   34s
```

快照就绪后，我们继续并删除数据库。

```
$ POD=`kubectl get pods -l app=mysql | grep Running | grep 1/1 | awk '{print $1}'`
$ kubectl exec -it $POD -- mysql -uroot -ppassword
```

```
drop database classicmodels;
```

由于快照和卷基本相同，我们可以使用它来启动MySQL的一个新实例。让我们通过恢复快照数据来创建一个MySQL的新实例。

```
$ cat > px-mysql-snap-pvc << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: px-mysql-snap-clone
  annotations:
    snapshot.alpha.kubernetes.io/snapshot: px-mysql-snapshot
spec:
  accessModes:
     - ReadWriteOnce
  storageClassName: stork-snapshot-sc
  resources:
    requests:
      storage: 2Gi
EOF

$ kubectl create -f px-mysql-snap-pvc.yaml
persistentvolumeclaim "px-mysql-snap-clone" created
```

从新的PVC创建MySQL pod。

```
$ cat < px-mysql-snap-restore.yaml >> EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mysql-snap
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql-snap
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: px/running
                operator: NotIn
                values:
                - "false"
              - key: px/enabled
                operator: NotIn
                values:
                - "false"
    spec:
      containers:
      - name: mysql
        image: mysql:5.6
        imagePullPolicy: "Always"
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password       
        ports:
        - containerPort: 3306
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mysql-data
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: px-mysql-snap-clone
EOF
```

```
$ kubectl create -f px-mysql-snap-restore.yaml
deployment.extensions "mysql-snap" created
```

验证新pod是否处于运行状态。

```
$ kubectl get pods -l app=mysql-snap
NAME                         READY     STATUS    RESTARTS   AGE
mysql-snap-5ddd6b6848-bb6wx   1/1       Running   0          30s
```

最后，我们访问本实例前面创建的示例数据。

```
$ POD=`kubectl get pods -l app=mysql-snap | grep Running | grep 1/1 | awk '{print $1}'`
$ kubectl exec -it $POD -- mysql -uroot -ppassword
mysql> USE `classicmodels`;
mysql> select `officeCode`, `city`, `phone`  from `offices` where `country` = "USA";
+------------+---------------+-----------------+
| officeCode | city          | phone           |
+------------+---------------+-----------------+
| 1          | San Francisco | +1 650 219 4782 |
| 2          | Boston        | +1 215 837 0825 |
| 3          | NYC           | +1 212 555 3000 |
+------------+---------------+-----------------+
3 rows in set (0.00 sec)
```

注意，集合仍然存在，数据仍然完整。如果希望在另一个区域创建灾难恢复备份，还可以将快照推送到Amazon S3。Portworx快照还可以与任何S3兼容的对象存储一起工作，因此可以备份到不同的云，或者可以备份到一个本地数据中心。

### 总结

Portworx可以轻松地部署在谷歌Kubernetes引擎上，以在生产环境中运行有状态工作负载。通过集成STORK, DevOps和StorageOps团队可以无缝地在GKE中运行高可用的数据库集群。它们可以为云本地应用程序执行卷扩展、快照、备份和恢复等传统操作。
