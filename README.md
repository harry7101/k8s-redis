一、前言
架构原理：每个Master都可以拥有多个Slave。当Master下线后，Redis集群会从多个Slave中选举出一个新的Master作为替代，而旧Master重新上线后变成新Master的Slave。

二、准备操作
本次部署主要基于该项目：

https://github.com/zuxqoj/kubernetes-redis-cluster
1
其包含了两种部署Redis集群的方式：

StatefulSet
Service&Deployment

两种方式各有优劣，对于像Redis、Mongodb、Zookeeper等有状态的服务，使用StatefulSet是首选方式。本文将主要介绍如何使用StatefulSet进行Redis集群的部署。

三、StatefulSet简介
RC、Deployment、DaemonSet都是面向无状态的服务，它们所管理的Pod的IP、名字，启停顺序等都是随机的，而StatefulSet是什么？顾名思义，有状态的集合，管理所有有状态的服务，比如MySQL、MongoDB集群等。
StatefulSet本质上是Deployment的一种变体，在v1.9版本中已成为GA版本，它为了解决有状态服务的问题，它所管理的Pod拥有固定的Pod名称，启停顺序，在StatefulSet中，Pod名字称为网络标识(hostname)，还必须要用到共享存储。
在Deployment中，与之对应的服务是service，而在StatefulSet中与之对应的headless service，headless service，即无头服务，与service的区别就是它没有Cluster IP，解析它的名称时将返回该Headless Service对应的全部Pod的Endpoint列表。
除此之外，StatefulSet在Headless Service的基础上又为StatefulSet控制的每个Pod副本创建了一个DNS域名，这个域名的格式为：

$(podname).(headless server name)   
FQDN： $(podname).(headless server name).namespace.svc.cluster.local
1
2
也即是说，对于有状态服务，我们最好使用固定的网络标识（如域名信息）来标记节点，当然这也需要应用程序的支持（如Zookeeper就支持在配置文件中写入主机域名）。
StatefulSet基于Headless Service（即没有Cluster IP的Service）为Pod实现了稳定的网络标志（包括Pod的hostname和DNS Records），在Pod重新调度后也保持不变。同时，结合PV/PVC，StatefulSet可以实现稳定的持久化存储，就算Pod重新调度后，还是能访问到原先的持久化数据。
以下为使用StatefulSet部署Redis的架构，无论是Master还是Slave，都作为StatefulSet的一个副本，并且数据通过PV进行持久化，对外暴露为一个Service，接受客户端请求。

四、部署过程
本文参考项目的README中，简要介绍了基于StatefulSet的Redis创建步骤：

1.创建NFS存储
2.创建PV
3.创建PVC
4.创建Configmap
5.创建headless服务
6.创建Redis StatefulSet
7.初始化Redis集群

这里，我将参考如上步骤，实践操作并详细介绍Redis集群的部署过程。文中会涉及到很多K8S的概念，希望大家能提前了解学习

1.创建NFS存储
创建NFS存储主要是为了给Redis提供稳定的后端存储，当Redis的Pod重启或迁移后，依然能获得原先的数据。这里，我们先要创建NFS，然后通过使用PV为Redis挂载一个远程的NFS路径。

安装NFS

yum -y install nfs-utils（主包提供文件系统）
yum -y install rpcbind（提供rpc协议）
1
2
然后，新增/etc/exports文件，用于设置需要共享的路径：

[root@ftp pv3]# cat /etc/exports
/usr/local/k8s/redis/pv1 192.168.0.0/24(rw,sync,no_root_squash)
/usr/local/k8s/redis/pv2 192.168.0.0/24(rw,sync,no_root_squash)
/usr/local/k8s/redis/pv3 192.168.0.0/24(rw,sync,no_root_squash)
/usr/local/k8s/redis/pv4 192.168.0.0/24(rw,sync,no_root_squash)
/usr/local/k8s/redis/pv5 192.168.0.0/24(rw,sync,no_root_squash)
/usr/local/k8s/redis/pv6 192.168.0.0/24(rw,sync,no_root_squash)

创建相应目录

[root@ftp quizii]# mkdir -p /usr/local/k8s/redis/pv{1..6}


客户端

yum -y install nfs-utils



创建PV
每一个Redis Pod都需要一个独立的PV来存储自己的数据，因此可以创建一个pv.yaml文件，包含6个PV：

[root@master redis]# cat pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv1
spec:
  capacity:
    storage: 200M
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  nfs:
    server: 192.168.1.3
    path: "/nfs/redis6"





如上，可以看到所有PV除了名称和挂载的路径外都基本一致。执行创建即可：

[root@master redis]#kubectl create -f pv.yaml 
persistentvolume "nfs-pv1" created
persistentvolume "nfs-pv2" created
persistentvolume "nfs-pv3" created
persistentvolume "nfs-pv4" created
persistentvolume "nfs-pv5" created
persistentvolume "nfs-pv6" created

2.创建Configmap
这里，我们可以直接将Redis的配置文件转化为Configmap，这是一种更方便的配置读取方式。配置文件redis.conf如下

[root@master redis]# cat redis.conf 
appendonly yes
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes.conf
cluster-node-timeout 5000
dir /var/lib/redis
port 6379


创建名为redis-conf的Configmap：

kubectl create configmap redis-conf --from-file=redis.conf
1
查看创建的configmap:

[root@master redis]# kubectl describe cm redis-conf
Name:         redis-conf
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis.conf:
----
appendonly yes
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes.conf
cluster-node-timeout 5000
dir /var/lib/redis
port 6379


如上，redis.conf中的所有配置项都保存到redis-conf这个Configmap中。

3.创建Headless service
Headless service是StatefulSet实现稳定网络标识的基础，我们需要提前创建。准备文件headless-service.yml如下：


apiVersion: v1
kind: Service
metadata:
  name: redis-service
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    port: 6379
  clusterIP: None
  selector:
    app: redis
    appCluster: redis-cluster




4.创建Redis 集群节点
创建好Headless service后，就可以利用StatefulSet创建Redis 集群节点，这也是本文的核心内容。我们先创建redis.yml文件：
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-app
spec:
  serviceName: redis-service
  replicas: 6
  selector:
    matchLabels:
      app: redis
      appCluster: redis-cluster
  template:
    metadata:
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      terminationGracePeriodSeconds: 20
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: "redis:5.0.14"
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
          - "--protected-mode"
          - "no"
          - "--cluster-announce-ip"
          - "$(POD_IP)"
        env:
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
        ports:
            - name: redis
              containerPort: 6379
              protocol: "TCP"
            - name: cluster
              containerPort: 16379
              protocol: "TCP"
        volumeMounts:
          - name: "redis-conf"
            mountPath: "/etc/redis"
          - name: "redis-data"
            mountPath: "/var/lib/redis"
      volumes:
      - name: "redis-conf"
        configMap:
          name: "redis-conf"
          items:
            - key: "redis.conf"
              path: "redis.conf"
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      storageClassName: nfs
      accessModes: [ "ReadWriteMany" ]
      resources:
        requests:
          storage: 10M





如上，可以看到这些Pods在部署时是以{0…N-1}的顺序依次创建的。注意，直到redis-app-0状态启动后达到Running状态之后，redis-app-1 才开始启动。
同时，每个Pod都会得到集群内的一个DNS域名，格式为$(podname).$(service name).$(namespace).svc.cluster.local ，也即是：



5.初始化Redis集群
StatefulSet创建完毕后，可以看到6个pod已经启动了，但这时候整个redis集群还没有初始化，需要使用官方提供的redis-trib工具。

我们当然可以在任意一个redis节点上运行对应的工具来初始化整个集群，但这么做显然有些不太合适，我们希望每个节点的职责尽可能地单一，所以最好单独起一个pod来运行整个集群的管理工具。


 
在这里需要先介绍一下redis-trib，它是官方提供的redis-cluster管理工具，可以实现redis集群的创建、更新等功能，在早期的redis版本中，它是以源码包里redis-trib.rb这个ruby脚本的方式来运作的（pip上也可以拉到python版本，但我运行失败），现在（我使用的5.0.3）已经被官方集成进redis-cli中。



 
接下来要获取已经创建好的6个节点的host ip，可以通过nslookup结合StatefulSet的域名规则来查找，举个例子，要查找redis-app-0这个pod的ip，运行如下命令：


 
root@redis-cluster-manager:/# nslookup redis-app-0.redis-service
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	redis-app-0.redis-service.gold.svc.cluster.local
Address: 172.17.0.10
172.17.0.10就是对应的ip。这次部署我们使用0，1，2作为Master节点；3，4，5作为Slave节点，先运行下面的命令来初始化集群的Master节点：

redis-cli --cluster create 10.1.2.244:6379 10.1.2.245:6379 10.1.2.246:6379

然后给他们分别附加对应的Slave节点，这里的cluster-master-id在上一步创建的时候会给出：

redis-cli --cluster add-node 10.1.2.247:6379 10.1.2.244:6379 --cluster-slave --cluster-master-id 603992476513b36cdaac0c09ffb19ff868347985
redis-cli --cluster add-node 10.1.2.248:6379 10.1.2.245:6379 --cluster-slave --cluster-master-id 6c28daf2d2046b620e78ae0e277b0917e3ca7738

 
redis-cli --cluster add-node 10.1.2.249:6379 10.1.2.246:6379 --cluster-slave --cluster-master-id af6abd0066751add765e7bb6c9a27386ba1f82cf

至此，我们的Redis集群就真正创建完毕了，连到任意一个Redis Pod中检验一下：

[root@master redis]# kubectl exec -it redis-app-2 /bin/bash
root@redis-app-2:/data# /usr/local/bin/redis-cli -c
127.0.0.1:6379> cluster nodes
5d3e77f6131c6f272576530b23d1cd7592942eec 172.17.24.3:6379@16379 master - 0 1559628533000 1 connected 0-5461
a4b529c40a920da314c6c93d17dc603625d6412c 172.17.63.10:6379@16379 master - 0 1559628531670 6 connected 10923-16383
368971dc8916611a86577a8726e4f1f3a69c5eb7 172.17.24.9:6379@16379 slave 0025e6140f85cb243c60c214467b7e77bf819ae3 0 1559628533672 4 connected
0025e6140f85cb243c60c214467b7e77bf819ae3 172.17.63.8:6379@16379 master - 0 1559628533000 2 connected 5462-10922
6d5ee94b78b279e7d3c77a55437695662e8c039e 172.17.24.8:6379@16379 myself,slave a4b529c40a920da314c6c93d17dc603625d6412c 0 1559628532000 5 connected
2eb3e06ce914e0e285d6284c4df32573e318bc01 172.17.63.9:6379@16379 slave 5d3e77f6131c6f272576530b23d1cd7592942eec 0 1559628533000 3 connected
127.0.0.1:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:6
cluster_stats_messages_ping_sent:14910
cluster_stats_messages_pong_sent:15139
cluster_stats_messages_sent:30049
cluster_stats_messages_ping_received:15139
cluster_stats_messages_pong_received:14910
cluster_stats_messages_received:30049
127.0.0.1:6379> 


6.创建用于访问Service
前面我们创建了用于实现StatefulSet的Headless Service，但该Service没有Cluster Ip，因此不能用于外界访问。所以，我们还需要创建一个Service，专用于为Redis集群提供访问和负载均衡：

[root@master redis]# cat redis-access-service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: redis-access-service
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    protocol: "TCP"
    port: 6379
    targetPort: 6379
  selector:
    app: redis
    appCluster: redis-cluster


如上，该Service名称为 redis-access-service，在K8S集群中暴露6379端口，并且会对labels name为app: redis或appCluster: redis-cluster的pod进行负载均衡。

创建后查看：

[root@master redis]#  kubectl get svc redis-access-service -o wide
NAME                   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE       SELECTOR
redis-access-service   ClusterIP   10.0.0.64    <none>        6379/TCP   2h        app=redis,appCluster=redis-cluster
1
2
3
如上，在K8S集群中，所有应用都可以通过10.0.0.64 :6379来访问Redis集群。当然，为了方便测试，我们也可以为Service添加一个NodePort映射到物理机上，这里不再详细介绍。

五、测试主从切换
在K8S上搭建完好Redis集群后，我们最关心的就是其原有的高可用机制是否正常。这里，我们可以任意挑选一个Master的Pod来测试集群的主从切换机制，如redis-app-0：

[root@master redis]# kubectl get pods redis-app-0 -o wide
NAME          READY     STATUS    RESTARTS   AGE       IP            NODE            NOMINATED NODE
redis-app-1   1/1       Running   0          3h        172.17.24.3   192.168.0.144   <none>


进入redis-app-0查看：

[root@master redis]# kubectl exec -it redis-app-0 /bin/bash
root@redis-app-0:/data# /usr/local/bin/redis-cli -c
127.0.0.1:6379> role
1) "master"
2) (integer) 13370
3) 1) 1) "172.17.63.9"
      2) "6379"
      3) "13370"
127.0.0.1:6379> 


如上可以看到，app-0为master，slave为172.17.63.9即redis-app-3。

接着，我们手动删除redis-app-0：

[root@master redis]# kubectl delete pod redis-app-0
pod "redis-app-0" deleted
[root@master redis]#  kubectl get pod redis-app-0 -o wide
NAME          READY     STATUS    RESTARTS   AGE       IP            NODE            NOMINATED NODE
redis-app-0   1/1       Running   0          4m        172.17.24.3   192.168.0.144   <none>

我们再进入redis-app-0内部查看：

[root@master redis]# kubectl exec -it redis-app-0 /bin/bash
root@redis-app-0:/data# /usr/local/bin/redis-cli -c
127.0.0.1:6379> role
1) "slave"
2) "172.17.63.9"
3) (integer) 6379
4) "connected"
5) (integer) 13958

如上，redis-app-0变成了slave，从属于它之前的从节点172.17.63.9即redis-app-3。

