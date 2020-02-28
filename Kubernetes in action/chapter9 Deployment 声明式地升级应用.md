### 本章内容涵盖

- 使用新版本替换 pod
- 升级已有的 pod
- 使用 Deployment 资源升级 pod
- 执行滚动升级
- 出错时自动停止滚动升级 pod
- 控制滚动升级比例
- 回滚 pod 到上个版本


## 9.1 更新运行在pod内的应用程序

新版本的应用打包成镜像， 并将其推送到镜像仓库，接下来想用这个新版本替换所有的 pod
由于 pod在创建之后， 不允许直接修改镜像， 只能通过删除
原有pod并使用新的镜像创建新的 pod替换。
有以下两种方法可以更新所有pod:
- 直接删除所有现有pod，然后创建新的pod
- 也可以先创建新的 pod, 并等待它们成功运行之后， 再删除旧的 pod。可以先创建所有新的 pod, 然后一次性删除所有旧的 pod, 或者按顺序创建新的pod,  然后逐渐删除旧的 pod。

---
这两种方法各有优缺点。第一种方法将会导致应用程序在一定的时间内不可用。
使用第二种方法，你的应用程序需要支持两个版本同时对外提供服务。 如果你的应
用程序使用数据库存储数据， 那么新版本不应该对原有的数据格式或者数据本身进行修改， 从而导致之前的版本运行异常。

---
### 9.1.1删除旧版本pod, 使用新版本pod替换

修改RC中的pod模板，不会影响现有pod，只会影响后来创建的pod。所以可以修改模板后，删除原有pod，然后RC会自动创建新的pod。

如果你可以接受从删除旧的pod到启动新pod之间短暂的服务不可用 那这将是更新一组pod的最简单方式。

---
### 9.1.2 先创建新pod再删除旧版本pod

如果短暂的服务不可用完全不能接受，并且你的应用程序支持多个版本同时对
外服务 那么可以先创建新的pod再删除原有的pod。 这会需要更多的硬件资源，因为你将在短时间内同时运行两倍数量的pod（新的pod与旧pod同时在运行）。

---
### 从新版本立即切换到新版本
pod通常通过Service来暴露。在运行新版本的pod之前， Service 只将流量切
换到初始版本的pod。一旦新版本的pod被创建并且正常运行之后， 就可以修改服务的标签选择器并将Service 的流量切换到新的pod。

>注意，可以使用`kubectl set selector`命令来修改service 的pod选择器。

例如：`kubeclt set selector svc my_svc type=serv-1.0`

---
### 执行滚动升级操作
还可以执行滚动升级操作来逐步替代原有的pod, 而不是同时创建所有新的pod并一并删除所有旧的pod。可以通过逐步对旧版本的 ReplicationController 进行**缩容**并对新版本的进行**扩容**， 来实现上述操作。

在这个过程中， 你希望服务的pod选择器同时包含新旧两个版本的pod, 因此它将请求切换到这两组 pod,手动执行滚动升级操作非常烦琐， 而且容易出错。 根据副本数量的不同， 需要以正确的顺序运行十几条甚至更多的命令来执行整个升级过程。 但是实际上Kubernetes可以实现仅仅通过一个命令来执行滚动升级。

---
## 9.2使用RC实现自动的滚动升级

### 9.2.1 运行第一第个版本的应用

### 创建v1版本的应用
创建一个RC来运行2个go_serv:1.0，并创建一个svc暴露两个pod。

### 9.2.2 使用kubectl 来执行滚动式升级

已经有了v2版本的镜像，需要创建一个v2版本的RS来替换运行v1版本的RS，将新的RS命名为ver2
。

运行`kubectl rolling-update ver1 ver2 --image=192.168.2.7:5000/go_serv:2.0`会立即创建新的RS——ver2，并且ver2的replicas的期望副本数被设置为0。

---
### 了解滚动升级前 kubectl 所执行的操作

kubectl 通过复制vl 的 ReplicationController 并在其 pod模板中改变镜像版本。如果仔细观察控制器的标签选择器， 会发现它也被做了修改。 它不仅包含一个简单的`type=serv-1.0`，还包含一个额外的 deployment标签， 为了
由这个ReplicationController 管理， pod 必须具备这个标签。

需要尽可能避免使用新的和旧的 ReplicationController 来管理
同一组 pod。

---
其实在滚动升级过程中， 第一个 ReplicationController 的选择器也会被修改（选择器会增加deployment标签）。但是这是不是意味着第 一个控制器现在选择不到 pod呢？因为在修改
ReplicationController的选择器之前， kubectl修改了当前 pod（v1版本的pod）的标签（增加了deployment标签）。

---
### 通过伸缩两个 ReplicationController 将旧 pod 替换成新的 pod
修改选择器与标签之后，就会开始替换pod。

一开始v1版本的RC的replicas为2，v2版本的RC的replicas为0。

然后v2增加一个:`scaling v2 up to 1`  
紧接着v1减少一个：`scaling v1 down to 1`  
接着v2增加一个：`scaling v2 up to 2`
最后后v1减少为0：`scaling v1 down to 0`  

整个升级过如上所示。

由于新旧pod都有相同的标签(type=serv-1.0)被服务的选择器匹配到了。
所以在这期间service会将请求同时切换到新旧pod上，直到升级结束，新pod完全替换旧pod，service请求全部被切到新pod上。

---

### 9.2.3 为什么 kubectl rolling-update 已经过时

提示 使用--v 6 选项会提高日志级别使得所有 kubectl 发起的到 API 服务器的请求都会被输出 。

---
伸缩的的请求由 kubectl 客户端执行的，而不是由 Kubemetes master 执行的。

---
提示： 使用详细日志模式运行其他 kubectl 命令， 将会看到 kubectl 和 API服务器之前的更多通信细节。

---
但是为什么由客户端执行升级过程， 而不是服务端执行是不好的呢，rolling-update升级过程看起来很顺利，但是如果在 kubectl 执行升级时失去了网络连接，升级进程将会中断。 pod 和 ReplicationController 最终会处于中间状态。

---
同样， 只需要在pod 定义中更改所期望的镜像 tag, 并让 Kubernetes 用运行新镜像的pod替换旧的pod。正是这一点推动了一种称为Deployment的新资源的引入，这种资源正是现在Kubernetes 中部署应用程序的首选方式。

---
## 9.3 s使用Deployment声明式的升级应用
Deployment是一种更高阶资源， 用千部署应用程序并以声明的方式升级应用，而不是通过 ReplicationController 或 ReplicaSet 进行部署， 它们都被认为是更底层的概念

---
当创建一个Deployment时， ReplicaSet资源也会随之创建（最终会有更多资源被创建）。**RS是新一代的RC并推荐使用它来代替RC来复制和管理pod**。在使用Deployment时， 实际的pod
是由 Deployment的 Replicaset创建和管理的， 而不是由 Deployment 直接创建和管
理的。

---
使用Deployment可以更容易地更新应用程序，因为可以直接定义单个Deployment资源所需达到的状态，并让Kubemetes处理中间的状态。

---
### 9.3.1 创建一个Deployment
创建Deployment和创建RC并没有任何区别。Deployment也是由标签选择器、期望副数和pod模板组成的。此外，它还包含另一个字段，指定一个部署策略，该策略定义在修改Deployment资源时应该如何执行更新。

---
### 创建一个Deployment Manifest

deployment manifest如下：  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zch
spec:
  replicas: 3
  template:
    metadata:
      labels:
        src: deployment
    spec:
      containers:
      - image: 192.168.2.7:5000/go_serv:1.0
        name: c1
        stdin: true
        tty: true
```

>之前的RC只管理了一个特定版本的pod，所以命名就命名为与版本号由关的名字。但是一个Deployment资源高于版本本身。Deployment可以同时管理多个版本的 pod, 所以在命名时不需要命名与版本有关的很方便。

--
### 创建Deployment资源
`kl create -f deploy.yaml --record`

>注意确保在创建时使用了 `--record`选项。 这个选项会记录历史版本号， 在之后的操作中非常有用。

---
### 展示Deployment 滚动过程中的状态

使用get和describe来查看deployment资源

还有另外一个命令， 专门用
于查看部署状态:  
`kubectl rollout status deployment zch`

会显示已完成滚动升级。

会发现pod已经创建、运行好了。

---
### 了解Deployment 如何创建 Replicaset 以及 pod
注意这些 pod 的命名， 之前当使用 ReplicationController 创建pod时， 它们的名称是由 Controller 的名称加上一个运行时生成的随机字符串组成的。

---
在由 Deployment 创建的三个pod名称中均包含一个额外的数字。 那是什么呢？

这个数字实际上对应Deployment 和 ReplicaSet中的 pod模板的哈希值。

Deployment 不能直接管理 pod。相反， 它创建ReplicaSet来管理 pod。查看RS长什么样子。

RS的名称中也包含了其pod模板的哈希值。Deployment会创建多个 ReplicaSet，用来对应和管理一个版本的pod模板。这样使用 pod 模板的哈希值， 可以让 Deployment始终对给定版本的pod 模板创建相同的（或使用已有）的ReplicaSet。

即一个应用版本对应的RS不变。

---
### 通过Service访问pod
略。

### 9.3.2 升级Deployment
与之前rolling-update显示地告诉Kubernetes来执行更新。必须为新的RC指定名称来指定名称来替换旧的资源。在整个过程中必须保持终端处于打开状态。

接下来更新Deployment的方式和上述的流程相比， 只需修改Deployment资源中定义的pod 模板，Kubernetes 会自动将实际的系统状态收敛为资源中定义的状态。

### 不同的Deployment 升级策略

实际上， 如何达到新的系统状态的过程是由 Deployment的升级策略决定的，默认策略是执行滚动更新（策略名为Rollingupdate。）另一种策略为 Recre­ate。会一次性删除所有旧pod，然后创建新的pod。整个行为类似千修改
Replication  Controller的pod 模板， 然后删除所有的pod.

---
### 演示如何减慢滚动升级速度
接下来将使用默认rollingUpdate策略， 但是需要略微减慢滚动升
级的速度， 以便观察升级过程确实是以滚动的方式执行的。

可以通过在Deployment
上设置minReadySeconds属性来实现。使用`kubectl patch deployment zch -p '{"spec": {"minReadySeconds": 10}}'`

>提示kubectl patch 对于修改单个或者少量资源属性非常有用，不需要再通过编辑器编辑。

使用patch命令更改Deployment的自有属性， 并不会导致pod的任何更新，因为pod模板并没有被修改。更改其他Deployment的属性， 比如所需的副本数或部
署策略， 也不会触发滚动升级， 现有运行中的pod也不会受其影响。

---
### 触发滚动升级
要触发滚动升级， 需要将pod镜像修改为v2，

只要修改了deployment中的pod模板就会触发。不管采用哪种方式修改。

例如使用`kubectl set image deployment zch container_name=192.168.2.7:5000/go_serv:1.0`。

然后就开始了滚动升级。与之前的rolling-update方式类似。


---
### Deployment 的优点
仅仅通过更改 Deployment 资源中的pod 模板。应用程序已经
被升级为一个更新的版本一一 仅仅通过更改一个字段而已！

个升级过程是由运行在 Kubernetes上的一个控制器处理和完成的。不再是有客户端执行了，让Kubernetes的控制器接管使得整个升级过程变得更加简单可靠。

>注意：意如果 Deployment 中的 pod 模板引用了 一个 ConfigMap (或 Secret), 那么
更改 ConfigMap 资原本身将不会触发升级操作。如果真的需要修改应用程序的配置并想触发更新的话， 可以通过创建一个新的 ConfigMap 并修改pod 模板引用新的ConfigMap。

---
Deployment 背后完成的整个升级过程和执行 kubectl rolling-update 命令非常相似(新的扩容，旧的收缩)。

与以前不同的是， 旧的RSet 仍然会被保留， 而旧的RC会在滚动升级过程结束后被删除。 之后马上会介绍这个被保留的旧 RS的用处。

---
和处理与维护多个ReplicationController相比， 管理单个Deployment 对象要容易得多。

尽管这种差异在滚动升级中可能不太明显， 但是如果在滚动升级过程中出错或遇到问题， 就可以明显看出两种方案的差异。

### 9.3.3 回滚Deployment
Kubernetes 取消最后一次部署的 Deployment:  
`kubectl rollout undo deployment zch`  

zch会被回滚到上一版本。

>提示 undo 命令也可以在滚动升级过程中运行，并直接停止滚动升级。 在升级
过程中已创建的 pod 会被删除并被老版本的 pod 替代。

---
### 显示 Deployment 的滚动升级历史

`kubectl  rollout history deployment  zch`
还记得创建 beployment 时的－－record 参数吗？如果不给定这个参数，版本历史中的 CHANGE - CAUSE 这一栏会为空。

---
### 回滚到一个特定的 Deployment 版本

`kubectl  rollout  undo  deployment  kubia  - -to-revision=1`

每个 ReplicaSet 都用特定的版本号来保存 Deployment
的完整信息，所以不应该手动删除 ReplicaSet。如果这么做便会丢失 Deployment 的历史版本记录而导致无法回滚。

---
旧版本的 ReplicaSet 过多会导致 ReplicaSet 列表过于混乱,可以通过指定Deployment 的 revisionHistoryLimit属性来限制历史版本数量。默认值是3

只会保留最近的3版（包括当前版本）还早的版本会被删除。

---
###  9.3.4控制滚动升级速率
在 Deployme nt 的 滚动升级期间，有两个属性会决定一 次替换多 少个 pod:max Surge 和 maxUnavailable 。可以通过 Deployment 的 strategy 字 段下
rollingUpdate 的子属性来配置：  
```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable：0
    type: RollingUpdate
```

- maxSurge: 决定了 Deployment 配置中期望的副本数之外，最多允许超出的 pod 实例的数量。默认值为 25% ，所以 pod 实例最多可以比期望数量多 25%。把百分数转换成绝对值时， 会将数字四舍五入。 这个值也可以不是百分数而是绝对值(可以允许最多多出一
个成两个 pod)

- maxUnavailable：决定了在滚动升级期间，相对于期望副本数能够允许有多少 pod 实例处于不可用状态。默认值也是 25% ，所以可用 pod 实例的数量不能低于期望副本数的 75%。与maxSurge 一样，也可以指定绝对值而不是百分比。


### 9.3.5 暂停滚动升级
但是通过 Deployment 自身的
一个选项， 便可以让部署过程暂停， 方便用户在继续完成滚动升级之前来验证新的版本的行为是否都符合预期。

### 暂停滚动升级
在滚动升级的过程中，使用`kubectl rollout pause deployment zch`来暂停滚动升级。

可以在暂停的时候验证新创建的pod是否符合预期（类似金丝雀版本），验证正常工作后，可以将剩余的pod继续升级（恢复升级）或者回滚到上一个版本（取消升级）。

---
### 恢复滚动升级
使用`kubectl rollout resume deployment zch`

在滚动升级过程中， 想要在一个确切的位置暂停滚动升级目前还无法做到， 以后可能会有一种新的升级策略来自动完成上面的需求 。

---
### 使用暂停功能来停止滚动升级
暂停部署还可以用于阻止更新Dploymnt而自动触发的滚动升级过程， 用户可以对Dploymnt进行多次更改， 并在完成所有更改后才恢复滚动升级。一旦更改完毕，则恢复并启动滚动更新过程。

>简单地说，因为更改pod模板就会触发滚动升级，所以当需要多次修改而不想触发升级时，使用暂停（pause），等到修改的差不多时，需要触发滚动升级时，在恢复（resume）就会启动滚动升级过程。

---
**注意：如果部署被暂停，那么在恢复部署之前，撤销命令不会撤销它。（简单地说，你使用了暂停——pause，再使用撤销——undo，undo是不会生效的）**

### 9.3.6 组织出错版本的滚动升级

minReadySeconds的主要功能是避免部署出错版本的应用， 而不只是单纯地减慢部署的速度。

---
minReadySeconds属性指定新创建的pod至少要成功运行多久之后 ，才能将其视为可用。

使用正确配置的就绪探针和适当的minReadySeconds值，Kubernetes将预先阻止我们发布部署带有bug的版本。

---
```yaml
kind: Deployment
...
spec:
  minReadySeconds: 10 #创建的pod成功运行10秒才视为可用
  ...
  containers:
    ...
    readinessProbe:
      periodSeconds: 1 #探测的周期
      httpGet:
        path: /
        port: 8080
```

### 使用kubectl apply 升级Deployment

apply命令可以用YAML文件中声明的字段来更新Dploymnt。不仅更新镜像，而且还添加了就绪探针以及在 YAML 中添加或修改的其他声明。如果新的 YAML也包含replicas字段，当它与现有Dploymnt中的数量不一致时， 那么apply 操作也会对Deploymnt进行扩容。

---
### 就绪探针如何阻止出错版本的滚动升级
在10秒内就绪探针一秒会请求一次，假如有一次请求的时候出现失败。那么pod会从 Service的 endpoint 中移除。所以新的 pod 一直处于不可用状态。

即使变为就绪状态之后，也至少需要保待10秒，才是真正可用的。在这之前滚动升级过程将不再创建任何新的 pod,因为当前maxUnavailable 属性设置为 O, 所以也不会删除任何原始的 pod。

>提示如果只定义就绪探针皮有正确设置minReadySeconds ,  一旦有一次就绪探针调用成功， 便会认为新的pod已经处于可用状态
因此最好适当地设置 minReadySeconds的值。

---
### 为滚动升级配置 deadline

默认情况下， 在10分钟内不能完成滚动升级的话， 将被视为失败
如果运行
kubectl  describe deployment命令， 将会显示一条ProgressDeadline­ Exceeded

判定Deployment滚动升级失败的超时时间， 可以通过设定Deployment 的`spec.progressDeadlineSeconds`字段来指定。

---
### 取消出错版本的滚动升级
`kubectl rollout undo deployment zch`