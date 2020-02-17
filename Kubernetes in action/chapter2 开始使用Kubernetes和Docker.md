## 2.1 创建、运行及共享容器镜像
。。。

## 2.2配置kubernetes集群

## 在Kubernetes上运行第一个应用
运行我们的httpserv:`kubectl run 命令`  
例如:`kubectl run serv --image=httpserv:latest --port=9190 --generator=run/v1`  


**介绍pod**  
你或许在想，是否有一个列表显示所有正在运行的容器，可以通过类似于
**kuberctl  get  containers** 的命令获取。 这并不是 Kubernetes 的工作，它不直
接处理单个容器。 相反，它使用多个共存容器的理念。 这组容器就叫作 pod 。  

一个pod是一组紧密相关的容器它们总是一起运行在同一个工作节点上，以
及同一个 Linux 命名空间中。每个 pod 就像一个独立的逻辑机器，拥有自己的 IP 、
主机名、进程等，运行一个独立的应用程序。  

细读下图可以更好的理解容器、pod和节点之间的关：  
![容器、pod、节点理解](https://haochen233.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E5%BA%8A/pod.png?Expires=1581921299&OSSAccessKeyId=TMP.hifNroPG9W8uEKU79kRvLjmeJh189enREfgWVwQ9t3TSkGkBnHNwCNEWJreR5owXAKrBWotkZiHE2vTWSDUb431AchcYDHX8ZY3XP2NEMNXdEEMwsbbWAEv97oc82a.tmp&Signature=BqY2x9QA5l9hd6Ia2DzbFXpH70M%3D)  

**列出pod**  
命令`kubectl get pods`  
要查看有关pod的更多信息，使用命令`kubectl get pods`,如果 pod 停留在挂起状态，那么可能是 Kubemetes 无法从镜像中心拉取镜像。如果你正在使用自己的镜像，确保它在 Docker Hub 上是公开的。  

**幕后发生的事情**  
Kubernetes中运行容器镜像的步骤：  
1. 当运行kubectl 命令，它通过向Kubernetes API服务器发送一个 REST HTTP请求，在集群master节点上创建一个新的Replicationcontroller对象（应答控制对象）。  
2. 然后该对象创建一个新的pod，调度器将其调度到一个工作节点上。
3. 在新pod调度到的工作节点上。kubelet看到新的pod被调度到本节点上会告知docker从镜像中心拉取指定的镜像。然后根据此镜像创建并运行容器。  

流程如下图：  
![在Kubernetes中运行 luksa/kubia 容器镜像](https://haochen233.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E5%BA%8A/%E5%9C%A8Kubernetes%E4%B8%AD%E8%BF%90%E8%A1%8C%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F.png?Expires=1581923980&OSSAccessKeyId=TMP.hifNroPG9W8uEKU79kRvLjmeJh189enREfgWVwQ9t3TSkGkBnHNwCNEWJreR5owXAKrBWotkZiHE2vTWSDUb431AchcYDHX8ZY3XP2NEMNXdEEMwsbbWAEv97oc82a.tmp&Signature=ZjXw71pceBgBei3d6GWpzvj47ck%3D)  

### 2.3.2 如何访问正在运行的pod？  
每个pod都有自己的IP地址，但是这个地址是集群内部的，不能从集群外部访问。要让 pod 能够从外部访问，需要通过服务对象公开它，要创建一个特殊的 **LoadBalancer** 类型的服务。因为如果你创建
一个常规服务（一个Cluster IP 服务）， 比如 pod ，它也只能从集群内部访问。将创建一个外部的负载均衡，可以通过负载
均衡的公共 IP 访问 pod 。  

**创建一个服务对象**  
要创建服务，需要告知 Kubemetes 对外暴露之前创建的 ReplicationController:  `$ kubectl expose rc serv --type-LoadBalancer --name http_serv`  
这里用的是 replicatio口controller的缩写 rc。大多数资源类型都有这样的缩写，所以不必输入全名（如，pods 的缩写是 po, service 的缩
写是 SVC ，等等 ） 。  

列出服务：`kubectl get services` 或`kubectl get svc`。  

仔细查看创建的
http-serv 服务。 它还没有外部 IP 地址，因为 Kubernetes 运行的云基础设施创建负载均衡需要一段时间。负载均衡启动后，应该会显示服务的外部 IP 地址。 让我们等待一段时间并再次列出服务。

### 2.3.3 系统的逻辑部分  

**pod和它的容器**  
pod有自己独立的私有IP地址和主机名。  
**ReplicationControl的角色**  
 ReplicationControl确保始终存在一个运行中的pod实例。  
 通常，ReplicationControl用于复制pod（即创建pod的多个副本）并让他们保持运行。如果你的pod因为任何原因消失了，那么ReplicationControl将创建一个新的pod来替换消失的pod。  

 **为什么需要服务**  
 pod的存在是短暂的，一个pod可能会在任何时候消失，或许因为它所在节点发生故障， 或许因为有人删除了pod, 或者因为pod被从一个健康的节点剔除了。 当其中任何一种情况发生时， 如前所述， 消失的pod 将被ReplicationContro  I  !er替换为新的pod。新的pod 与替换它的pod 具有不同的IP地址。这就是需要服务的地方 解决不断变化的pod IP地址的问题，以及在一个固定的IP 和端口对上对外暴露多个pod。  
 当一个服务被创建时，  它会得到一个静态的 IP, 在服务的生命周期中这个IP不会发生改变。客户端应该通过固定IP地址连接到服务， 而不是直接连接pod。服
务会确保其中一个pod接收连接， 而不关心 pod 当前运行在哪里（以及它的 IP地址是什么）。  
服务表示一组或多组提供相同服务的pod的静态地址。到达服务IP和端口的请求将被转发到属于该服务的一个容器的IP和端口。  

### 2.3.4 水平伸缩应用
现在有了一个正在运行的应用，由ReplicationController 监控并保持运行，并通过服务暴露访问。 现在让我们来创造更多魔法。  
使用Kubenetes 的一个主要好处是可以简单地扩展部署。 让我们看看扩容 pod有多容易。 接下来要把运行实例的数量增加到三个。  
pod 由一个 ReplicationController 管理。 让我们来查看kubectl get 命令：`kubectl get replicationcontrollers`  
DESIRED项，渴望的副本数。
> **使用kubectl get列出所有类型的资源**  
&emsp;&emsp;一直在使用相同的基本命令 kubectl get 来列出集群中的资源。你已经使用此命令列出节点、 pod、服务和 ReplicationController 对象
不指定资源类型调用 kubectl get可以列出所有可能类型的对象。这些类型可以使用各种kubectl命令，例如get、describe等。当然资源名称可以使用缩写（例如：pod缩写为po、replicationcontroller为rc、service为svc，等等）  

**增加期望的副本数**  
为了增加pod的副本数，需要改变ReplicationController期望的副本数，如下所示：  
`kubectl scale rc serv --replicas=3`  
命令告诉Kubernetes需要确保pod始终有三个实例在运行。  

现在已经告诉 Kubernetes需要确保pod 始终有三个实例在运行。 注意， 你没有
告诉 Kubernetes需要采取什么行动， 也没有告诉 Kubernetes 增加两个 pod, 只设置新的期望的实例数量并让Kubernetes 决定需要采取哪些操作来实现期望的状态。  
这是Kubernetes 最基本的原则之一。不是告诉 Kubemetes 应该执行什么操作，而是声明性地改变系统的期望状态， 并让 Kubernetes 检查当前的状态是否与期望的
状态一致。 在整个 Kubernetes 世界中都是这样的。  

正如你所看到的，给应用扩容是非常简单的，一旦应用在生产中运行并且需要扩容，可以使用一个命令添加额外的实例， 而不必手动安装和运行其他副本。记住， 应用本身需要支持水平伸缩。 Kubemetes 并不会让你的应用变得可扩展，
它只是让应用的扩容或缩容变得简单。  

**当切换到服务时请求切换到所有三个pod上**    
例如应用为自己搭建的httpserv，http请求会随机的切换到不同的pod。当pod有多个实例时Kubernetes服务就会这样做。无论服务后面是单个pod 还是一组pod, 这些 pod在集群内创建、消失，
这意味着它们的IP地址会发生变化， 但服务的地址总是相同的。 这使得无论有多少
pod,  以及它们的地址如何变化， 客户端都可以很容易地连接到 pod。  

**可视化系统的新状态**  
如下图：  
![rc管理并通过服务暴露的pod的三个实例](https://haochen233.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E5%BA%8A/%E7%94%B1%E5%90%8C%E4%B8%80rc%E7%AE%A1%E7%90%86%E5%B9%B6%E9%80%9A%E8%BF%87%E6%9C%8D%E5%8A%A1IP%E5%92%8C%E7%AB%AF%E5%8F%A3%E6%9A%B4%E9%9C%B2%E7%9A%84pod%E7%9A%84%E4%B8%89%E4%B8%AA%E5%AE%9E%E4%BE%8B.png?Expires=1581933582&OSSAccessKeyId=TMP.hhnuns8CjSMmD7ymaQdcar9UvDBMRPeMfqP4w6Atrw8yvnEEshUerZA38qTwd6fe8AbQBeDRstHbMy71QKPLRZmPJ7MLFEweKrmmVwBJyw9L4DaSt7RqMqHaKkJrHM.tmp&Signature=dh%2BLTFXjYHmgjkfj05JoSe2embM%3D)  

### 2.3.4 查看应用运行在哪个节点上
不管调度到哪个节点， 容器中运行的所有应用都具有相同类型的操作系统。 每
个pod都有自己的IP, 并且可以与任何其他pod通信， 不论其他pod是运行在同一
个节点上， 还是运行在另一个节点上。 每个pod都被分配到所需的计算资源， 因此
这些资源是由一个节点提供还是由另一个节点提供， 并没有任何区别。  

**列出pod时显示的podIP和pod节点**
以使用-o wide选项请求显示其他列。 在列出pod时， 该选项显示pod
的IP和所运行的节点：  
```shell
$ kubectl get pods -o wide  
NAME      READY   STATUS    RESTART AGE   IP    NODE
```

**使用kubectl describe 查看 pod 的其他细节**  


### 2.3.6 介绍kubernetes dashboard  
到目前为止， 只使用kubectl 命令行工具。 如果更喜欢图形化的 web用户界面，你会很高兴地听到 Kubemetes 也提供了一个不错的 web dashboard。  
