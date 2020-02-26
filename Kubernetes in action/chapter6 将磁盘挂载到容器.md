### 本章内容包括
>-  创建多容器pod  
>-  创建一个可在容器间共享磁盘存储的卷
>-  在pod中使用git仓库
>-  将持久性存挂载到pod
>-  使用预先配置的持久性存储
>-  动态调配持久存储
---

之前，我们介绍了pod和与之交互的其他 Kubernetes 资源，即：RC（副本控制器）、RS（副本集）、DS（守护进程集）、Job（作业）和SVC(服务)。  

在某些场景下， 我们可能希望新的容器可以在之前容器结束的位置继续运行，比如在物理机上重启进程。 可能不需要（或者不想要）整个文件系统被持久化， 但又希望能保存实际数据的目录。

Kubernetes 通过定义存储卷来满足这个需求， 它们不像pod 这样的顶级资源，而是被定义为 pod 的一部分， 并和 pod共享相同的生命期。 这意味着在 pod启动时创建卷， 并在删除pod时销毁卷。因此， 在容器重新启动期间， 卷的内容将保持
不变， 在重新启动容器之后， 新容器可以识别前一个容器写入卷的所有文件。 另外，如果一个 pod包含多个容器， 那这个卷可以同时被所有的容器使用。

## 6.1 介绍卷
Kubernetes的卷是pod一个组成部分，因此像容器一样在 pod 的规范中就定义了。 它们不是独立的 Kubernetes 对象，也不能单独创建或删除。 pod 中的所
有容器都可以使用卷，但必须先将它挂载在每个需要访问它的容器中。 在每个容器中， 都可以在其文件系统的任意位置挂载卷。

### 6.1.1 卷的应用示例
例如：一个带有三个容器的 pod一个容器运行了一个 web 服务器，该服务器的HTML页面位于/var/htdoc，并将站点访问日志存储到/var/logs 目录中。第二个容器运行了一个代理来创建HTML文件， 并将它们存放在
/var/html 中；第三个容器处理在 /var/logs 目录中找到的日

每个容器都有一很明确的用途，但是每个容器单独使用就没有多大用处了。在没有共享磁盘存储的情况下，用这三个容器创建一个 pod没有任何意义。 因为内容生成器(content generator) 会在自己的容器中存放生成的 HTML文件， 而 web 服
务器无法访问这些文件， 因为它运行在一个隔离的独立容器内。所以他们三个互相都是隔离的。

但是将两个卷添加到pod中，并在三个容器的适当路径上挂载他们。

通过将相同的卷挂载到两个容器中， 它们可以对相同的文件进行操作。

首先pod下有一个名为publicHtml的卷， 这个卷被挂载在WebServer容器
的/va/htdocs中，因为这是web服务器的服务目录。 在ContentAgent容器中也挂载了相同的卷，通过这种方
式挂载这个卷， web服务器现在将为contentagent生成的内容提供服务。

同样pod还有一个logvol的卷，用于存放日志此卷在WebServer
和LogRotator容器中的/va/log中挂载， 注意，它没有挂载在ContentAgent容器中， 这个容器不能访问它的文件，即使容器和卷是同一个pod的一部分， 在
pod的规范中定义卷是不够的。 如果我们希望容器能够访问它， 还需要在容器的规范中定义一个VolumeMount。


即web容器和log容器共享一个vol，web和agt容器共享一个vol。这样系统就完整了。

>注意，容器挂载了那个卷，才可以访问它的文件。

在本例中， 两个卷最初都是空的， 因此可以使用一种名为emptyDir的卷。Kubernetes还支持其他类型的卷，这些卷要么是在从外部源初始化卷时填充的， 要么是在卷内挂载现有目录。 这个填充或装入卷的过程是在pod内的容器启动之前执行的。

卷被绑定到pod的lifecycle(生命周期）中，只有在pod存在时才会存在， 但取决于卷的类型，即使在pod和卷消失之后， 卷的文件也可能保待原样， 并可以挂载到新的卷中。 让我们来看看卷有哪些类型。

---

### 6.1.2  介绍可用的卷类型
几种可用卷类型的列表：  
- emptyDir——用于存储临时数据的简单空目录
- hostPath——用于将目录从工作节点的文件系统挂载到pod中。
- gitrepo——通过检测Git仓库的内容来初始化的卷
- nfs——挂载到pod中NFS共享卷
...
- configMap、 secret、 downwardAPI——用于将Kubernetes部分资源和集群信息公开给pod的特殊类型的卷。
- persistentVolumeClaim——一种使用预置或者动态配置的持久存储类
型。

单个容器可以同时使用不同类型的多个卷。

---

## 6.2 通过卷在容器之间共享数据

### 6.2.1使用emptyDir卷
顾名思义，卷从一个空 目录开始运行，在 pod 内的应用程序可以写入它需要的任何文件。

---

### 在 pod 中使用 emptyDir 卷

### 创建pod
例如(一个pod中1有两个共用一个卷的容器)：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test1
spec:
  containers:
  - image: 192.168.2.7:5000/go_serv:1.0
    name: ctn1
    volumeMounts:
    - name: vol1
      mountpath: /tmp #名为vol1的卷挂载在容器的/tmp目录下
  - image: 192.168.2.7:5000/go_serv:1.0
    name: ctn2
    volumeMounts: 
    - name: vol
      mountPath: /mnt
      readOnly: true
    ports:
    - containerPort: 9190
      protocol: TCP
  volumes:
  - name: vol1
    emptyDir: {}

```
---

### 指定用于RMPTYDIR的介质
作为卷来使用的emptyDir，是在承载 pod 的工作节点的实际磁盘上创建的，因此其性能取决于节点的磁盘类型。但我们可以通知 Kubernetes 在tmfs 文件系统（即内存上）创建emptyDir。因此可以**将emptyDir的medium字段设置为Memory**：  
```yaml
volumes:
- name: vol1
  emptyDir:
    medium: Memory #enptyDir的文件将会存储在内存中
```

> emptyDir 卷是最简单的卷类型，但是其他类型的卷都是在它的基础上构建的，在创建空 目 录后，它们会用数据填充它。有一种称作gitRepo 的卷类型，

### 6.2.2 使用Git仓库作为存储卷
gitRepo 卷基本上也是一个 emptyDir 卷，它通过克隆 Git 仓库并在 pod 启动时（但在创建容器之前 ） 检出特定版本来填充数据，

> gitRepo 卷是一个 emptyDir 卷， 最初填充了 Git 仓库的内容

>创建 gitRepo 卷后，它并不能和对应 repo 保持同步。 当向 Git 仓库推送新增的提交时， 卷中的文件将不会被更新。  
但是，如果这个pod是由RC管理的，删除这个pod将新建一个新的pod，而这个新的pod的卷将包含最新的提交。

例如：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vol-repo
spec:
  containers:
  - image: 192.168.2.7:5000/go_serv
    name: repo
    volumeMounts:
      name: repo
      volumePath: /tmp
      readOnly: true
  volumes:
  - name: repo
    gitRepo:
      repository: https://github.com/haochen233/words.git
      reversion: master
      directory: . #将repo克隆到卷的根目录

```

总结 gitRepo 存储卷：  
    
    gitRepo容器就像 emptyDir 卷一样， 基本上是一个专用目录， 专门用于包含卷的容器并单独使用。 当pod \被删除时， 卷及其内容被删除。 然而， 其他类型的卷并不创建新目录， 而是将现有的外部目录挂载到 pod的容器文件系统中。 该卷的内容可以保存多个 pod 实例化

---

## 6.3访问工作节点文件系统上的文件

大多数 pod应该忽略它们的主机节点， 因此它们不应该访问节点文件系统上的任何文件。 但是某些系统级别的 pod( 切记， 这些通常由 DaemonSet 管理）确实需要
读取节点的文件或使用节点文件系统来访问节点设备。 Kubemetes通过hostPath卷实现了这一点。

---

### 6.3.1 介绍hostpath卷
hostPath 卷指向节点文件系统上的特定文件或目录在**同一个节点上运行并在其 hostPath 卷中使用相同路径的 pod** 可以看到相同的文件。

hostpath卷是我们介绍的第一种类型的持久性存储，因为emptyDir和gitRepo卷的内容都会在pod被删除时被删除，而hostpath卷的内容则不会被删除。如果删除了一个pod，并且下一个pod使用了指向了主机上相同路径的hostpath卷，则新pod将会发现上一个pod留下的数据，但前提必须将其调度到与第一个pod相同的节点上（即运行在同一个节点上的pod）。

    如果你正在考虑使用hos七Pa七h卷作为存储数据库数据的目录， 请重新考虑。
    因为卷的内容存储在特定节点的文件系统中，所以当数据库pod被重新安排在另一个节点时，会找不到数据。所以这不是一个好主意。

### 6.3.2 检查使用hostpath卷的系统的pod

我们可以查看`kube-system`namespace中运行的pod来查看其中的HostPath类型的卷。

但是当我们检查大多数pod会发现没有一个使用hostpath卷来存储自己的数据，都是使用这种卷来访问节点的数据。

> 提示，请记住仅当需要在节点上读取或写入系统文件时才使用hostpath，切勿使用它们来持久化跨pod的数据。

## 6.4 使用持久化存储
当运行在一个pod中的应用程序需要将数据保存在磁盘上，并且及时该pod重新调度到另一个节点时也要求具有相同的数据可用。这就不能使用到目前为止我们提供的任何卷类型。由于这些数据需要从任何集群节点访问（跨节点），因此必须将其存储在某种类型的网络存储（NAS）中。

---

### 6.4.1 使用GCE持久磁盘作为pod存储卷
略。。
1.首先要创建一个GCE持久磁盘作为pod存储卷
2.然后使用创建一个使用GCE持久磁盘卷的pod

然后就可以存储了，重新创建一个pod后，依然可以读取到前一个pod保存的数据
---

### 6.4.22 通过底层持久化存储使用其他类型的卷

### 使用NFS卷
如果集群运行在自由服务器上，那么就有大量其他可移植的选项用于在卷内挂载外部存储。例如，要挂载一个简单的NFS（network file system）共享，只需指定NFS服务器和共享路径。

```yaml
volumes:
  nfs:
    server: 1.2.3.4  #NFS服务器的IP
    path: /some/path #服务器提供的路径
```

### 使用其他存储技术
要了解每个卷类型设置需要哪些属性的详细信息，可在使用`kubectl explain`或在Kubernetes API定义中了解更多信息。


### 6.5.1 介绍持久卷和持久卷声明
在 Kubemetes集群中为了使应用能够正常请求存储资源， 同时避免处理基础设施细节， 引入了两个新的资源， 分别是待久卷和持久卷声明，

在pod中使用persistentVolume要比要比使用常规的 pod卷复杂一些

当集群用户需要在其 pod 中使用持久化存储时， 他们首先创建持久卷声明（PersistentVolumeClaim，简称PVC），指定所需要的最低容量要求和访问模式，然后用户将待久卷声明清单提交给 Kubernetes API 服务器， Kubernetes 将找到可匹配的持久卷并将其绑定到持久卷声明。

> 简单来说:持久卷由集群管理员提供，并被pod通过持久卷声明来消费。

### 6.5.2创建持久卷
你将首先承担集群管理员的角色， 并创建 一个
支持GCE持久磁盘的持久卷。 然后，你将承担应用程序研发入员的色， 首先声明持久卷， 然后在 pod 中使用。

创建一个gcePeresistentDisk持久卷，清单如下：  
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec: 
  capacity:
    storage: 1Gi
  accessMode: 
  - ReadWriteOnce:
  - ReadOnlyMany:
  persistentVolumeReclaimPolicy: retain 
  gcePersistentDisk:
    pdName: gced
    fsType: ext4
```

> 注意：持久卷不属于任何命名空间， 它跟节点一样是集群层面的资源。


### 6.6.3 不指定存储类的动态配置

### 列出存储类
`kl get sc`

> sc是storageclass的简写

