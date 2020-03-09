本章内容涵盖：  
- 基于CPU使用率配置pod的自动横向伸缩
- 基于自定义度量配置pod的自动横向伸缩
- 了解为何pod纵向伸缩暂时无法实现
- 了解集群节点的自动横向伸缩

---
## 15.1 pod的横向自动伸缩
横向pod自动伸缩是指由控制器管理的pod副本数量的自动伸缩。它由Horizontal控制器执行，我们通过创建一个HorizontalpodAutoscaler（HPA）资源来启用和配置Horizontal，该控制器会周期性检查pod度量，计算满足HPA资源所配置的目标数值所需的副本数量，进而调整目标资源（如Deployment、
ReplicaSet、 ReplicationController、 StatefulSet等）的replicas字段。

---
### 15.1.1 了解自动伸缩过程
自动伸缩的过程可以分为三个步骤：  
- 获取被伸缩资源对象所管理的所有pod度量。
- 计算使度量数值到达（或接近）所指定目标数值所需的pod数量。
- 更新被伸缩资源的replicas字段。

---
### 获取pod度量
AutoScaler本身并不负责采集pod度量数据，而是从另外的来源获取。pod与节点度量数据是由运行在每个节点的kubelet之上， 名为cAdvisor的agent采集的；这些数据将由 集群级的组件Heapster聚合。 HPA控制器向Heapster 发起REST调用来获取所有pod度量数据。

---
### 计算所需的pod数量
一旦Autoscaler获得了它所调整的资源（delpoy、 RS、RC、sts）算所辖pod的全部度量，它便可以利用这些度量计算出所需的副本数量。它需要计算出一个合适的副本数量， 以使所有副本上度量的平均值尽量接近配置的目标值。


该计算的输入是一组pod度量每个pod可能有多个）， 输出则是一个整数(pod副本数量）。

当Autoscaler配置为只考虑单个度量时（例如只计算Cpu或只计算内存），计算所需 副本数很简单。 只要将所有pod的度量求和后除以HPA资源上配置的目标值，再向上取整即可。

基于多个pod度量的自动伸缩（例如： CPU使用率和每秒查询率[QPS])的计算也并不复杂。AutoScaler单独计算每个度量的副本数， 然后取最大值。（例如：如果需要4个pod达到目标CPU使用率， 以及需要3个pod来达到目标QPS, 那么Autoscaler 将扩展到4个pod)。

---
### 更新被伸缩资源的副本数
自动伸缩操作的最后一 步是更新被伸缩资源对象（比如ReplicaSet)上的副本数字段 然后让ReplicaSet控制器负责启动更多pod或者删除多余的pod。

AutoScaler控制器通过控制Scale子资源来修改被伸缩资源的replicas字段。

这意味着只要API服务器为某个可伸缩资源暴露了Scale子资源， Autoscaler即
可操作该资源。 目前暴露了Scale子资源的资源有：  
- Deployment
- ReplicaSet
- ReplicationController
- StatefulSet

---
### 了解生整个伸缩过程
从pod指向cAdvisor，再经过Heapster，而最终到达HPA的箭头代表度量数据的流向值得

注意的是， 每个组件从其他组件拉取数据的动作是周期性的

---
### 15.1.2 基于CPU使用率进行自动伸缩
对于AutoScaler而言，只有pod的保证CPU用量（CPU 请求）才与确认pod的CPU使用有关。AutoScaler对比pod的实际CPU使用与它的请求用量，这意味着你需要给被伸缩的pod设置CPU请求，不管是直接设置还是通过LimitRange对象间接设置，这样Autoscaler才能确定CPU使用率。

---
### 基于CPU使用率创建HPA
下面看看如何创建 一个HPA, 并让它基于CPU使用率来伸缩 pod。

首先，创建一个Deploy，并且确保创建的所有pod都指定了CPU资源请求，这样才有可能实现自动伸缩。所以需要给deploy的pod模板添加一个CPU资源请求。如下：  

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tt
spec:
  replicas: 2
  template:
    metadata:
      Labels:
        owner: tt
    spec:
      containers:
      - name: ct
        image: go_serv:2.0
        stdin: true
        tty: true
        resources:
          request:
            cpu: 100m
```
这是一个名为tt的deploy对象，，它会运行2个pod实例，每个实例请求100mcpu。

创建了deploy对象后，为了给它的pod启用横向自动伸缩，需要创建一个HorizontalPodAutoScaler（HPA）对象，并把它指向Deployment。可以给HPA准备YAML manifest，但有个办法更简单——还可以用kubectl autoscale命令：

`kl autoscale deployment tt --cpu-percent=30 --min=1 --max=5`

这会帮你创建HPA对象，并将名为tt的deploly设置为伸缩目标（被伸缩资源对象）。你还设置了pod的目标CPU使用率为30%，指定了副本的最小和最大数量。

Autoscaler会持续调整副本的数量以使CPU使用率接近30%, 但它永远不会调整到少于1个或者多于5个。

看看HPA资源的定义：  
```yaml
apiVersion: autoscaling/v2beta
kind: HorizontalPodAutoscaler
metadata: 
  name: hpa-tt
spec:
  maxReplicas: 5
  minReplicas: 1
  metrics: 
  - resources:
      name: cpu
      targetAverageUtilization: 30 #想让AutoScaler调整pod数量以使每个pod都使用所请求CPU的30%
    type: Resource
  scaleTargetRef:   #hpa的伸缩目标（被伸缩资源对象）
    apiVersion: apps/v1
    kind: Deployment
    name: tt
```
上面的是新版


下面的是旧版
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata: 
  name: hpa-tt
spec:
  maxReplicas: 5
  minReplicas: 1
  targetCPUUtilizationPercentage: 30
  scaleTargetRef:   #hpa的伸缩目标（被伸缩资源对象）
    apiVersion: apps/v1
    kind: Deployment
    name: tt
```

可以通过改变pod的CPU的使用率，就可以观察到hpa伸缩deploy。

---
### 了解伸缩操作的最大速率

Autoscaler 在单次扩容操作中可增加的副本数受到限制。如果当前副本数大于 2,Autoscaler 单次操作至多使副本数翻倍。如果副本数只有 l 或 2, Autoscaler 最多扩容到 4 个副本。

另外， Autoscaler 两次扩容操作之间的时间间隔也有限制。只有当 3 分钟内没有任何伸缩操作时才会触发扩容，缩容操作频率更低一－5 分钟。 记住这一点。

---
### 修改一个已有 HPA 对象的目标度量值

可以通过 `kl edit `命令设置targetcpuutilizationpercentage字段的数值。


---
### 基于内存使用进行自动伸缩
基于内存的自动伸缩比基于 CPU 的困难很多。 主要原因在于，扩容之后原有的pod 需要有办法释放内存。 这只能由应用完成，系统无法代芳。 系统所能做的只有杀死并重启应用，希望它能比之前少占用一些内存；但如果应用使用了跟之前一样多的内存， Autoscaler 就会扩容、扩容， 再扩容， 直到达到 HPA 资源上配置的最大pod 数量。 显然没有人想要这种行为。

基于内存使用的自动伸缩在 Kubernetes 1.8 中得到支持，配置方法与基于 CPU 的自动伸缩完全相同。

---
### 15.1.4 基于其他自定义度量进行自动伸缩
需要apiVersion: autoscaling/v2
主要spec。metrics字段中，进行自定义。就不多赘述了，可以参考相关文档。

---
### 缩容到 0 个副本
HPA目前不允许设置 minReplicas 宇段为 0 ，所以 autoscaler 永远不会缩容
到 0 个副本，即便 pod 什么都没做也不会。允许 pod 数量缩容到 0 可以大｜｜幅提升硬件利用率 ：如果你运行的服务几个小时甚至几天才会收到一次请求，就没有道理留着它们一直运行，占用本来可以给其他服务利用的资源：然而一旦客户端请求进来了，你仍然还想让这些服务马上可用

这叫 空载（idling ）与解除空载 （un-idling ），即允许提供特定服务的 pod 被缩容量到0副本。在新的请求到来时，请求会先被阻塞，直到 pod 被启动，从而请求被转发到新的 pod 为止。

Kubernetes 目前暂时没有提供这个特性，但在未来会实现。

---
### 15.2 pod的纵向自动伸缩
目前，一份新的纵向pod自动伸缩提案正在定稿中。


---
### 集群节点的横向伸缩
HA在需要的时候会创建更多的pod实例。 但万一所有的节点都满了， 放不下更多pod了， 怎么办？显然这个问题并不局限于Autoscaler创建新pod实例的场景。即便是手动创建pod, 也可能碰到因为资源被已有pod使用殆尽， 以至于没有节点能接收新pod的清况。

在这种情况下， 你需要删除一些已有的pod, 或者纵向缩容它们， 抑或向集群中添加更多节点。 如果你 的Kubernetes集群运行在自建Conpremise)基础架构上，你得添加一台物理机， 并将其加入集群。 但如果你的集群运行于云端基础架构之上，添加新的节点通常就是点击几下鼠标， 或者向云端做API调用。这可以自动化的，对吧？

---
### 15.3.1 Cluster AutoScaler 介绍
Cluster  Autoscaler负责在由于节点资源不足， 而无法调度某pod到已有节点时，自动部署新节点。它也会在节点长时间使用率低下的情况下下线节点。

由于集群自动伸缩得在云服务提供商的集群上才可用，故不做介绍。


`kl run` 命令与`kl exec `命令类似，都可以使用-it选项 在--后跟上需要在容器中运行的命令。

`kl run -it --rm test --image=go_serv:2.0 --restart=Never -- ash`
-it 允许你进入容器， --rm选项使得pod在退出后自动被删除； --restart=Never表示创建一个非托管的pod，而不是通过deploy对象简介创建。

相当于运行一个一次性的pod，们不仅与本地运行的效果相同，甚至在运行结束之后还会把现场清理干净！


