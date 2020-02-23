pod代表了Kubemetes中的基本部署单元，而且你已知道如何手动创建、 监督和管理它们。 但是在实际的用例里， 你希望你的部署能自动保待运行， 并且保持健康， 无须任何手动干预。要做到这一点， 你几乎不会直接创
建pod, 而是创建ReplicatonContolle r或Deployment这样的资源， 接着由它们来创
建并管理实际的pod。  

当你创建未托管的pod（直接创建的pod）时，会选择一个集群节点来运行pod,然后在该节点上运行容器。Kubernetes接下来会监控这些容器， 并且在它们失败的时候自动重新启动它们。

但是如果整个节点失败， 那么节点上的pod会丢失， 并且不会被新节点替换， 除非这些pod由前面提到的ReplicationController或类似资源来管理。

## 4.1 保持pod健康
如果容器的主进程崩溃， Kubelet将重启容器。  

### 4.1.1 介绍存活探针
Kubemetes可以通过**存活探针** (liveness probe) 检查容器是否还在运行。 可以
为pod中的每个容器单独指定存活探针。 如果探测失败， Kubemetes将定期执行探
针并重新启动容器。  

Kubernetes有以下三种探测容器的机制：
- HTTP GET探针对容器的IP地址（你指定的端口和路径）执行HTTP GET 请求。如果服务器返回错误响应状态码（非2xx或3xx）或者根本没有响应，那么探测就被认为是失败的，容器将被重新启动。
- TCP套接字探针尝试与容器指定端口建立TCP连接。如果连接成功建立，则
探测成功。否则，容器重新启动。  

- Exec探针在容器内执行任意命令，并检查命令的退出状态码。如果退出状态码是0，则探测成功，其他状态码都被认为失败。  

### 4.1.2 创建基于http的存活探针
创建一个包含HTTP GET存活探针的新pod，配置manifest如下：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - image: go_serv:1.0
  name: test-serv
  livenessProbe:
    httpGet:
      path: /
      port: 8080
```

### 4.1.3 使用存活探针
然后根据yaml创建pod，然后用describe查看pod探针的描述。

>**获取崩溃容器的应用日志**
如果你的容器重启，`kubectl logs`命令将显示当前容器的日志。当你想知道为什么前一个容器重启时，你需要看前一个容器的日志，而不是当前容器的。可以通过添加`--previous`选项完成。  

通过`kubectl describe`命令，查看pod详细信息。`kubectl describe`还显示关于存活探针的附加信息，每次探测都会发送http请求，如果返回错误状态码，那么Kubernetes探测后就会重启容器。

>**注意** 当容器被强行终止后，会创建一个全新的容器——而不是重启原来的容器。  

### 4.1.4 配置存活探针的附加属性
`kubectl describe`可以看到有关存活探针相关的信息，除了明确指定的存活探针选项，还可以看到其他属性，例如delay(延迟）、timeout(超时）、period(周期）等。 delay=0s表示容器启动后立即开始探测。timeout仅设置为1秒，即容器必须在1秒内响应，不然这次探测记作失败。period=10s说明每10秒探测一次。还有连续探测3次后（#failure=3）重启容器。  

定义探针时可以自定义这些附加参数。例如设置初始延迟，将`initialDelaySeconds`属性添加到存货探针的配置中。例如：  
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 15 #r容器启动后15秒后开始探测
``` 

如果没有设置延迟，存活探针会在启动时立即开始探测容器，这通常会导致探测失败。因为应用程序还没有准备好开始接受请求，如果失败次数超过阅值，在应用程序能正确响应请求之前，容器就会重启。  

>**提示** 务必设置一个初始延迟来说明应用程序的启动时间

例如：很多场合都会看到这种情况，用户很困惑为什么他们的容器正在重启。 但是如果使用kubect1 describe,  他们会看到容器以退出码137或143结束， 并告诉他们该pod是被迫终止的。 此外，pod事件的列表将显示容器因liveness探测失败而被终止。 如果你在pod启动时看到这种情况， 那是因为未能适当设置
initialDelaySeconds。  

### 4.1.5 创建有效的存活探针

对于在生产中运行的pod,一定要定义 一个存活探针。 没有探针的话，Kubernetes无法知道你的应用是否还活着。 只要进程还在运行， Kubernetes会认为容器是健康的。  


**存活探针应该检查什么**

但为了更好地进行存活检查， 需要将探针配置为请求特定的URL路径（例如 /health）,并让应用从内部对内部运行的所有重要组件执行状态检查，以确保它们都没有终止或停止响应。  

> 提示请确保/health HTTP瑞点不需要认证， 否则探剧会一直失败，导致你的容器无限重启。

**保持探针轻量**  
存活探针不应消耗太多的计算资源， 并且运行不应该花太长时间。默认情况下，探测器执行的频率相对较高， 必须在一秒之内执行完毕。 一个过重的探针会大大减慢你的容器运行。  

**无须在探针中实现重试循环**  
你已经看到， 探针的失败阙值是可配置的， 并且通常在容器被终止之前探针必须失败多次。 但即使你将失败阙值设置为1 ,  Kubemetes为了确认一次探测的失败，会尝试若干次。因此 在探针中自己实现重试循环是浪费精力。  

**存活探针小结**  
现在知道了Kubernetes会在你的容器崩溃或其存活探针失败时， 通过重启容器来保持运行。 这项任务由承载pod的节点上的Kubelet执行——在主服务器上运行的KubernetesControl Plane组件不会参与此 过程。  

## 4.2 了解ReplicationController（副本控制器）

ReplicationController是一种Kubernetes资源，可确保它的pod始终保持运行状态。如果pod因任何原因消失（例如节点从集群中消失或由于该pod已从节点中逐出），则ReplicationController 会注意到缺少了pod并创建替代pod。

**ReplicationController**旨在创建和管理多个pod的多个副本（replicates）。  

节点故障时，只有ReplicationController管理的pod会被重新创建。  

### 4.2.1 ReplicationController的操作
RC（ReplicationController是我的俗称）会持续监控正在运行的pod列表，并保证相应 ”类型”的pod的数目与期望相符 如正在运行的pod太少， 它会根据pod模板创建新的副本。如正在运行的pod太多，它将删除多余的副本。多余的副本可能有这几个原因：  
- 有人手动创建相同类型的pod
- 有人更改现有的pod的“类型”
- 有人减少了所需的pod的数量（scale），等等

pod的“类型”这种说法是不存在的。RC不是根据pod类型来执行操作的，而是根据pod是否匹配某个标签选择器。  


**介绍控制器的协调流程**

RC的工作是确保pod的数量始终与其标签选择器匹配。如果不匹配， 则ReplicationController将根据所需， 采取适当的操作来协调pod的数量。  


**了解RC的三个组成部分**

1. label selector（标签选择器），用于确定RC的作用域有哪些pod。
2. replica count（副本数量），指定运行的pod数量。
3. pod template（pod模板），用于创建新的pod。

> ReplicationController的副本个数、标签选择器，甚至是 pod模板都可以随时修改，但只有副本数目的变更会影响现有的 pod。  

**更改控制器的标签选择器或 pod 模板的效果**  
更改标签选择器和pod模板对现有 pod 没有影响。 更改标签选择器会使现有的pod脱离 RC的范围，因此控制器会停止关注（监控）它们。

在创建 pod后，RC 也不关心其pod的实际 “内容”（容器镜像、 环境变量及其他）。因此， 该模板仅影响由此RC创建的新 pod。

使用RC的好处：  
- 确保一个pod（或多个pod副本的持续运行）

- 集群节点发生故障，它将为故障节点上运行的所有pod（受RC监控（关注）的pod创建替代副本）。

- 它能轻松实现pod的水平伸缩（scale）。

> **注意** pod实例永远不会重新安置到另一个节点。相反，RC会创建一个全新的pod实例，它与正在替换的pod实例无关。  

### 4.2.2 创建一个ReplicationController
可以通过上传YAML描述文件到Kubernetes API 服务器来创建RC。

Yaml文件如下：
```yaml
apiVersion: v1
kind: ReplicationController
metadate:
  name: zch
spec:
  replicas: 2
  select:
    type: clnt
  template: 
    metadate:
      labels:
        owner: rc
    spec:
      containers:
      - name: rc-test
        image: go_serv
```

上传文件到API服务器时，Kubernetes回创建一个名为zch的RC，它确保符合标签选择器owner=rc的pod实例始终是2个（replicas是2）。**模板中的pod标签必须和RC中的标签选择器匹配，否则控制器将会无休止的创建新容器**。

根本不指定标签选择器。在这种情况下，它会自动根据pod模板中的标签自动配置。 

> **提示** 定义RC时不要指定pod选择器，让Kubernetes从pod模板中提取它。这样YAML更简短。  

创建RC使用`kubectl create命令`


### 4.2.3 使用RC

获取有关RC的信息：`kubectl get rc`。  

获取有关RC的详细信息：`kubectl describe rc rc_name`。

**控制器如何创建新的pod**  

略。。。

**应对节点故障**  

### 4.2.4 将 pod 移入或移出 ReplicationController 的作用域

由RC创建的pod并不是绑定RC。RC管理与标签选择器匹配的pod。 通过更改pod的标签， 可以将它从RC的作用域中添加或删除。 它甚至可以从一个RC移动到另一个RC。

**给RC管理的pod添加标签**  
需要确认的是， 如果你向 ReplicationController 管理的 pod 添加其他标签，它并不关心。

**更改已托管的pod的标签**
如果被RC托管的pod，更改与RC标签选择器匹配的标签的话，就会造成已匹配的pod数量减少，RC会根据缺少的数量自动启动新的pod，已达到设置规模。  

被更改后的这个pod，成为了完全独立的pod，会一直运行，知道你手动删除它。  

**更改RC的标签选择器**
如果更改的是RC的标签选择器，而不是pod的标签，那么所有的pod会脱离RC的管理，并且RC会重新创建多个全新的pod。Kubernetes确实允许你更改RC的标签选择器。记住，你永远不会更改控制器的标签选择器，但你时不时会更改它的pod模板。  

### 4.2.5 修改pod模板
RC的pod模板可以随时修改。只影响之后创建的 pod, 并且不会影响现有的 pod。


可以用以下命令`kubectl edit rc rc_name`编辑对应的RC。

### 4.2.6 水平缩放

RC 扩容和缩放，可以使用`kubectl scale`命令增加或减少replicas。当然还可以通过编辑RC的定义对其进行扩容`kubectl deit rc rc_name`,更改RC.spec.replicas 字段就可以进行扩容或缩放。  

**伸缩集群的声明式方法**  
在 Kubernetes 中水平伸缩pod是陈述式的：＂我想要运行x 个实例。”你不是告诉 Kubernetes做什么或如何去做， 只是指定了期望的状态。

### 4.2.7 删除一个ReplicationController
**当你通过 `kubectl delete` 删除 ReplicationController 时， pod也会被删除。***

由于由 ReplicationController 创建的 pod不是ReplicationController 的组成部分，只是由其进行管理， 因此可以只删除ReplicationController并保待 pod运行。

当你最初拥有一组由RC管理的pod, 然后决定用ReplicaSet(你
接下来会知道）替换RC 时， 这就很有用。 可以在不影响 pod 的情
况下执行此操作，并在替换管理它们的 RC 时保待 pod不中断运行。  

使用`--cascade=false`删除RC时托架不受管理。可以通过给命令增加`--cascade=false`选项来保持pod的运行。

例如：`kubectl delete rc rc_name --cascade=false`

你已经删除了该RC，所以这些pod独立了，他们不再被管理。但是你始终可以使用当前的标签选择器创建新的RC，并且再次将他们管理起来。


## 4.3 使用ReplicaSet而不是ReplicationController

最初， ReplicationController 是用于复制和在异常时重新调度节点的唯一Kubernetes 组件，后来又引入了 一个名为ReplicaSet的类似资源 。 它是新一代的
ReplicationController,  并且将其完全替换掉（RC最终将被弃用）。

另外， 你仍然可以看
到使用中的RC, 所以你最好知道它们。 也就是说从现在起， 你应该始终创建 ReplicaSet 而不是 ReplicationController。它们几乎完全相同， 所以你不会碰到任何麻烦。

你通常不会直接创建它们， 而是在创建更高层级的 Deployment资源时自动创建他们无论如何， 你应该了解ReplicaSet, 所以让我们看看
它们与ReplicationController的区别。


### 4.3.1 比较ReplicaSet和RC
RS（ReplicaSet的简称）的行为和RC完全相同。但pod选择器的表达能力
更强。虽然RC的标签选择器只允许包含某个标签的匹配 pod,但RS的选择器还允许匹配缺少某个标签的 pod,或包含特定标签名的 pod不管其值如何。

单个RC无法将pod与标签（例如env=production和env=devel）同时匹配。它只能匹配一组，但是RS可以匹配两组并将他们视为一个大组。

RC无法基于标签名（只有名，不在意值）来匹配pod。而RS可以，例如可以匹配所有包含env标签的pod，即可以理解为`env=*`，即只要有env标签。

### 4.3.2 定义RS
将之前创建的一个RC.yaml文件进行修改，既可以改写为RS的新文件。例如：  
```yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: zch-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      owner: zch-rs
  template:
    metadata:
      labels:
        owner: zch-rs
    spec:
      nodeSelector: 
        func: test
      containers:
      - name: zch-rs
        image: 192.168.2.7:5000/alpine:1.0
        stdin: true
        tty: true
```


### 4.3.3 创建和检查ReplicaSet

使用get 和 describe 命令可以查看rs（ReplicaSet）


### 4.3.4 使用ReplicaSet的的更富表达力的标签选择器

RS相对于RC的主要改进是它更具表达力的标签选择器，在4.2.2中创建的rs中，我们使用的是较简单的matchLabels选择器，这与RC并没有区别，现在，使用使强大的
matchExpressions 属性来重写选择器，修改部分的manifest如下：  
```yaml
selector:
  matchExpressions:
  - key: type 
    operator: In
    values:
    - serv
```

会标签匹配器会匹配标签type=serv的pod。

每个标签表达式必须包含一个key、一个operator（运算符），并且可能还有一个values的列表。有四个有效的运算符：  
- In： label的值必须与其中一个指定的values匹配。
- NotIn: label的值必须与任何指定的values不匹配。
- Exists： pod必须包含一个指定名称的标签（值不重要）。使用此运算符，不指定values字段。
- DoesNotExist：即Pod不包含指定名称标签。values属性不得指定。如果指定多个表达式（matchExpressions），则所有表达式必须都为true才能使选择器与pod匹配。

如果同时指定matchLabels和matchcExpressions，则所有标签都必须匹配，并且所有表达式必须计算为true才能使该pod与选择器匹配。


## 4.4 使用DaemonSet在每个节点上运行一个pod

RC和RS都用于在Kubernetes集群上运行部署特定数量的pod。但是，当你希望pod在集群中的每个节点上运行时（每个节点都需要一个正好运行的pod实例），就会出现某些情况。  

这些情况包括pod执行系统级别的与基础结构相关的操作。例如， 希望在每个节点上运行日志收集器和资源监控器。另 一个典型的例子是Kubernetes 自己的kube-proxy进程， 它需要运行在所有节点上才能使服务工作。

DaemonSet在每个节点上只运行一个pod副本，而副本则将他们随即地分布在整个集群中。  

### 4.4.1 使用DaemonSet在每个节点上运行一个pod

要在所有集群节点上运行一个pod 需要创建一个DaemonSet对象， 这很像一个RC或RS,除了由DaemonSet创建的pod，已经有一个指定的目标节点并跳过Kubernetes调度程序。他们并不是随即分布在集群上的。  

DaemonSet确保创建足够的pod，并在自己的节点上部署每个pod。

尽管RS（或RC）确保集群中存在期望数量的pod副本，但DaemonSet并没有期望的副本数的概念。它不需要， 因为它的工作是确保一个pod匹配它的选择器并在每个节点上运行。  

如果节点下线DaemonSet不会在其他地方重新创建pod。但是，当将一个新节点添加到集群中时，DaemonSet会立即部署一个新的pod的实例。如果有人无意删除了一个pod，那么它也会重新创建一个新的pod。和RS一样，DaemonSet从配置的pod模板创建pod。  

### 4.4.2 使用 DaemonSet 只在特定的节点上运行 pod

DaemonSet将 pod 部署到集群中的所有节点上，除非指定这些 pod 只在部分节点上运行。这是通过 pod 模板中的 nodeSelector 属性指定的这是 DaemonSet定义的一部分


创建一个DaemonSet yaml，通过节点选择器，在特定节点上部署pod：  
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: zch-ds
spec:
  selector:
    matchLabels:
      owner: zch-ds
  template:
    metadata:
      labels:
        owner: zch-ds
    spec: 
      nodeSelector:
        func: test
      containers:
      - name: ds
        image: 192.168.2.7:5000/daemon-test:1.0
        stdin: true 
        tty: true
```
使用`kubectl create -f `创建即可，记住删除Daemon-set，同时也会一起删除这些pod。

## 4.5 运行指向单个任务的pod
到目前为止， 我们只谈论了需要持续运行的 pod。你会遇到只想运行完成工作
后就终止任务的清况。RC、RS和DS会持续运行任务，永远达不到完成状态，这些pod的进程会在退出时重新启动。但是在一个可完成的任务中，其进程终止后，不应该再重新启动。

### 4.5.1 介绍job资源

Kubernetes 通过Job资源提供了对此的支持，它允许你运行一种pod，该pod在内部进程成功结束时，不重启容器，一旦任务完成，pod就被认为处于完成状态。
在发生节点故障时，该节点上由 Job管理的pod将按照 ReplicaSet的pod的方式，
重新安排到其他节点。如果进程本身异常退出（进程返回错误代码时），可以将job配置为重新启动容器。

Job对于临时任务很有用，关键是任务要以正确的方式结束。可以在未托管的pod中运行任务并等待它完成，但是为托管的pod在节点故障时或被节点逐出时，则需要手动创建该任务。手动做这件事并不合理——特别是如果任务需要几个小时才能完成。

### 4.5.2 定义Job资源
按照下面的代码清单创建Job manifest。
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: zch-job
spec: 
  template: 
  metadata:
    labels:
      owner: zch-job
  spec:
    restartPolicy: OnFailure #Job不能使用Always作为默认的重新启动策略
    containers:
    - name: zch-job
      image: 192.168.2.7:5000/ubuntu:1.0
      stdin: true
      tty: true
```

在容器中执行一个脚本，sleep2分钟后打印一个mission successed。
在一个pod的定义中，可以指定在容器中运行的今结束时，Kubernetes会做什么，这是通过pod配置的属性restartPo巨cy完成的， 默认为Always。Job
pod不能使用默认策略， 因为它们不是要无限期地运行。因此， 需要明确地将重启
策略设置为OnFa辽ure或Never。此设置防止容器在完成任务时重新启动(pod
被Job管理时并不是这样的）。

### 4.5.3 看Job运行一个pod

使用`kuebctl create -f `创建job后，应该看到它立即启动一个pod：
`kubectl get jobs`.
`kubectl get po`

等任务执行成功后，工作将被标记为已完成。默认情况下，除非使用--show-all（或-a） 选项，否则在列出po时不显示已经完成的pod：
`kubectl get po --show-all`或`kubectl get po -a`**注意v17.3 不能使用**

完成后pod未被删除的原因是允许你查阅其日志。

pod可以直接删除，或者在删除创建它的Job时被删除。

### 4.5.4 在ob中运行多个pod实例

作业可以配置为创建多个pod实例，并以并行（同时运行）或串行（按顺序一个个来）方式运行。这是通过在Job配置中设置completions和parallelism属性来完成的。

**顺序运行Job pod**
如果需要一个Job运行多次，则可以将completions设为你希望作业的pod运行多少次。
例如：
```yaml
apiVersion: batch/v1
kind: 
metadata:
  name:multi-completion-job
spec:
  completions: 5
  template: 
    <同上例>
```

Job将一个接一个地运行五个pod。它最初创建一个pod, 当pod的容器运行完
成时，它创建第二个pod, 以此类推，直到五个pod成功完成。如果其中一个pod
发生故障，工作会创建一个新的pod, 所以Job总共可以创建五个以上的pod。

**并行运行Job pod**
不必一个接一个地运行单个Job Pod，也可让该Job并行运行多个pod。可以铜鼓parallelism Job配置属性，指定允许多少个pod并行执行。

例如接上例，允许两个pod并行执行：
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-job2
spec:
  completions: 5
  parallelism: 2 #最多允许两个pod可以并行
  。。。上同


```

主要一个pod完成任务，工作将运行下一个pod，知道五个pod都完成任务。

**Job的缩放**
你甚至可以在 Job 运行时更改 Job 的parallelism属性。这与缩放RS和RC类似。同样使用`kubectl scale`命令完成：

例如将parallelism属性更改为3个：`kubectl scale job job_name --replicas=3`

由于你将parallelism 从 2 增加到 3, 另一个pod立即启动， 因此现在有三
个pod在运行。

### 4.5.5 限制Job pod完成任务的时间

关于 Job我们需要讨论最后一件事。 Job要等待一个pod多久来完成任务？如
果pod卡住并且根本无法完成（或者无法足够快完成）， 该怎么办？

通过在pod配置中设置activeDeadlineSeconds属性，可以限制pod的时间。如果pod运行时间超过此时间，系统将尝试终止pod，并将job标记为失败。

> 注意：通过制定Job manifest  中的spec.backoffLimit字段，可以配置Job在被标记为失败之前可以重试的次数。如果你没有明确指定它， 则默认为6。  

## 4.6安排Job定期运行或在将来运行一次
Job 资源在创建时会立即运行pod。但是许多批处理任务需要在特定的时间运行，或者在指定的时间间隔内重复运行。在 Linux 和类UNIX 操作系统中， 这些任务通常被称为cron 任务。 Kubernetes 也支持这种任务。

kubernetes 中的cron任务通过创建CronJob资源进行配置。运行任务的时间表以知名的cron格式指定。

在配置的时间， Kubernetes将根据在 CronJob对象中配置的Job 模板创建 Job 资
源。 创建 Job 资源时， 将根据任务的pod模板创建并启动一个或多个pod副本。

### 4.6.1创建一个CronJob
例如：创建一个每15分钟运行一次前一个示例中的批处理任务。为此，请使用以下规范创建一个CronJob资源:  
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata: 
  name: job-every-fifteen-minutes
spec:
  schedlue: "0, 15, 30, 45 * * * *" 
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            owner: Cron-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cron-task
            image: 192.168.2.7:5000/cron-task
```
**配置时间表安排**
如果你不熟悉 cron 时间表格式，你会在网上找到很棒的教程和解释，但作为－
个快速介绍， 时间表从左到右包含以下五个条目：
- min  分钟
- h    小时
- d    每月中的第几天
- m    月
- w    星期几

可以参考linux的cron格式：分钟(0-59) 小时(0-24) 日(1-31) 月(1-12) 星期(0-6) 要执行的命令（从左到右）每种单位之间有空格

每种单位之间有空格，同种单位之间逗号分开，例如上例：这意味着每小时的0、 15 、 30和45 分钟（第一个星号），每月的每一天（第二个星号）每月（第三个星号），每周的每一天（第四个星号）。由以上解释可以自己推几个，例如：

希望每隔30分钟运行一次，但仅在每月的第一天运行：`0, 30 * 1 * *`
希望每个星期天的上午三点（3AM）运行：`0 3 * * * 0`


**配置Job模板**
略。相关的见创建Job那节


### 4.6.2 了解计划任务的运行方式
在计划的时间内，CronJob资源会创建Job资源，然后Job创建pod。

可能发生Job或pod 创建并运行 得相对较晚的情况。你可能对 这项工作 有很
高的要求，任务开始不能落后于预定的时间过 多。 在这种情况下，可以通过指定 CronJob规范中的startingDeadlineSeconds字段来指定截止日期，例如：
```yaml
apiVersion: batch/v1beta1
kind: CronJob
spec:
  schedule: "0, 15, 30, 45 * * * *"
  startingDeadlineSeconds: 15
  ...
```

如果在15秒内不能启动的话，任务将不会运行，并将显示为Failed。

