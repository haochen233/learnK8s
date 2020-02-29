### 本章内容涵盖
- 部署有状态集群应用
- 为每个副本 pod 实例提供独立存储
- 保证 pod 副本有固定的名字和主机名
- 按预期顺序启停 pod 副本
- 通过DNS SRV记录查询伙伴节点



## 10.1 复制有状态pod
RS通过一个pod模板创建多个pod副本。这些副本除了他们的名字和IP不同外，没有别的差异。

RS 里的所有 pod 共享相同的持久卷声明和持久卷。

---
### 10.1.1 运行每个实例都有单独存储的多副本
问题来了，那如何运行一个pod的多个副本，让每个pod都有独立的存储卷呢?

### 手动创建 pod
可以手动创建多个pod, 每个pod使用一个独立的持久卷声明，但是没有
一个RS在后面对应它们， 所以需要手动管理它们。当有的pod消失后， 需要手动创建它们。 因此这不是一个好的选择。

---
### 一个pod实例对应一个RS
与直接创建不同可以创建多个RS, 每个RS的副本数设为1,
做到pod和RS的一一对应， 为每个RS的pod模板关联一个专属的
持久卷声明，这样就解决了上面手动创建的问题。

但是与单个RS相比，它还是比较笨重。在这种情况下要如何伸缩
pod? 扩容的话， 必须重新创建新的RS 。所以创建多个RS，replicacs为1，也不是一个好的方案。

---
### 使用同一数据卷中的不同目录
一个比较取巧的做法是：所有pod共享同一数据卷， 但是每个pod在数据卷中
使用不同的数据目录。

因为不能在一个pod模板中差异化配置pod副本所以不能指定一个实例使用哪个特定目录！但是可以让每个实例自动选择（或创建）一个别的实例还没有使用的数据目录。

这种方案要求实例之间相互协作， 其正确性很难保证， 同时共享存储
也会成为整个应用的性能瓶颈。

---
### 10.1.2 每个pod都提供稳定的标识
除了上面说的存储需求， 集群应用也会要求每一个实例拥有生命周期内唯一标
识。当一个RS中的pod被替换时，尽管新的pod也可能使用被删掉pod数据卷中的数据， 但它却是拥有全新主机名和IP的崭新pod。

在K8s 中， 每次重新调度一个pod  这个新的pod就有一个新的主机名和IP地址， 这样就要求当集群中任何一个成员被重新调度后， 整个应用集群都需要重新配置。

--- 
### 每个pod实例配置单独的 Service
略。

## 10.2 了解StatefulSet（StS）
可以创建一个sts资源替代RS 来运行这类pod。 它是专门定制的一类应用， 这类应用中每一个实例都是不可替代的个体，都拥有稳定的名字和状态。

无状态的应用（相当于一只蚂蚁），挂掉一个后，并没有什么影响再创建一个新实例就ok了。但是有状态的应用（你心爱的宠物）是唯一的。对应用来说，意味着新的实例需要拥有跟旧的案例完全 一致的状态和标识。

---
### StatefulSet（StS）与RS和RC的对比
RS或RS管理的pod副本比较像牛， 这是因为它们都是无状态的， 任何时候它们都可以被一个全新的pod替换。

而有状态的pod需要不同的方法， 当一个有状态的pod挂掉后， 这个pod实例
需要在别的节点上重建， 但是新的实例必须与被替换的 实例拥有相同的**名称**、**网络标识**和**状态**。这就是sts如何管理pod的（保留pod的唯一）。

sts保证了pod在重新调度后保留它们的标识和状态。它让你方便地扩容、缩容。与RS类似，sts也会指定期望的副本个数， 它决定了在同一时间内运行的有状态的实例（你心爱的宠物）的数量。

与RS类似， pod也是依据sts的pod模板创建的与RS 不同的是， sts创建的pod副
本并不是完全一样的。每个pod都可以拥有一组独立的数据卷（持久化状态）而有所区别。另外你所创建的pod的名字都是有规律的（固定的），而不是每个新pod都随机地获取名字。

---
### 10.2.2 提供稳定的网络标识
一个sts创建的每个pod都有一个从零开始的顺序索引，这个会体现在pod的名称和主机名上，同样还会体现在pod对应的固定存储上。 这些pd的名称则是可预知的，因为它是由sts 的名称加该实例的顺序索引值组成的。 不同于pod随机生成一个名称，这样有规则的pd名称是很方便管理的。

---
### 控制服务介绍
与普通的pd不一样的是， 有状态的pod有时候需要通过其 主机名来定位， 而无状态的pd则不需要，因为每个无状态的pod都是一样的，在需要的时候随便选择一个即可。 但对于有状态的pod来说，因为它们都是彼此不同的（比如拥有不同的状态）， 通常希望操作的是其中特定的一个。

基于以上原因，一个sts通常要求求你创建一个用来记录每个pod网络标记的headless Service 。 通过这个Service, 每个pod将拥有独立的DNS记录，这
样集群里它的伙伴或者客户端可以通过主机名方便地找到它。

比如说，比如说， 一个属于default命名空间， 名为foo的控制服务，它的一个pod名称为A-0,那么可以通过下面的完整域名来访问它：A-0.foo.defualt.svc.cluster.local。而在RS中这是行不通的。

另外，可以通过DNS服务，查找域名foo.default.svc.cluster.local对应的所有SRV记录。获取一个Statefuset中所有pod的名称。

---
### 替换消失的pod（宠物）
当一个sts管理的pod实例消失后，sts保证会重启一个新的pod实例来代替它，不过与RS不同的是，新的pod会于之前消失的pod实例拥有完全一致的名称和主机名。

> sts使用标识完全一致的新的pod替换， Rs则是使用一个不相干的新的 pod替换.

pod运行在哪个节点上并不重要， 新的pod并不一定会调度到相同的节点上。 对于有状态的pod来说也是这样， 即使新的pod被调度到 一个不同的节点， 也同样可以通过主机名来访问。

---
### 扩缩容sts
扩容 一个sts会使用下一个还没用到的顺序索引值创建一个新的pod实
例。

当缩容一个sts时，可以很明确的知道哪个pod将被删除。作为对比RS的缩容操作则不同，不知道那个实例会被删除，也不能指定删除哪个。缩容一个 sts 将会最**先删除索引值最高的实例**。

因为sts缩容任何时候只会操作一个 pod 实例，所以有状态应用的缩容不会很迅速。因此，sts在有实例不健康的情况下是不允许做缩容操作的。

---
### 10.2.3 为每个有状态实例提供稳定的专属存储

### 在pod模板中添加卷声明模板

像sts 创建pod一样，sts也需要创建持久卷声明。所以sts可以拥有一个或多个卷声明模板，这些持久卷声明会在创建 pod 前创建出来， 绑定到一个 pod 实例上。

---
### 持久卷的创建和删除
扩容 sts 增加一个副本数时， 会创建两个或更多的 API 对象（一个pod和与之关联的一个或多个持久卷声明）。但对于缩容来说， 则只会删除一个 pod，而遗留下之前创建的声明。如果声明被删除，与之绑定的持久卷就会被回收或删除，则其上面的数据就会丢失。

因为有状态的 pod 是用来运行有状态应用的， 所以其在数据卷上存储的数据非
常重要，在 sts 缩容时删除这个声明将是灾难性的，特别是对于 sts 来说， 缩容就像减少其replicas数值一样简单。基于这个原因，当你需要释放特定的持久卷时，需要手动删除对应的持久卷声明 。

---
### 重新挂载持久卷声明到相同pod的新实例上
因为缩容 sts 时会保留持久卷声明 ， 所以在随后的扩容操作中， 新的pod实例会使用绑定在持久卷上的相同声明和其上的数据当你因为误操作而缩容一个 sts 后，可以做一次扩容来弥补 自己的过失， 新的 pod 实例会运行到与之前完全一致的状态（名字也是一样的〉。


---
### 10.2.4 Statefulset的保障

Statefulset 不仅拥有稳定的标记和独立的存储，它的 pod 还有其他的一些保障。

### 稳定标识和独立存储的影响
当 K8s 不能确定一个 pod 的状态时。如果它创建一个完全一致的pod，那系统中就会有两个完全一致的 pod 在同时运行。这两个 pod 会绑定到相同的存储，所以这两个相同标记的进程会同时写相同的文件。对于 ReplicaSet 的 pod 来
说，这不是问题，因为应用本来就是设计为在相同的文件上工作的（同一个持久卷）。并且我们知道Rs 会以一个随机的标识来创建 pod ，所以不可能存在两个相同标识的进程同时运行。

---
### 介绍sts的at-most-one的语义
k8s必须保证两个拥有相同标记和绑定相同持久卷声明的有状态pod实例不会运行。一个sts必须保证有状态的pod实例的at-most-one。

也就是说一个 Stat eful set 必须在准确确认一个 pod 不再运行后，才会去创建它的替换 pod 。

---
## 10.3 使用Statefulset

为了恰当的展示sts的行为，将会创建一个小的集群数据存储。没有其他功能，就像石器时代的一个数据存储。

---
### 10.3.1 创建应用和容器镜像
略

---
### 通过sts部署应用

为了部署你的应用， 需要创建两个（或三个） 不同类型的对象：  
- 存储你数据文件的持久卷（当集群不支持持久卷的动态供应时， 需要手动创
建）
- sts必须的一个控制Service
- sts本身

---
### 创建持久化存储卷
gce不支持，就用hostpath：  
两个持久卷：  
```yaml
kind: List
apiVersion: v1
items:
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-a
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    hostPath:
      path: /root/storage/pv1
- apiVersion: v1
  kind: PersistentVolume
  metadata: 
    name: pv-b
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    hostPath:
      path: /root/storage/pv2
```

> 注意：使用---来区分多个资源。也可以创建一个List对象，然后把各个资源资源作为List对象的各个项。效果是一样的。

上面的文件创建了两个持久卷pv-a、pv-b。使用hostPath卷。

---
### 创建控制Service
```yaml
apiVersion: v1
kind: Service
metadata: 
  name: st-s
spec:
  clusterIP: None 
  selector:
    func: sts
  ports:
  - port: 10000
    targetPort: 10000

```

---
### 创建StatefulSet详单
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts
spec:
  serviceName: st-s
  replicas: 2
  template:
    metadata:
      labels:
        func: sts
    spec:
      containers:
      - image: 192.168.2.7:5000/go_serv:1.0
        name: z
        stdin: true
        tty: true
        volumeMounts:
        - name: pv-a
          mountPath: /mnt/pv1
  volumeClaimTemplates:
  - metadata:
      name: pvc1
    spec:
      resources:
        requests:
          starage: 5Mi
      accessModes:
      - ReadWriteOnce
```

---
### 创建sts
命令：`kubectl create -f sts.yaml`

sts会依次启动pod，即上一个pod启动完成后并且处于就绪状态，才会继续启动剩下的。因为多个集群成员同时启动时引起的竞态条件是非常敏感的。所以会这样。


---

### 10.3.3 使用你的pod

### 通过API服务器与pod通信
API服务器的一个很有用的功能就是通过代理直接连接到指定的 pod。如果想请求当前的 sts-0 pod，可以使用如下URL：  
`<apiServerHost>:<port>/api/v1/namespaces/defualt/pods/sts-0/proxy<path>`  
因为 API 服务器是有安全保障的，使用kubectl proxy来与 API 服务器通信`kubectl proxy &`  

---
### 删除一个由状态pod来检查重新调度的pod是否关联了相同的存储

### 扩缩容sts

---
## 10.4 在Statefulset中发现伙伴节点

### 介绍SRV记录


### 10.4.2 更新Statefulset
使用`kubectl edit sts sts_name`，那么可更改replicas或pod模板等等。sts更像RS，更新后不会影响pod，删除旧的pod，就会根据新的模板创建新pod。

---
### 10.4.3 尝试集群数据存储
。。。

## 10.5 了解Statefulset如何处理节点失效

### 10.5.1 模拟一个节点的网络断开
如果节点网络断开，pod会进入terminating状态，不管你修改replicas还是直接delete都不会删除。为什么呢？

这是因为，这个 pod 之前已经被标记为删除，只要它所在节点上的 Kubelet 
通知 API 服务器说这个 pod 的容器己经终止，那么它就会被清除掉。但是此时节点的网络以断开，那么kubelet就不能通知到API 服务器。所以没法删除。

---
### 强制删除pod

使用 `kubectl delete po po_name --force --grace-period 0`使用--force 强制删除，使用--grace-period 0 表示宽容的期限。

