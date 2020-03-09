本章内容涵盖: 
- 使用节点污点和pod容忍度组织pod到特定节点
- 将节点亲缘性规则作为节点选择器的一种替代
- 使用节点亲缘性进行多个pod的共同调度
- 使用节点非亲缘性来分离多个pod
  
k8s允许你去影响pod被调度到哪个节点。起初， 只能通过在pod规范里指定节点选择器来实现， 后面其他的机制逐渐加入来扩容这项功能。

---
## 16.1 使用污点和容忍度阻止节点调度到特定节点
首先要介绍的高级调度的两个特性是节点污点，以及pod对于污点的容忍度。这些特性被用于限制哪些pod可以被调度到某一个节点。 只有当一个pod容忍某个节点的污点， 这个pod才能被调度到该节点。

这与使用节点选择器和节点亲缘性有些许不同。节点选择器和节点亲缘性规则，是通过明确的在pod中添加的信息，来决定一个pod可以或者不可以被调度到哪些节点上。而污点则是在不修改巳有pod信息的前提下，通过在节点上添加污点信息，来拒绝pod在某些节点上的部署。

---
### 16.1.1 介绍污点和容忍度
学习节点污点的最佳路径就是看一个已有的污点。使用kubeadm工具部署的一个多节点集群，默认情况下这样一个集群中的主节点需要设置污点，这样才能保证只有控制面板pod才能部署在主节点上。

---
### 显示节点的污点信息
可以通过`kubectl describe node`查看节点的污点信息。

主节点包含一个污点，污点包含了 一个key、value, 以及一个effect, 表现为 \<key>=\<value>:\<effect>。

这个污点将阻止pod调度到这个节点上面，除非有pod能容忍这个污点，而通常能够容忍这个污点的pod都是系统级别的pod。

> 一个pod只有容忍了节点的污点，才能被调度到该节点上面。
> 没有容忍度的pod只会调度到没有污点的节点上。

---
### 显示pod的污点容忍度
通过`kl describe 命令查看pod的Toleration字段`

通过查看kube-proxy的信息得到下面的Tolerations:
```yaml
node-role.kuberne七es.io/master=:NoSchedule
node.alpha.kubernetes.io/notReady=:Exists:NoExecute
node.alpha.kubernetes.io/unreachable=:Exists:NoExecute
```

> 注意尽管在pod的污点容忍度中显示了等号，但是在节点的污点信息中却没有。
当污点或者污点容忍度中的value为null时，kubectl故意将污点和污点容忍度进
行不同形式的显示。

---
### 了解污点的效果
另外两个在kube-proxy pod 上的污点定义了当节点状态是没有ready 或者是
unreachable 时，该pod 允许运行在该节点多长时间。这两个污点容忍度使用的效果是 NoExecute 而不是 NoSchedule。

每一个污点都可以关联一个效果， 效果包含了以下三种：
- NoSchedule表示如果pod 没有容忍这些污点， pod 则不能被调度到包含这些污点的节点上。
- PreferNoSchedule是 NoSchedule 的一个宽松的版本， 表示尽量阻止pod被调度到这个节点上， 但是如果没有其他节点可以调度， pod 依然会被调度到这个节点上。（尽量不要调度到该节点，除非没有其他可以调度的节点的情况下，才可以调度带给节点）。
- NoExecute不同于 NoSchedule以及 PreferNoSchedule,后两者只在调度期间起作用， 而 NoExecute 也会影响正在节点上运行着的pod。如果在一个节点上添加了 NoExecute污点， 那些在该节点上运行着的pod, 如果没有容忍这个 NoExecute污点， 将会从这个节点去除。

---
### 16.1.2 在节点上添加自定义污点
假设你有一个单独的Kubernetes 集群， 上面同时有生产环境和非生产环境的流量。 其中最重要的一点是， 非生产环境的pod 不能运行在生产环境的节点上。 可以通过在生产环境的节点上添加污点来满足这个要求， 可以使用 kubectl taint 命令来添加污点：

`kubectl taint node node1 node-type=production:NoSchedule`
这个命令增加了一个taint（污点），其中key为node-type，value为production。

如果你部署常规pod的多个副本，你会发现没有一个pod被部署到你添加了污点信息的节点上面。

现在，没人能够随意地将 pod部署到生产环境节点上了。

---
### 16.1.3 在pod上添加污点容忍度
为了将生产环境pod部署到生成环境节点上，pod需要能容忍那些你添加在节点上的污点。你的生产环境pod的清单里面需要增加以下的YAML代码片段。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod
spec:
  containers:
  - name: p1
    image: go_serv:2.0
    stdin: true
    tty: true
  tolerations:
  - key: node-type      #key
    operator: Equal     #操作符（默认的）即指定匹配的value，或者使用Exists操作符来匹配污点的key。
    value: production   #value
    effect: NoSchedule  #效果不能调度
```

---
### 16.1.4 了解污点和污点容忍度的使用场景
污点可以只有一个key和一个效果，而不必设置value。污点容忍度可以通过设置Equal操作符Equal操作符来指定匹配的value (默认情况下的操作符），或者也可以通过设置Exists操作符来匹配污点的key。

---
### 在调度时使用污点和容忍度
污点可以用来组织新pod的调度（即effect: NoSchedule）或者定义非优先调度的节点（使用PreferNoSchedule效果）， 甚至是将已有的pd从当前节点剔除。

---
### 配置节点失效之后的pod重新调度最长等待时间
你也可以配置一个容忍度， 用于当某个pd运行所在的节点变成Notready或者Unreachable状态时，k8s可以等待该pod被调度到其他节点的最长等待时间，

例如：  
```yaml
...
  tolerations:
  - effect: NoExecute
    key: node1.kubernetes.io/notready
    opeartor: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node1.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
```

这两个容忍度表示， 该pod将 容忍所 在节点处于notReady或者unreachable状 态维持300秒。 当Kubernetes控制器检测到有节点处于notReady或者unreachable状态时， 将会等待300秒，如果状态持续的话， 之后将把该pod重新调度到其他节点上。

当没有定义这两个容忍度时， 他们会自动添加到pod上。 如果你觉得对于你的pod来说，5分钟太长的话，可以在pd描述中显式地将这两个容忍度设置得更短一些。

> 注意：当前这是一个alpha阶段的特性，在未来的Kubemees版本中可能会有所改变。基于污点信息的pd剔除也不是默认启用的， 如果要启用这个特性， 需要在运行控制器管理器时使用--feature-gates=TaintBasedEvictions=true 选项。

---
## 16.2 使用节点亲缘性将pod调度到特定节点上
污点可以让pod远离特定的节点。现在，学习一种更新的机制，叫作节点亲缘性（node affinity），这种机制允许你通知k8s将pod只调度到某个节点子集上面。

---
### 对比节点亲缘性和节点选择器
在早期版本的Kubernetes 中， 初始的节点亲缘性机制， 就是 pod 描述中的nodeSelector 字段。 节点必须包含所有 pod对应字段中的指定label, 才能成为pod调度的目标节点。

节点选择器实现简单， 但是它不能满足你的所有需求。 正因为如此，一种更强大的机制被引入。 节点选择器最终会被弃用， 所以现在了解新的节点亲缘性机制就变得重要起来。

与节点选择器类似， 每个pod可以定义自己的节点亲缘性规则。 这些规则可以允许你指定硬性限制或者偏好。 如果指定一种偏好的话， 你将告知 Kubernetes 对于某个特定的pod, 它更倾向于调度到某些节点上， 之后 Kubernetes 将尽量把这个pod调度到这些节点上面。 如果没法实现的话， pod将被调度到其他某个节点上。

---
### 16.2.1 指定强制性节点亲缘性规则
比如之前做过的一个例子：将pod调度到有gpu标签的节点，使用的是nodeSelector。现在将nodeSelector替换为节点亲缘性规则。

```yaml
...
spec:
  affinity：
    nodeAffinoty:
      requiredDuringSchedulingIgnoreDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cpu
            operator: In
            values:
            - "true"

```

首先你会注意到的是。这种写法比简单的节点选择器要复杂得多，但这是因为它的表达能力更强，让我们再详细看一下这个规则。

---
### 较长的节点亲缘性属性名的意义
requiredDuringSchedulingIgnoreDuringExecution，这个字段有极其长的名字，所以，让我们先重点关注这个。拆开来看

- requiredDuringScheduling：表明了该字段下定义的规则，为了让pod 能调度到该节点上，明确指出了该节点必须包含的标签。
- IgnoredDuringExecution：表明了该字段下定义的规则， 不会影响己经在节点上运行着的pod。

目前，当你知道当前的亲缘性规则只会影响正在被调度的pod ， 并且不会导致己经在运行的 pod 被剔除，情况可能会更简单一些。 这就是为什么目前的规则都是以 IgnoredDuringExecution结尾的。

最终，Kubernetes 也会支持RequiredDuringExecution，表示如果去除掉节点上的某个标签，那些需要节点包含该标签的 pod 将会被剔除。（让pod可以运行在节点上）

Kubernetes 目前还不支持次特性。所以，我们可以暂时不去关心这个长字段的第二部分。

> 一个pod的节点的亲缘性指定了节点必须包含哪些标签才能满足pod调度的条件

---
### 16.2.2 调度pod时优先考虑某些节点

最近介绍的节点亲缘性的最大好处就是，当调度某一个 pod 时，指定调度器可以优先考虑哪些节点，这个功能是通过preferredDuringSchedulingIgnoredDuringExecution字段来实现的。

---
### 给节点加上标签
首先，节点必须加上合适的标签。节点的亲缘性规则才能使用。
`kl lable` 命令


---
### 指定优先级节点亲缘性规则
```yaml
...
spec:
  affinity:
    nodeAffinity:
    preferredDuringSchedulingIgnoreDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: func
          operator: In
          values:
          - dev
    - weigut: 20
      preference:
        matchExpressions:
        - key: func
          operator: In
          values:
          - test
```

优先调度到func=dev节点。这是最重要的偏好（权重weight80），同时优先调度到func=test节点上。

第一个优先级规则是相对重要的， 因此将其weight设置为80, 而第二个优先级规则就不那么重要(weight设置为20)。

---
### 了解节点优先级是如何工作的
> 基于pod节点亲缘性优先级对节点进行排序

根据计算出来的权重（weight），进行排序。权重大的优先考虑。

例如，现有四个节点，1号节点包含两个func=dev和func=test标签则weight为100，2号节点包含func=dev标签weight为80，3号节点包含func=test标签weight为20。4号节点不包含任何标签weight为20。那么调度的优先级分别为 1、2、3、4号节点。

---
## 16.3 使用pod亲缘性与非亲缘性对pod进行协同部署
你已经了解了节点亲缘性规则是如何影响pod被调度到哪个节点。但是， 这些规则只影响了pod和节点之间的亲缘性。然而，有些时候你也希望能有能力指定pod自身之间的亲缘性。

举例来说，想象一下你有一个前端pod和一个后端pod, 将这些节点部署得比较靠近，可以降低延时，提高应用的性能。可以使用节点亲缘性规则来确保这两个pod被调度到同一个节点

---
### 16.3.1 用pod间亲缘性将多个pod部署在同一个节点上
例如：将一个前端pod调度到和其他包含app=backend标签的pod所在的相同节点上（通过
topologyKey字段指定）

```yaml
...
metadata:
  name: frontend
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: kubernetes.io/hostname
        labelSelector:
          matchLabels:
            app: backend

```
其中topologyKey的值其实就是一个标签的key。简单滴理解代表一个标签的key，并没有value。

---
### 16.3.3表达pod亲缘性优先级取代强制性要求
nodeAffnity可以表示一种强制性要求，表示pod只能被调度到符合节点亲缘性规则的节点上。它也可以表示一种节点优先级，用于告知调度器将pod调度到某些节点上，同时也满足当这些节点出于各种原因无法满足 pod 要求时，将pod调度到其他节点上。

这种特性同样适用于podAffinity, 你可以告诉调度器，优先将前端pod调度到和后端pod相同的节点上，但是如果不满足需求，调度到其他节点上也是可以的。

一个使用了preferredDuringSchedulingIgnoredDuringExecution的pod
```yaml
...

spec:
  affinity:
    podAffinity
      preferredDuringSchedulingIgnoreDuringExecution:
      - weight: 80
        podAffinityTerm:
          topolgyKey: kubernetes.io/hostname
          labelSelector: 
            matchLabels:
              app: backend
  ...
```

>pod亲缘性可以用来告知调度器优先考虑那些有包含某些标签的pod正在运行着的节点。

---
### 16.3.4 利用pod的非亲缘性分开调度pod

有时候你的需求却恰恰相反， 你可能希望pod远离彼此。 这种特性叫作pod非亲缘性。 它和pod亲缘性的表示方式一样， 只不过是将podAffinty字段换成podAntiAffinty, 这将导致调度器永远不会选择那些有包含podAnantiAffinity匹配标签的pod所在的节点。

例子就不写了。

---
一个为什么需要使用pod非亲缘性的例子， 就是当两个集合的pod, 如果运行在同一个节点上会影响彼此的性能。 在这种情况下， 你需要告知调度器永远不要将这些pod部署在同一个节点上。另一个例子是强制让调度器将同一组的pod分在不同的可用性区域或者地域， 这样让整个区域或地域失效之后， 不会使得整个服务完全不可用。

---
### 理解pod非亲缘性优先级
与使用pod情缘性相反（podAffinity），toplogyKey字段决定了pod不能调度的范围（不能往有该标签的key的节点集合调度），而亲缘性相反（必须在有改标签key的节点集合内调度）。