本章内容涵盖：  
- Kubernetes集群包含哪些组件
- 每个组件的作用以及他们是如何工作的
- 运行的pod是如何创建一个部署对象
- 运行的pod是什么
- pod至今的网络如何工作
- Kubernetes服务如何工作
- 如何保证高可用性


## 11.1 了解架构
Kubernetes集群分为两部分：  
- Kubernetes 控制平面（control plane）
- （工作）节点

---
控制平面的组件：  
- etcd 分布式持久化存储
- API服务器
- 调度器
- 控制器管理器

---
工作节点上运行的组件：  
- Kubelet
- kubelet代理服务（kube-proxy）
- 容器运行时

---
附加组件：  
- Kubernetes DNS 服务器
- 仪表盘
- Ingress 控制器
- Heapster（容器集群监控）
- 容器网络接口插件

---
### 11.1.1 Kubernetes组件的分布式特性

检查控制平面组件的状态，API服务器对外暴露了一个名为ComponentStatus的API资源（简称CS）。

查看控制平面**组件**的状态：  
`kubectl getc componentstatus`

---
### 组件间如何通信
Kubernetes 系统组件间只能通过API服务器通信，他们不会直接通信。
API服务器是和etcd通信的唯一组件。其他组件不会直接和etcd通信，而是通过API服务器来修改集群状态。

API服务器和其他组件的连接基本都是**由组件发起的**，但是当你使用kubectl 获取日志，使用kubectl attach 连接到一个运行中的容器。或运行kubectl port-forward。

>注意，kubectl attach命令和kubectl exec命令类似，区别是：前者会附属到容器中运行着的主进程上，而后者是重新运行一个进程。

---
### 单组件运行多实例
尽管工作节点上的组件都需要运行在同一个节点上， 控制平面的组件可以被简
单地分割在多台服务器上。为了保证高可用性， 控制平面的每个组件可以有多个实
例。etcd和API服务器的多个实例可以同时并行工作， 但是， 调度器和控制器管理器在给定时间内只能有一个实例起作用 ，其他实例处于待命模式。

---
### 组件是如何运行的
控制平面的组件以及kube-proxy可以直接部署在系统上或者作为pod来运行。

Kubelet是唯一一直作为常规系统组件来运行的组件，它把其他组件作为pod来
运行。

---
### 11.1.2 Kubernetes如何使用pod

之前我们创建的所有对象（资源）——pod、RC、RS、Job、DS、PV、PVC、deploy、cm、sts、svc和私密凭据等，需要以持久化方式存储到某个地方，这样它们的mainfest在API服务器重启和失败的时候才不会丢失。为此，Kubernetes使用了etcd。etcd是一个响应快、 分布式、一致的key-value存储 。因为它 是分布式的，故可以运行多个etcd 实例来获取高可用性和更好的性能。

唯一能和etcd通信的是k8s的API 服务器。所有其他组件通过API服务器简洁地读取、写入到etcd。etcd是k8s存储集群状态和元数据的唯一的地方。

---
#### 资源如何存储在 etcd 中
Kubernetes存储所有数据到etcd的/registry下。API服务器将资源的完整
JSON形式存储到 etcd中。 由于etcd的层级键空间，可以想象成把资源以JSON文
件格式存储到文件系统中。

>警告 Kubemetes  I. 7之前的版本，密钥凭据的JSON内容也像上面一样存储（没
有加密）。 如果有人有权限直接访问etcd, 那么可以荻取所有的密钥凭据。 从1.7版本开始， 密钥凭据会被加密， 这样存储起来更加安全。

---

### 确保存储对象的一致性和可验证性
k8s 要求所有控制平面组件只能通过API服务器操作存储模块。使用这种方式更新集群状态总是一致的， 因为 API 服务器实现了乐观锁机制， 如果有错误的话， 也会更少。 API 服务器同时确保写入存储的数据总是有效的，只有授权的客户端才能更改数据。

### 确保etcd集群一致性
为保证高可用性， 常常会运行多个etcd实例。多个 etcd 实例需要保持一致。 这
种分布式系统需要对系统的实际状态达成一致。

etcd 使用 RAFT 一致性算法来保证这一点， 确保在任何时间点， 每个节点的状态要么是大部分节点的当前状态， 要么是之前确认过的状态。

---
### 为什么etcd实例数量应该是奇数
ecd通常部署奇数个实例。你一定想知道为什么。让我们比较有一个实例和有
两个实例的情况时。 有两个实例时， 要求另一个实例必须在线， 这样才能符合超过半数的数量要求。 如果有一个宕机（没有超过半数的实例在线）， 那么ecd集群就不能转换到新状态，

所以两个实例的情况比一个实例的情况更糟。因为一个实例挂的概率是100%，那么整个集群就挂了。如果有2个实例，只要任意挂一个，那么整个集群就挂了，所以一个比两个实例好。

对比三节点与四节点同样的道理。3节点，一个实例宕机，但有2个实例还在正常运行着（超过半数）。但对于四个节点来说，一个宕机，必须要3个节点才能超过半数。所以假设一个实例宕机，4节点需要正常运行的条件比3节点更苛刻（都是挂1个实例，而你需要正常运行3个，而我只需正常运行2个）。

通常。对于大集群，etcd集群由5个或7个节点就足够了。

---
### 11.1.3 API服务器做了什么
k8s API服务器作为中心组件，其他组件或者客户端（kl等）都会去调用它。以RESTful API的形式提供了可以查询、修改集群状态的CRUD（create、Read、Update、Delete）接口。它将状态存储到etcd服务器中。

API服务器除了提供一种一致的方式将对象存储到ecd,也对这些对象做校验，这样客户端就无法存入非法的对象了。除了校验，还会处理乐观锁，对于并发更新的情况，对对象做修改就不会被其他客户端覆盖了。

---
### 通过认证插件认证客户端
首先， API服务器需要认证发送请求的客户端。 这是通过配置在API服务器上的一个或多个认证插件来实现的。API服务器会轮流调用这些插件， 直到有一个能确认是谁发送了该请求。 这是通过检查HTTP请求实现的。

---
### 通过授权插件授权客户端
除了认证插件， API 服务器还可以配置使用一个或多个授权插件。 它们的作用
是决定认证的用户是否可以对请求资源执行请求操作。

例如：创建一个pod，API服务器会轮询所有的授权插件，来确认用户是否可以在请求命名空间创建pod。一旦插件授权插件确认了用户可以执行该操作，API服务器会继续下一步操作。

---
### 通过准入控制插件验证AND/OR修改资源请求
如果请求尝试创建、 修改或者删除一个资源， 请求需要经过准入控制插件的验证。 同理， 服务器会配置多个准入控制插件。这些插件会因为各种原因修改资源。插件甚至会去修改并不在请求中的相关资源， 同时也会因为某些原因拒绝一个请求。 资源需要经过所有准入控制插件的验证。

> 注意：如果请求只是尝试读取数据，则不会做准入控制的验证。

准入控制插件包括：  
- AlwaysPullImages——重写pod的imagePullPolicy为Always，强制每次部署pod时拉取镜像
- ServiceAccount——未明确定义服务的，使用默认账户。
- NamespaceLifecycle——防止在命名空间中就正在被删除的pod，或在不存在的命名空间中创建pod
- ResourceQuota——保证特定命名空间中的pod只能使用该命名空间分配数量的资源，如cpu和内存

---
### 验证资源以及持久化存储
请求通过了所有的准入控制插件后， API服务器会验证存储到etcd的对象， 然
后返回一个响应给客户端。

---
### 11.1.4 API服务器如何通知客户端资源变更
，API服务器没有做其他额外的工作。例如，当你创建一个ReplicaSet资源时，它不会去创建pod, 同时它不会去管理服务的端点。那是控制器管理器的工作。

客户端通过创建到API 服务器的 HTTP 连接来监听变更。通过此连接，客户端
会接收到监听对象的一系列变更通知。每当更新对象，服务器把新版本对象发送至
所有监听该对象的客户端。


kubectl工具作为API服务器的客户端之一，也支持监听资源。  
例如：当部署pod时，不需要重复执行kubectl get  pods来定期查询pod列表。可以使用－－watch标志，每当创建、修改、删除pod时就会通知你，

例如: `kubectl get po --watch` 监听po的增删改事件。

甚至可以让 kubectl 打印出整个监听事件的YAML 文件  
`kubectl get po -o yaml --watch`

---
### 11.1.5 了解调度器
调度器做的就是通过 API 服务器更新 pod的定义。然后 API 服务器再去通知 Kubelet（客户端会监听）该 pod已经被调度过。当目标节点上的Kubelet发现该
pod被调度到本节点， 它就会创建并且运行pod的容器。

---
### 默认的调度算法
选择节点操作可以分解为两部分：  
- 过滤所有节点，找出能分配给pod的可用节点列表
- 对可用节点按优先级排序， 找出最优节点。 如果多个节点都有最高的优先级
分数， 那么则循环分配， 确保平均分配给 pod。

---
### 查找可用节点
为了决定哪些节点对pod可用，调度器会给每个节点**下发**一组配置预测函数。这些函数检查：  
- 节点是否否能满足pod对硬件资源的请求
- 节点是否耗尽资源
- pod是否要求被调度到指定节点（通过名字）， 是否是当前节点？
- 节点是否有和pod规格定义里的节点选择器一致的标签（如果定义了的话）
- 如果pod要求绑定指定的主机端口，那么这个节点上的这个端口是否已经 被占用？
- 如果pod要求有特定类型的卷， 该节点是否能为此pod加载此卷， 或者说该
节点上是否已经有pod在使用该卷了？
- pod是否能够容忍节点的污点。
- pod是否定义了节点、pod的亲缘性以及非亲缘性规则？如果是， 那么调度节
点给该pod是否会违反规则？

所有这些测试都必须通过， 节点才有资格调度给pod。在对每个节点做过这些
检查后， 调度器得到节点集的一个子集 （可用节点集合）。 任何这些节点都可以运行pod, 因为它们都有足够的可用资源， 也确认过满足pod定义的所有要求。

---
### 为pod选择最佳节点
如有2个节点都能运行pod，但是一个运行10个pod，另一个没有1个。明显调度器会选第二个节点。

---
### pod高级调度
假设一个pod有多个副本 。理想情况下， 你会期望副本能够分散在尽可能多的节点上， 而不是全部分配到单独一个节点上 。即使单个节点宕机，并不会对服务造成影响。

默认情况下， 归属同一服务和ReplicaSet的pod会分散在多个节点上。但不保
证每次都是这样。不过可以通过定义pod的亲缘性、 非亲缘规则强制pod**分散**在集群内或者**集中**在一起。

---
### 使用多个调度器
可以在集群中运行多个调度器。然后，对每一个pod，可以通过在pod特性中设置schedulerName属性指定调度器来调度特定的pod。  未设置该属性的pod由默认调度器调度default-scheduler。其他设置了了该属性的 pod会被默认调度器忽略掉， 它们要么是手动调用， 要么被监听这类 pod 的调度器调用。

可以实现自己的调度器， 部署到集群， 或者可以部署有不同配置项的额外
Kubemetes 调度器实例。

---
### 11.1.6 介绍控制器管理器中运行的控制器
控制器包括：  
- RC管理器
- RS、DS以及Job的管理器1
- Deployment控制器
- sts控制器
- Node控制器
- Service控制器
- Endpoingts控制器
- Namespace 控制器
- PersistentVolume控制器
- 其他
每个控制器做什么通过名字显而易见。 通过上述列表， 几乎可以知道创建每个
资源对应的控制器是什么。 资源描述了集群中应该运行什么， 而控制器就是活跃的
Kubernetes组件， 去做具体工作部署资源。

---
### 了解控制器做了什么以及如何做的
控制器做了许多不同的事情， 但是它们都通过API 服务器监听资源变更。


### Replication 管理器
启动 ReplicationController资源的控制器叫作 Replication 管理器。

可以理解为一个无限循环，
每次循环， 控制器都会查找符合其 pod 选择器定义的pod 的数量， 并且将该数值和期望的复制集 (replica) 数量做比较。

### RS、DS以及Job控制器
RS控制器基本与Replication管理器做了一样的事。DS和Job控制器从各自pod模板中创建pod资源。这些控制器不会运行pod，而是将pod定义发布到API服务器上，让kubelet（监听API Server）创建容器并运行。

### Deployment控制器

Deployment 控制器负责使 deployment 的实际状态与对应 Deployment API 对象
的期望状态同步。每次 Deployment 对象修改后（如果修改会影响到部署的 pod), Deployment 控制器都会滚动升级到新的版本。 通过创建一个 ReplicaSet ，然后按照 Deployment 中定义的策略同时伸缩新、旧 RelicaSet ， 直到旧 pod 被新的代替。 并不会直接创建任何 pod 。


### StatefulSet控制器
类似于RS控制器以及其他控制器。根据资源sts资源创建、管理、删除pod。其他的控制器只管理pod，而sts控制器会初始化并管理每个pod实例的持久卷声明字段


### Node控制器
Node 控制器管理 Node 资源，描述了集群工作节点中， Node 控制器使节
点对象列表与集群中实际运行的机器列表保持同步。 同时监控每个节点的健康状态，删除不可达节点的 pod 。

Node控制器不是唯一对Node对象做更改的组件。kubelet也可以做更改。

### Service控制器
Service 控制器就是用来在 负载均衡 类型服务被创建或删除时， 从基础设施
服务请求、释放负载均衡器的。


### Endpoint 控制器
Service 不会直接连接到 pod ， 而是包含一个端点列表（Endpoint），该列表要么是手动，要么是根据Service定义的pod选择器自动创建、更新。Endpoint控制器作为活动的组件，定期根据匹配标签选择器的 pod 的 IP 、端口更新端点列表。

请记住， Endpo int 对象是个独立的对象，所以当需要的时候控制器会创建它。 同样地，当删除 Serv i ce 时， Endpoint 对象也会被删除。

---
### namespace控制器
当删除一个 Namespace 资源时，该命名空间里的所有资源都会被删除。 这就是 Namespace 控制器做的事情。 当收到删除 Namespace 对象的通知时，控制器通过API 服务器删除所有归属该命名空间的资源。

### PersistentVolume控制器
一旦用户创建了 一个持久卷声明，Kubemetes 必须找到一个合适的持久卷同时将其和声明绑定。这些由持久卷控制器实现。

对于一个持久卷声明，控制器为声明查找最佳匹配项，通过选择匹配声明中的
访问模式，并且声明的容量大于需求的容量的最小持久卷。实现方式是保存一份有序的持久卷列表

当用户删除持久卷声明时，会解绑卷，然后根据卷的回收策略（persistentVolumeReclaimPolicy）进行回收（原样
保留、删除或清空）。

---
### 唤醒控制器
再一次强调，所有这些控制器是通过 API 服务器来操作 API 对象的。它们不会直接和 Kubelet 通信或者发送任何类型的指令。

控制器更新API 服务器的一个资源后，Kubelet 和 Kubernetes Service Proxy（订阅了资源更新的通知）（并不知道控制器到的存在）。


### 11.1.7 kubelet做了什么
所有k8s控制平面的控制器都运行在主节点上。而kubelet以及service proxy都运行在工作节点（pod容器运行的地方）。

### 了解kubelet的工作内容
简单地说，Kubelet 就是负责所有在工作节点上运行的内容的组件。它第一个任务
就是在 API 服务器中创建一个 Node 资源来注册该节点。然后需要持续监控 API 服务器是否把该节点分配给 pod ，然后启动 pod 容器。具体实现方式式告知容器运行时（docker等）来从特定容器镜像运行容器。kubelet随后持续监控运行的容器，向API服务器报告它们的状态、事件、资源消耗。

kubelet也是运行容器存活探针的组件，当探针报错时它会重启容器。最后一点，
当 pod 从 API 服务器删除时， Kubelet 终止容器，并通知服务器 pod 己经被终止了 。

---
### 抛开API服务器运行静态pod
尽管 Kubelet 一般会和 API 服务器通信并从中获取 pod 清单，它也可以基于本
地指定目录下的 pod 清单来运行 pod 。

---
### 11.1.8 Kubernetes Service Proxy 的作用
除了kubelet，每个工作节点还会运行kube-proxy，用于确保客户端可以通过
Kubemetes API 连接到你定义的服务。kube-proxy 确保对服务 IP 和端口的连接最终能到达支持服务的某个 pod 处。如果有多个 pod 支撑一个服务，那么代理会发挥对 pod 的负载均衡作用。

---
### 为什么被叫作代理
kube-proxy最初实现为userspace代理，利用**实际的服务器**集成接收连接，同时代理给pod。为了拦截发往服务IP（即提供服务的pod的IP）的连接，代理配置了iptables规则（该规则是一个管理Linux内核数据包过滤功能的工具），重定向连接到代理服务器。

kube-proxy之所以叫这个名字是因为它确实就是一个代理器， 不过当前性能更
好的实现方式仅仅通过iptables规则重定向数据包到一个随机选择的后端pod,（即直接通过该规则将数据包重定向到一个后端的pod，而不是之前将数据包发送到代理服务器，然后在通过iptables过滤规则重定向到pod）。

---
### 11.1.9 介绍k8s插件
上面已经讨论了k8s集群正常工作所需要的一些核心组件。

还有一些插件，不是必须的；这些插件用于启用k8s服务的DNS查询，通过单个外部IP地址暴露多个HTTP服务、k8s web 仪表盘等特性。

---
### 如何部署插件
通过提交AML清单文件到API服务器，这些组件会成为插件并作为pod部署。有些组件是通过Deployment资源或者ReplicationController资源部署的，有些是通过DaemonSet 。

---
### DNS服务器如何工作
集群中的所有pod默认配置使用集群内部DNS服务器。 这使得pod能够轻松
地通过名称查询到服务，甚至是无头服务pod 的IP地址。

DNS服务pod 通过kube-dns服务对外 暴露，使得该pod能够像其他pod 一样在集群中移动。kube-dns pod利用API服务器的监控机制来订阅Service和Endpoint 的变动，以及DNS记录的变更，使得其客户端（相对地）总是能够获取到最新的DNS信息。

客观地说，在Service和Endpoint资源发生变化到 DNS pod 收到订阅通知时间点之间，DNS记录可能会无效。

### Ingress控制器如何工作
根据集群中定义的 Ingress、Service 以及 Endpoint 资源来配置该控制器。所以需要订阅这些资源（通过监听机制）， 然后每次其中一个发生变化则更新代理服务器的配置。

尽管 Ingress 资源的定义指向一个 Service, Ingress控制器会直接将流量转到服
务的 pod 而不经过服务IP。当外部客户端通过 Ingress控制器连接时， 会对客户端IP 进行保存， 这使得在某些用例中， 控制器比Service 更受欢迎。

---
## 11.2 控制器如何协作

### 11.2.1 了解设计哪些组件
在你启动整个流程之前， 控制器、 调度器、 Kubelet就已经通过 API服务器监
听它们各自资源类型的变化了。

---
### 11.2.2 事件链
准备包含Deployment 清单的yaml文件，通过kubectl提交到API服务器API服务器检
查Deployment定义，存储到etcd,返回响应给kubect让现在事件链开始被揭示出来，过程如下：  
1. kubectl 发送包含Deployment清单的yaml文件，请求创建depoly资源。API服务器检查清单中的deployment定义，检查无误后，存储到etcd中，返回响应给Deployment控制器。
2. Deployment检查到有新的deploy对象时，Deployment向API服务器发送创建ReplicaSet的请求。API服务器弧通知ReplicaSet控制器有新的RS对象
3. 新的RS对象被ReplicaSet控制器接收，控制器会考虑replica数量、ReplicaSet中定义的pod选择器，然后检查是否有足够的满足选择器的pod。然后控制器会基千ReplicatSet的pod模板创建 pod资源（当Deployment控制器创建RS时，会复制该Deployment的pod模板）。**同样的创建的请求会发送到API 服务器中**
4. 目前新创建的pod保存在etcd中，但是它们每个都缺少一个重要的东西－它
们还没有任何关联节点。它们的nodeName属性还未被设置。调度器会监控像这样
的pod（没有关联任何节点的pod）,发现一个pod，就会为pod选择最佳节点，并将节点分配给 pod。pod的 定义**现在**就会包含它应该运行在哪个节点（即nodeName）。
> 目前，所有的一切都发生在k8s控制平面中（control plane）中。

5. Kubelet通过API服务器监听 pod变更，发现有新的pod分配 到本节点后， 会去检查pod定义，然后命令任何使用的 容器运行时（比如docker）来启动pod容器，器运行时就会去运行容器。

---
### 11.2.3 观察集群事件
控制平面 组件和Kubelet执行动作时，都会发送事件给API服务器。发送事件是通过创建事件资源来实现的，事件资源和其他的Kubemetes资源类似。每次使用
kubectl  describe来检查资源的时候，就能看到资源相关的事件，也可以直接
用`kubectl get events`获取事件。

---
## 11.3 了解运行中的pod是什么
查看创建一个pod时实际运行的docker是什么情况，使用`docekr ps`命令查看所有正在运行的容器。

比如，我们创建了一个pod，然后利用查看docker ps 命令查看，会发现不仅有我们自己的pod容器，还有一个附加容器，附加容器没有做任何事（容器命令是“pause”）。该附加容器，在pod的前面创建。它的作用是什么呢？

被暂停的容器将一个pod所有的容器收纳到一起。还记得一个pod的所有容器是如何共享同一个网络和Linux命名空间的吗？暂停的容器是一个基础容器， 它的唯一 目的就是保存所有的命名空间。 所有pod的其他用户定义容器 使用pod的该基础容器的命名空间。

比如一个双容器pod有3个运行的容器，共享用一个linux命名空间。

实际的应用容器可能会挂掉然后重启，当容器重启，容器需要处于与之前相同的Linux命名空间中。 基础容器使这成为可能，因为它的生命周期和pod绑定， 基础
容器pod被调度直 到被删除一 直会运行 。 如果基础pod在这期间被关闭，Kubee会 重新创建它， 以及pod的所有容器。

简单地来说，基础容器保证该pod内的所有容器共享相同的Linux命名空间。

---
## 11.4 跨pod间网络
Kubernetes并不会要求你使用特定的网络技术，但是pod用于通信的网络必须是：pod自己认为的IP地址一定和所有其他节点认为该pod拥有的IP地址（更准确的说是pod中容器的地址）。

即pod的间的网络必须是平坦的网络，不能经过NAt，即两个pod之间直接（互相看到对方的地址就是对方的IP地址）进行通信。保证pod内部应用网络的简洁性。

---
### 11.4.2 深入了解网络工作原理

### 同节点pod通信
基础设施容器启动之前， 会为容器创建一个虚拟Ethernet（以太网）接口对，其中一个对的接口保存在主机的命名空间中（veth...），而其他的对被移入到容器网络命名空间，并重命名为eth0。两个虚拟接口就像管道的两端，从一端进，另一端出来。

主机网络命名空间的接口会绑定到容器运行时配置使用的网络桥接上。 从网桥
的地址段中取IP地址赋值给容器内的etO接口。

如果podA发送网络包到pod B,  报文首先会经过podA的veth 对到网桥然后经过podB的veth 对。节点上的所有容器都会连接这个网桥。意味着他们都能够相互通信。

但是要让运行在不同节点上的容器之间能够通信， 这些节点的网桥需要以某种方式连接起来。

---
### 不同节点上的pod通信
跨整个集群的pod的IP地址必须是唯一的，所以跨节点的网桥必须使用非重叠地址段。防止不同节点的pod拿到同一个IP。（即子网数大点）。例如master节点的网桥使用10.244.0.0地址段，而Node1节点的网桥使用10.244.1.0地址段，这样就保证了跨节点pod的IP地址唯一性。


按照该配置， 当报文从一个节点上容器发送到其他节点上的容器， 报文先通过veth pair  通过网桥到节点物理适配器， 然后通过网线传到其他节点的物理适配器，再通过其他节点的网桥， 最终经过vethpair到达目标容器。

仅当节点连接到相同网关、 之间没有任何路由时上述方案有效。 否则， 路由器会扔包因为它们所涉及的podIP是私有的。
flannel
---
### 11.4.3 引入容器网络接口
为了让连接容器到网络更加方便，启动一个项目容器网络接口(CNI)。 CNI允许Kubemetes可配置使用任何CNI插件。 这些插件包含：  
- Calico
- flannel
- Romana
- Weave Net
- 其他
安装一个网络插件 并不难。只要部署一个包含DS以及支持其他资源的Yaml。每个插件项目首页都会提供一个这样的Yaml文件。DS往往用于在集群的所有节点部署一个网络代理，然后会绑定CNI接口节点。

需要kubelet需要用 --network-plugin=cni命令启动才能适用CNI。

---
### 11.5 服务时如何实现的

### 11.5.1 引入kube-proxy
和Service相关的任何 事情都由每个节点上 运行的kube-proxy 进程处理。开始
的时候 kube-proxy确实是一个proxy, 等待连接 对每个进来的连接 连接到一个
pod。这称为userspace (用户空间）代理模式。后来 性能 更好的iptables代
理模式取代 了它。iptables代理模式目前是默认的模式。

每个Service有其自己稳定的IP地址和端口 。客户端（通常为pod)通过连接该 IP 和端口使用服务。IP地址是虚拟的 没有被分配给任何网络接口 当数据包离开节点时也不会列为 数据包的源 或目的IP地址Service的一个 关键细节是 它们包含一个IP、端口对 （或者针对多端口Service有多个IP、端口对）所以服务 IP本身并不代表任何东西。这就是为什么你 不能够ping它们。

---
### 11.5.2 kube-proxy如何使用iptables规则
当客户端pod发送数据包时，发生了什么。  

包目的地初始设置为服务（svc）的IP和端口。在发送到网络之前，pod客户端所在的节点的内核会根据配置在该节点上的iptables规则处理数据包。

内核会检查数据包是否匹配任何这些iptables规则。其中有个规则规定如果有任何数据包的目的地是否匹配任何这些iptables规则。其中有个规则规定如果有任何数据包的目的地IP和端口等于服务的IP和端口。那么数据包的目的IP和端口应该被替换为随机选中的pod的IP和端口。

实际上可能比描述的还复杂点。

---
## 11.6 运行高可用集群
在k8s上运行应用的一个理由就是，保证运行不被中断，或者说尽量少地人工介入基础设施导致的宕机。了能够不中断地运行服务， 不仅应用要一直运行，Kubemetes控制平面的组件也要不间断运行。 接下来我们了解一下达到高可用性需要做到什么。

---
### 11.6.1 让你的应用变得高可用
 为了保证你的应用的高可用性， 只需通过Deployment资源运行应用， 配置合适数量的复制集， 其他的交给Kubernetes处理。

 ### 运行多实例来减少宕机可能性
 需要你的应用可以水平扩展， 不过即使不可以， 仍然可以使用Deployment, 将复制集数量设为1。 如果复制集不可用，会快速替换为一个新的，尽管不会同时发生。让所有相关控制器都发现有节点宥机、 创建新的pod复制集、 启动pod容器可能需要一些时间。 不可避免中间会有小段宕机时间。

---
 ### 对不能水平扩展的应用使用领导选举机制
 为了避免宅机， 需要在运行一个活跃的应用的同时再运行一个附加的非活跃复制集， 通过一个快速起效租约或者领导选举机制来确保只有一个是有效的。 以防你不熟悉领导者选举算法， 提一下， 它是一种分布式环境中多应用实例对谁是领导者达成一致的方式。 例如， 领导者要么是唯一执行任务的那个， 其他所有节点都在等待该领导者宕机， 然后自己变成领导者；或者是都是活跃的， 但是领导者是唯一能够执行写操作的， 而其他的只能读数据。 这样能保证两个实例不会做同一个任务，否则会因为竞争条件导致不可预测的系统行为。

 该机制自身不需要集成到应用中，可以使用一个 sidecar 容器来执行所有的领导选举操作， 通知主容器什么时候它应该活跃起来。

保证应用高可用相对简单，因为 Kubemetes 几乎替你完成所有事情。 但是假如Kubernetes 自身若机了呢？如果是运行 Kubemetes 控制平面组件的服务器挂了呢？这些组件是如何做到高可用的呢？

---
### 11.6.2 让k8s控制平面变得高可用
为了使得 Kubernetes 高可用， 需要运行多个主节点master。

### 运行etcd集群
因为 etcd 被设计为一个分布式系统， 其核心特性之一就是可以运行多个 etcd 实
例，所以它做到高可用并非难事。 你要做的就是将其运行在合适数量的机器上（数量最好是奇数）。

使得它们能够互相感知。 实现方式通过在每个实例的配置中包含其他实例的列表。例如，当启动一个实例时，指定其他 etcd 实例可达的 IP 和端口 。

etcd会跨实例复制数据，所以3节点其中一个宕机并不会影响处理读写操作。

---
### 运行多实例API服务器
保证 API 服务器高可用甚至更简单，因为 API 服务器是（几乎全部）无状态的（所有数据存储在 etcd 中， API 服务器不做缓存），你需要多少就能运行多少 API 服务器，它们直接不需要感知对方存在。通常，一个 API 服务器会和每个etcd 实例搭配。这样做，etcd 实例之间就不需要任何负载均衡器，因为每个 API 服务器只和本地 etcd 实例通信。

而API服务器确实需要一个负载均衡器（将连接均衡分配到每个健康的API服务器），这样客户端总是连接到健康的API服务器实例。

---
### 确保控制器和调度器的高可用性
对比 API 服务器可以同时运行多个复制集，运行控制器管理器或者调度器的多实例情况就没那么简单了。因为控制器和调度器都会积极地监听集群状态，发生变更时做出相应操作，可能未来还会修改集群状态（例如，当 ReplicaSet 上期望的复制集数量增加 l 时 ， ReplicaSet 控制器会额外创建一个 pod ），多实例运行这些组件会导致它们执行同一个操作，**会导致产生竞争状态**，从而造成非预期影响（如前例提到的，创建了两个新 pod 而非一个〉。

由于这个原因，当运行这些组件的多个实例时，给定时间内只有一个实例有效。幸运的是，这些工作组件自己都做了（ 由一leader-elect 选项控制，默认为 true ）。只有当成为选定的领导者时， 组件才可能有效。只有领导者才会执行实际的工作，而其他实例都处于待命状态，等待当前领导者宕机。当领导者宕机，剩余实例会选举新的领导者， 接管工作。这种机制确保不会出现同一时间有两个有效组件做同样的工作。

控制器管理器和调度器可以和 API 服务器、 etcd 搭配运行，或者也可以运行在不同的机器上。当搭配运行时，可以直接跟本地 API 服务器通信； 否则就是通过负载均衡器连接到 API 服务器。

### 控制平面使用的领导选举机制
我发现最有趣的是：选举领导时这些组件不需要互相通信。领导选举机制的实
现方式是在 API 服务器中创建一个资源，而且甚至不是什么特殊种类的资源－－
Endpoint 资源就可以拿来用于达到目的（滥用更贴切一点）

使用 Endpoint 对象来完成该工作没有什么特别之处。使用Endpoint 对象的原因是只要没有同名 Service 存在，就没有副作用。 也可以使用任何其他资源（事实上，领导选举机制不会就使用 ConfigMap 来替代 Endpoint ）。


让我们以调度器为例。 所有调度器实例都会尝试创建（之后更新）一个Endpoint 资源，称为kube-scheduler 。 可以在 kube-system 命名空间中找到它。

使用kl get命令查看用于选举的`kube-scheduler` Endpoint资源。会在annotation字段，看搭配k8s.../leader注解的值中有holderIdentity字段，该字段包含了当前领导者的名字。第一个将名字填入该字段的实例成为领导者。实例之间会竞争，但是最终只有一个胜出。

一旦成为领导者，必须顶起更新资源（默认每 2 秒），这样所有其他的实例就
知道它是否还存活。当领导者若机，其他实例会发现资源有一阵没被更新了，就会
尝试将自己的名字写到资源中尝试成为领导者。简单吧，对吧？