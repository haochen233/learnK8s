### 本章内容涵盖：  
- 为容器申请CPU、内存以及其他计算资源
- 配置CPU、内存的硬限制
- 理解 pod Qos机制
- 为命名空间中每个pod配置默认、最大、最小资源限制
- 为命名空间配置可以用资源总数

之前，我们在创建pod时并不关心它们使用的CPU和内存资源的最大值，不过为一个pod配置资源的预期使用量和最大使用量是pod定义中的重要组成部分。通过设置这两个参数，可以确保pod公平地使用k8s集群资源，同时也影响着整个集群pod的调度方式。

---
## 14.1 为pod中的容器申请资源
我们创建一个pod时。可以指定容器对 CPU 和内存的资源请求量（即 requests），以及资源限制量（limits）。他们并不在pod里定义，而是针对每个容器单独指定。所以pod对资源的请求量和限制量是它所包含的所有容器的请求量和限制量之和。

---
### 14.1.1 创建包含资源requests的pod
例如：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tt
spec:
  containers:
  - name: t1
    image: 192.168.2.7:5000/go_serv:2.0
    stdin: true
    tty: true
    resources:
      requests:
        cpu: 200m  #申请200毫核
        memory: 10Mi #申请10MB内存
```

在pod spec里，我们 同时 为容器申请了10MB的内存，说明我们期望 容器 内的 进程最大消耗10MB的RAM。它们可能实际占用较小，但在正常情况下我们并不希望它们超过这个值。

当pod启动时，可以在容器中运行top命令快速查看进程的CPU使用。

---
### 14.1.2 资源requests如何影响调度
 通过设置资源requests我们指定了pod对资源需求的最小值。调度器在将pod调度到节点的过程中会用到该信息。每个节点可分配给pod的CPU和内存数量都是一定的 。度器在调度时只考虑那些未分配资源量满足pod 需求量 的节点。如果节点的未分配资源量小千pod 需求量，这时节点没有能力提供pod对资源需求的最小量，因此Kubernetes不会将该pod调度到这个节点

---
### 调度器如何判断一个pod是否适合调度到某个节点
调度器在调度时并不关注各类资源在当前时刻的实际使用量，而只关心节点上部署的所有的pod的资源申请量之和。尽管现有pods 的资源实际使用量 可能小于它的 申请量，但如果使用基于 实际资源消耗量的调度算法将打破系统为这些已部署成功的pods提供足够资源的保证（即所有pod的资源请求量会超过节点资源的上限）。

> 所以，调度器只关注资源 requests，并不关注实际使用量

---
### 调度器如何利用 pod requests 为其选择最佳节点
调度器首先会对节点列表进行过滤，根据预先配置的优先级函数对其余节点进行排序 。其中有两个基于资源请求量的优先级排序函数：LeastRequestedPriority和MostRequestedPriority。前者优先将pod调度到请求量少的节点上（即拥有更多的未分配资源的节点），后者相反。同样，它们只考虑资源请求量，而不关注实际使用资源量。

---
调度器只能配置一 种优先级函数。 你可能在想为什么有人会使用MostRequestedPriority函数。 毕竟如果你有一组节点， 通常会使其负载平均分布，但是但是在随时可以增加或删除节点的云基础设施上运行时并非如此。 配置调度器使用MostRequestedPriority函数， 可以在为每个pod提供足量CPU/内存资源的同时， 确保Kubernetes使用尽可能少的节点。通过紧凑的编排pod，一些节点可以保持空闲并可以随时从集群中删除，通常会按照单个节点付费，这样可以节省开销。

---
### 查看节点资源总量
因为调度器需要知道每个节点拥有多少内存和CPU资源。kubelet会向API服务器报告相关数据。使用`kl describe nodes`命令进行查看。


---
### 创建一个不适合任何节点的 pod
假设创建一个请求cpu大于节点的未分配cpu资源量的pod，该pod会创建会会检查与pending状态，因为没有节点满足它的1cpu requests。假如通过释放节点上的资源（拥有更多的cpu资源后），该pod就会恢复正常运行了。

---
调度器处理CPU和内存requests的方式没有什么不同，但与内存requests相反的是，pod的CPU requests在其运行时也扮演着一个角色。接下来了解下。

---
### 14.1.3 CPU requests如何影响CPU时间分配

们还没有定义任何limits 因此每个pod 分别可以消耗多少CPU并没有做任何限制。那么假设每个pod内的进程都尽情消耗CPU时间，每个pod最终能分到多少CPU时间呢？

CPU requests不仅仅在调度时起作用，它还决定着剩余（未使用）的CPU时间如何在pod之间分配。

假设现在节点上运行了2个pod，一个pod请求了100m，而另一个pod请求了1000m。所以未使的CPU将按照1:5（CPU requests）的比值划分给这两pod，如果两个pod都全力使用CPU的话。

另一方面，如果一个容器能够跑满CPU，而另一个容器在该时段处于空闲状态，那么前者可以将使用可以使用整个CPU时间。（当然会减掉第二个容器消耗的少量时间）。毕竟当没有其他入使用时提高整个CPU的利用率也是有意义的，对吧？当然 ，第二个容器需要CPU时间的时候就会获取到，同时第一个容器会被限制回来。

---
### 14.1.4 定义和申请自定义资源

k8s允许用户为节点添加属于自己的自定义资源，同时支持在pod资源requests里申请这种资源。因为目前是一个apha特性，所以不打算描述其细节。

---
### 14.2 限制容器的可用资源

设置pod的容器的资源申请量（requests）保证了每个容器能够获得它所需要资源的最小量。现在我们再看看硬币的另一面——容器可以消耗资源的最大量。

---
### 14.2.1 设置容器可使用资源量的硬限制
CPU 是一种可压缩资源，意味着我们可以在不对容器内运行的进程产生不利影响的同时，对其使用量进行限制（无非处理速度慢了一点）。而内存明显不同一一是一种不可压缩资源。一旦系统为进程分配了一块内存，这块内存在进程主动释放之前将无法被回收。这就是我们为什么需要限制容器的最大内存分配量的根本原因。

如果不对内存进行限制，工作节点上的容器（或者 pod ）可能会吃掉所有可用内存，会对该节点上所有其他 pod 和任何新调度上来的 pod （记住新调度的 pod 是**基于内存的申请量而不是实际使用量的**）造成影响。单个故障 pod 或恶意 pod 几乎可以导致整个节点不可用。

---
### 创建一个带有资源 limits 的pod
k8s允许用户为每个容器指定资源limits

例如：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lim1
spec:
  containers:
  - name: l1
    image: 192.168.2.7:5000/go_serv
    stdin: true
    tty: true
    resources:
      limits:
        cpu: 1          #这个容器最大允许使用 1核CPU
        memory: 20Mi    #这个容器最大允许使用 20MB内存
```

>注意： 因为没有指定资源 requests， 他将被设置为与资源limits相同的值。

---
### 可超卖的 limits
与requests不同的是，资源limits不受节点可分配资源量的约束。所有limits的总和允许超过节点的资源总量。但是如果节点资源使用量超过100%，一些容器将被杀掉。

> 简单地说，节点上所有pod的资源limits 之和可以超过节点拥有的资源总量（100%）

---
### 14.2.2 超过limits
当进程尝试申请分配比限额更多的内存时会被杀掉（我们会说这个容器被 OOMKilled 了， OOM 是 Out Of Memory 的缩写〉。

CrashLoopBackOff 状态表示 Kubelet 还没有放弃，它意味着在每次崩溃之后，Kubelet 就会增加下次重启之前的间隔时间。 第一次崩渍之后， Kubelet 立即重启容器，如果容器再次崩溃， Kubel et 会等待 10 秒钟后再重启。 随着不断崩溃，延迟时间也会按照 20 、 40、 80 、 160 秒以几何倍数增长，最终收敛在 300 秒。一旦间隔时间达到 300 秒， Kubelet 将以 5 分钟为间隔时间对容器进行无限重启，直到容器正常运行或被删除。

OOMKilled状态告诉我们容器因为内存不存而被系统杀掉了，因此，如果你不希望容器被杀掉，重要的一点就是不要将内存 limits 设置得很低，而容器有时即使没有超限也依然会被 OOMKilled。

---
### 容器中的应用如何看待 limits

在容器内用top命令看到的始终是节点的内存， 而不是容器本身的内存。

即使你为容器设置了最大可用内存的限额， top 命令显示的是运行该容器的节点的内存数量，而容器无法感知到此限制。

与内存完全一样，无论有没有配置 CPU limits ， 容器内也会看到节点所有的CPU。

不要依赖应用程序从系统获取的 CPU 数量，你可能需要使用 Downward API 将CPU 限额传递至容器并使用这个值。也可以通过 cgroup 系统直接获取配置的 CPU
限制，请查看下面的文件：  
/sys/fs/cgroup/cpu/cpu.cfs_quota_us
/sys/fs/cgroup/cpu/cpu.cfs_period_us

---
## 14.3 了解pod Qos等级
前面己经提到资源 limits 可以超卖， 换句话说， 一个节点不一定能提供所有pod 所指定的资源 limits 之和那么多的资源量。

假设有两个 pod, podA 使用了节点内存的 90%, podB 突然需要比之前更多的内存，这时节点无法提供足量内存，哪个容器将被杀掉呢？应该是 podB 吗？因为节点无法满足它的内存请求。 或者应该是 podA 吗？这样释放的内存就可以提供给podB了 。

显然，需要分情况讨论。k8s无法自己做出正确决策，因此就需要一种方式，我们通过这种方式可以指定哪种 pod 在该场景中优先级更高。 Kubernetes将
pod 划分为3种 QoS 等级 ：  
- BestEffort （优先级最低）
- Burstable
- Guaranteed（优先级最高）

---
### 14.3.1 定义pod的Qos等级
你也许希望这个等级会通过一个独立的字段分配，但并非如此。 QoS 等级来源
于 pod 所包含的容器的资源 requests 和 limits 的配置。面介绍分配QoS 等级的方法。

---
### 为pod分配BestEffort等级
最低优先级的 QoS 等级是 BestEffort 。会分配给那些没有（为任何容器）设置任何 requests 和 limits 的 pod 。（所以，之前的学习中创建的大部分pod都是这种优先级的）。

在这个等级运行的容器没有任何资源保证。在最坏情况下它们分不到任何 CPU 时间，同时在需要为其他 pod 释放内存时， 这些容器会第一批被杀死。

不过因为`BestEffort`等级的pod没有设置limits，当有充足的可用内存时，这些容器可以使用任意多的内存（不会出被OOMKilled）。

---
### 为pod分配Guaranteed等级

Guaranteed 等级，会分配给那些所有资源 request 和limits 相等的 pod 。对于一个 Guaranteed 级别的 pod ，有以下几个条件：  
- CPU和内存都要设置requests和limits
- 每个容器都要设置
- requests和limits必须相等

如果没有设置容器资源的requests，则默认与limits相同。

所以只设置所有资源（pod内每个容器的每种资源）的限制量就可以使 pod 的 QoS 等级为Guaranteed。

### 为pod 分配 Burstable 等级
Burstable等级介于BestEffort和Guaranteed之间。其他所有的的pod都属于这个等级。包括容器的requests和limits不相同的单容器pod，至少有一个容器只定义了requests但没有定义limits的pod，以及一个容器指定了相等的requests和limits，但是其余的容器不指定requests（没指定requests当然没指定limits）或limits的pod。

---
### 明白容器的QOS等级
对于单容器pod，容器的Qos等级也适用于pod：  

>记住，所有资源（CPU和内存）都没有设置requests和limits的是BestEffort QOS等级

>记住，所有资源（CPU和内存）都设置了requests和limits，并且同一资源的requests和limits相等的处于Guaranteed

其余所有情况都属于Burstable等级

---
### 了解多容器pod的Qos等级
对千多容器pod, 如果所有的容器的QoS等级相同， 那么这个等级就是pod的
QoS等级（即与单容器无差别）。

但是至少有一个容器的Qos等级与其他不同，无论这个容器什么等级，这个多容器pod的Qos等级都是Burstable等级。

就是上面两条规则，决定了多容器pod属于什么等级。

>注意，可以通过 kubectl describe pod 以及通过查看pod的yaml格式的描述的status.qosClass 字段都可以查看pod的Qos等级。

我们已经清楚如何划分QoS的等级，我们接下来需要了解在一个超卖的系统中如何确定那个容器先被杀掉。

---
### 内存不足时哪个进程会被杀死

在一个超卖的系统，QoS等级决定着哪个容器第一个被杀掉， 这样释放出的资
源可以提供给高优先级的pod使用。 BestEffort等级的pod首先被杀掉， 其次是
Burstable pod,  最后是Guaranteed pod 。 Guaranteedpod只有在系统进程
需要内存时才会被杀掉。

---
### 如何处理相同QoS等级的容器
每个运行中的进程都有 一个称为OutOfMemory (OOM)分数的值。系统通过比较所有运行进程的OOM分数来选择要杀掉的进程。 当需要释放内存时， 分数最高的进程将被杀死。

对于两个属于Burstable等级的单容器的pod, 系统会杀掉 内存实际使用量占内存申请量比例更高的pod。

---
## 14.4 为命名空间中的pod设置默认的requests和limits

### 14.4.1 LimitRange 资源简介
用户可以通过创建一个LimitRange资源来避免必须配置每个容器。 LimitRange资源不仅允许用户 （为每个命名空间）指定能给容器配置的每种资源的最小和最大限额， 还支持在没有显式指定资源 requests 时为容器设置默认值，

> 简单地说，LimitRange资源为每个命名空间指定容器配置的每种的资源的大小范围（限制范围），而且还可以为没有显式指定requests资源时，为容器设置一个默认值。

LimitRange资源被LimitRange准入控制插件。 API服务器接收到带有pod描述信息的POST请求时， LimitRange插件对pod.spec 进行校验。 如果校验失败， 将直接拒绝。 因此， LimitRange对象的一个广泛应用场景就是阻止用户创建大于单个节点资源量的pod。 如果没有LimitRange API服务器将欣然接收pod创建请求， 但永远无法调度成功。

LimitRange应用于同一个命名空间中每个独立的pod、 容器，或者其他类型的对象。它并不会限制这个命名空间中所有pod可用资源的总量。总量是通过 ResourceQuota对象指定。

---
### LimitRange 对象的创建
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example
spec:
  limits:           #指定整个pod的资源的limits
  - type: Pod
    min:            #pod 中所有容器的 CPU 和内存的请求量之和的最小值
      cpu: 50m      
      memory: 5Mi
    max:            #pod 中所有容器的 CPU 和内存的请求量之和的最大值
      cpu: 1
      memory: 1Gi
  - type: Container 
    defaultRequest: #容器没有指定 CPU 或内存请求量时设置的默认值
      cpu: 100m
      memory: 10Mi
    default:        #如果没有指定limits时设置的默认值
      cpu: 200m
      memory: 100Mi
    min:            #容器的 CPU 和内存的资源requests和 limits的最小值和最大值
      cpu: 50m
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
    maxLimitRequestRatio: #每种资源的requests和limits的最大比值
      cpu: 4
      memory: 10
  - type: PersistentVolumeClaim #LimitRange还可以指定请求PVC存储容量的最小和最大值
    min:
      storage: 1Gi
    max:
      storage: 10Gi
```

整个pod资源限制的最小值和最大值是可以配置的。它应用于pod内所有容器的requests和limits之和（- type: Pod字段）。

其他级别（如容器的限制）都有注释说明。

>多个LimitRange对象的限制会在校验pod或PVC 合法性时进行合并。由于LimitRange象中配置的校验（和默认值）信息在API服务器接收到新的pod或PVC创建请求时执行，如果之后 修改了限制，已经存在的pod和PVC将不会再次进行校验，新的限制只会应用于之后创建的pod和PVC 。（与更新pod模板类似，只会影响后来创建的pod）。

---
### 14.4.3 强制进行限制
假如创建一个Cpu requests大于LimitRange中设置的最大值，创建时Pod会从server返回错误信息（too-big）。

---
### 14.4.4 应用资源requests和limits的默认值
如果不指定资源的requests和limits，k8s就会使用LimitRange设置的默认值。

> 注意，LimitRange中配置的limits只能应用于单独的pod或容器。用户仍然可以创建大量的pod吃掉集群所有可用资源。LimitRange并不能防止这个 问题，而 相反，我们将在下文了解的ResourceQuota对象可以做到这点。

---
### 14.5 限制命名空间中的可用资源总量
LimitRange只 应用于单独的pod, 而我们同时也需要一种手段可以限制命名空间中的 可用资源总量。这通过创建一个ResourceQuota对象来实现 。

---
### 14.5.1 ResourceQuota资源介绍
ResourceQuota的接纳控制插件会检查将要创建的pod是否会引起总资源量超出ResourceQuota。如果超出，创建请求会被拒绝，因为资源配额在pod创建时检查，所以ResourceQuota对象仅仅作用于其后创建的pod——并不影响已经存在的pod。

资源配额限制了一个命名空间中pod和PVC存储最多可以使用的资源总量。同时也可以限制用户允许在该命名空间中创建pod、PVC, 以及其他API对象的数量。

下面创建一个ResourceQuota对象来限制命名空间中所有pod允许使用的CPU和内存总量可以通过创建ResourceQuota对象来实现。
```yaml
apiVersion: v1
kind: ResourceQuota
metadata: 
  name: cpu-mem
spec:
  hard:
    requests.cpu: 500m
    requests.memory: 100Mi
    limits.cpu: 600m
    limits.memory: 150Mi
```

这个ResourceQuota设置了命名空间下所有pod最多可申请的CPU数量为500毫核，limits最大总量为600毫核。对于内存，设置所有requests的最大总量为100MiB，limits为150MiB。

与LimitRange一样，ResourceQuota对象应用于它所创建的那个命名空间，但不同的是，后者可以限制所有pod资源requests和limits 的总量，而不是每个单独的pod或容器。

> LimitRnage应用于单独的pod，ResourceQuota应用于命名空间中所有pod。

---
### 查看配额和配额使用情况
使用命令: `kl describe quota`

---
### 与ResourceQuota 同时创建 LimitRange
假如我们没有在LimitRange中配置容器资源的默认的limits和requests，而且还不指定limits和requests，那么将不能创建。

因此当热定资源配置了配额。在pod中必须为这些资源指定requests和limits（不管是手动指定还是利用LimitRange中的默认设置），否则API服务器不会接收该pod创建请求。

---
### 14.5.2 为持久化存储指定配额
ResourceQuota 对象同样可以限制某个命名空间中最多可以声明的持久化存储总量。
例如：  
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage
spec:
  hard:
    requests.storage: 5Gi
```

namespace中申请的PVC总量被限制为5GiB。

---
### 14.5.3 限制可创建对象的个数
资源配额同样可以限制单个命名空间中的 pod 、 Repli cationController、 Servi ce以及其他对象的个数。

下面看看这个例子，如何限制资源最大个数：  
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: objs
spec:
  hard:
    pods: 10
    replicationcontrollers: 5
    secrets: 10
    configmaps: 10
    persistentvolumeclaims: 6
    services: 5
    services.nodeports: 2

```

允许用户在一个命名空间中最多创建 10 个 pod ，无论是于动创建还是通过 ReplicationController、 ReplicaSet、 DaemonSet 或者 Job 创建的。 同时限制了ReplicationController 最大个数为 5, Service 最大个数为 5 ，NotPort 类型最多 2 个。

---
对象个数配额数目前可以为以下对象配置：  
- pod
- RC
- Secret
- ConfigMap
- PersistentVolumeClaim
- Service，NodePrt类型，LoadBalancer类型

最后甚至可以为ResourceQuota对象本身设置对象个数配额，其他对象的个数，RS、Job、Deploy、Ingress等暂时不能限制。

---
### 14.5.4 为特定的 pod 状态或者 QoS 等级指定配额
目前为止我们创建的 Quota 应用于所有的 pod ， 不管 pod 的当前状态和 QoS 等
级如何。

但是 Quota可以被一组quota scopes限制。目前配额作用范围共有4种：BestEffort、NotBestEffort、Termination、Noterminating。

BestEffort 和 NotBestEffort 范围决定配额是否应用于 BestEffort，QoS 等级或者其他两种等级（Burstable 和 Guaranteed ） 的 pod。

其他两个范围的名称或许有些误导作用，实际上并不应用于处在（或不处在）停止过程中的 pod 。

但你可以为每个 pod 指定被标记为 Failed ，然后真正停止之前还可以运行多长时间。 这是通过在 pod spec 中配置 activeDeadlineSeconds 来实现的。该属性定义了一个 pod 从开始尝试停止的时间到其被标记为 Failed 然后真正停止之前，允许其在节点上继续运行的秒数。 Terminating 配额作用范围应用于这些配置了 activeDeadlineSeconds 的 pod ， 而 NotTerminating 应用于那些没有指定该配置的 pod 。

> 简单地说，Termination应用于配置了activeDeadlineSeconds字段的pod，而Notterminating应用于没有配置该字段的pod。

例如，配置BestEffort以及Notterminating的pod的quota：  
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: exp
spec:
  scopes:
  - BestEffort
  - NotTerminating
  hard:
    pods: 5       #这样的pod只允许存在5
```

---
## 14.6 监控pod的资源使用量
Kubelet 自身就包含了一个名为 cAdvisor 的 agent ，它会收集整个节点和节点上运行的所有单独容器的资源消耗情况。 集中统计整个集群的监控信息需要运行一个叫作 Heapster 的附加组件。

Heapster以 pod 的方式运行在某个节点上，它通过普通的 Kubernetes Service 暴
露服务，使外部可以通过一个稳定的 IP 地址访问。 它从集群中所有的 cAdvi sor 收
集数据，然后通过一个单独的地址暴露。

> Heapster以pod的形式运行在其中一个节点上并会收集所有节点的指标。

pod感知不到cAdvisor的存在，cAdvisor也感知不到Heapster的存在。Heapster 主请求所有的 cAdvisor ，同时 cAdvisor 无须通过与 pod 容器内进程通信就可以收集到容器和节点的资源使用数据。

---
### 启用 Heapster
