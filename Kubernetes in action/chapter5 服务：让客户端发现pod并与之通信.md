 **本章内容涵盖：**  
- 创建服务资源，利用单个地址访问一组pod
- 发现集群中的服务
- 将服务公开给外部客户端
- 从集群内部连接外部服务
- 控制pod是与服务关联
- 排除服务故障

现在已经学习过了 pod, 以及如何通过ReplicaSet 和类似资源部署运行。 尽管特定的pod可以独立地应对外部刺激， 现在大多数应用都需要根据外部请求做出响应。 例如， 就微服务而言， pod通常需要对来自集群内部其他 pod, 以及来自集群外部的客户端的HTTP请求做出响应。

pod需要 种寻找其他 pod的方法来使用其他 pod 提供的服务， 不像在没有Kubernetes的世界，系统管理员要在用户端配置文件中明确指出服务的精确的IP 地址或者主机名来配置每个客户端应用， 但是同样的方式在 Kubernetes 中并不适用，因为：  
- pod是短暂的-它们随时会启动或者关闭， 无论是为了给其他 pod 提供空间而从节点中被移除， 或者是减少了 pod的数量， 又或者是因为集群中存在，节点异常。
- Kubernetes在pod启动前会给已经调度到节点上的pod分配IP地址——因此客户端不能提前知道提供服务的pod的IP 地址。
- 水平仲缩意味着多个pod 可能会提供相同的服务——每个 pod都有自己的 IP地址， 客户端无须关心后端提供服务pod 的数量， 以及各自对应的 IP 地址。它们无须记录每个 pod的 IP 地址。 相反， 所有的 pod可以通过一个单一的IP地址进行访问。 

为了解决上述问题， Kubemetes 提供了一种资源类型一服务 (service)

## 5.1 介绍服务



### 5.1.1 创建服务

**通过kubectl expose 创建服务**  
利用expose 命令和pod选择器来创建服务资源，从而通过单个的IP和端口来访问所有的pod。

现在除了使用expose命令，可以通过将配置的YAML文件传递到Kubernetes API服务器来手动创建服务。  

**通过YAML 描述文件来创建服务**  
service也是k8s的资源，缩写为svc。
创建zch-svc-文件：  
```YAML
apiVersion: v1
kind: Service
metadata: 
  name: zch-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    type: serv
```

创建一个名为zch-svc的服务，它将在80端口接收请求并将连接路由（转发到）到标签type=serv的pod的8080端口上。  

使用`kubectl create`创建服务。  

**检测新的服务**  
`kl get svc`  
列表会显示出新创建的svc。因为只是集群的IP地址，只能在集群内部访问。服务的主要目标就是使集群内部的其他pod可以访问当期这组pod。


**在集群内部测试服务**  
在已有pod中运行命令。

**在运行的容器中远程执行命令**  

命令为：`kubectl exec pod_name -- command`，为什么是双杠，双杠之后的内容是pod内部需要执行的命令。以防止kubectl exec 解析带单杠的选项，如果需要在pod内部执行的命令，没有带单杠的选项，当然也就不需要加双杠了。

例如`kubectl exec zch-test -- ./httpclnt 10.104.244.223:9189`
在一个其他pod中向一个后端有三个pod服务的IP发送了HTTP请求，Kubernetes服务代理截取该连接，在三个pod中任意选择了一个pod，然后将请求转发给它。然后该pod就可以get到http服务器的内容。 

**配置服务上的会话亲和性。**  

如果多次执行同样的命令， 每次调用执行应该在不同的pod上。 因为服务代理
通常将每个连接随机指向选中的后端 pod中的一个，即使连接来自于同一个客户端。
另一方面，如果希望特定客户端产生的所有请求每次都指向同一个 pod,可以
设置服务的 sessionAffinity 属性为 ClientIP（而不是None，None是默认值）。  

例如：  
```yaml
apiVersion: v1
kind: Service
spec: 
  sessionAffinity: ClientIP
  ...
```
这种方式将会使服务代理将来自同一个 client IP的所有请求转发至同一个 pod
上。  

Kubernetes 仅仅支持两种形式的会话亲和性服务： None 和 ClientIP。你或
许惊讶竞然不支持基于 cookie的会话亲和性的选项，但是你要了解 Kubernetes 服务
不是在HTTP 层面上工作。因为cookie 是 HTTP 协议中的一部分，服务并不知道它们，这就解释了为什么会话亲和性不能基于 cookie 。  

**同一个服务暴露多个端口**  
创建一个服务可以暴露一个端口，也可以暴露多个端口。比如，你的 pod监听两
个端口，比如 HTTP 监听8080端口、 HTTPS 监听8443 端口，可以使用一个服务从
端口 80和 443 转发至 pod端口 8080和 8443。在这种情况下，无须创建两个不同的
服务。通过一个集群 IP, 使用一个服务就可以将多个端口全部暴露出来。  

> 注意在创建一个有多个瑞口的服务的时候，必须给每个瑞口指定名字。

在服务定义中指定多端口：  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: zch-svc2
spec:
  ports:
  - name: http
    port: 9190
    targetPort: 9190
  - name: https
    port: 9191
    targetPort: 9191
  selector:
    type: serv
```

> **标签选择器应用于**整个服务，不能对每个端口做单独的配置。如果不同的pod有不用的端口映射关系，需要创建两个服务。  

**使用命名端口**  

在pod 的定义中指定 pod名称：  
```yaml
kind: Pod
spec: 
  containers:
  - name: zch-serv
    ports: 
    - name: http
      containerPort: 9190
    - name: https
      containerPort: 9191
```

在在服务中引用命名pod端口：在服务中直接将targetPort用pod中的port那么来替换：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zch-svc2
spec:
  ports:
  - name: http
    port: 9190
    targetPort: http
  - name: https
    port: 9191
    targetPort: https
  selector:
    type: serv
```


为什么要采用命名端口？最大的好处就是即使更换端口号也无须更改服
务 spec。可以减少维护的工作量，不论你的pod端口号如何变（但是名字不能变），都不需要改变服务中的targetPort。  

---

## 5.1.2 服务发现

通过创建服务，现在就可以通过一个单一稳定的 IP 地址访问到 pod 。在服务整
个生命周期内这个地址保持不变。 在服务后面的 pod 可能删除重建， 它们的 IP 地址
可能改变， 数量也会增减，但是始终可以通过服务的单一不变的 IP 地址访问到这些
pod 。

但客户端 pod 如何知道服务的 IP 和端口？是否需要先创建服务， 然后于动查找
其 IP 地址并将 IP 传递给客户端 pod 的配置选项？当然不是。 Kubernetes 还为客户端提供了发现服务的 IP 和端口的方式。  

**通过环境变量发现服务**  
在 pod 开始运行的时候， Kubernetes 会初始化一系列的环境变量指向现在存在
的服务。 如果你创建的服务**早于客户端 pod 的创建**，pod 上的进程可以根据环境变量获得服务的 IP 地址和端口号。  

然后先删除所有的pod，让rs重新创建三个，这样服务就早于pod创建了。然后进入一个pod，来查看环境变量。  
`kl exec pod_name -- env`
然后就可以看到代表了我们创建的服务的IP地址和端口号的环境变量。SVC_NAME_HOST和  SVC_NAME_PORT。

> 注意：服务名称中的横杠被转换为下画线，并且当服务名称用作环境变量名称中
的前级时，所有的字母都是大写的。

环境变量是获得服务 IP 地址和端口号的一种方式，为什么不用 DNS 域名？为
什么 Kubemetes 中没有 DNS 服务器，并且允许通过 DNS 来获得所有服务的 IP 地址？事实证明，它的确如此 ！  

**通过DNS发现服务**  
还记得kube - system 命名空间下列出的所有 pod 的名称吗？其中
一个 pod 被称作 kube-dns ，当前的 kube - system 的命名空间中也包含了一个具
有相同名字的响应服务。

就像名字的暗示， 这个 pod 运行 DNS 服务，在集群中的其他 pod 都被配置成
使用其作为 dns（Kubernetes通过修改每个容器的/etc/resolve.conf文件实现）。运行在pod上的进程DNS查询都会被Kubernetes自身的DNS服务器响应，该服务器知道系统中运行的所有服务。  

> **注意**：pod 是否使用内部的 DNS 服务器是根据 pod 中 spec 的 dnsPolicy 属性来决定的。

每个服务从内部 DNS 服务器中获得一个 DNS 条目，客户端的 pod 在知道服务名称的情况下可以通过全限定域名 CFQDN ）来访问，而不是诉诸于环境变量。  

**通过 FQDN 连接服务**  
`backend-database.default.svc.cluster.local`backend-database 对应于服务名称， default 表示服务在其中定义的命名空间，而 svc.cluster.local 是在所有集群本地服务名称中使用的可配置集群域后缀。

> 注意：客户端仍然必须知道服务的端口号。如果服务使用标准端口号（例如，
HTTP的80端口或Postgres 的 5432端口 ），这样是没问题的。如果并不是标准端口，
客户端可以从环境变量中获取端口号。  

连接一个服务可能比这更简单。如果前端 pod 和数据库 pod 在同一个命名
空间下，可以省略 svc.cluster . local 后缀，甚至命名空间。 因此可以使用
backend- database 来指代服务。 这简单到不可思议，不是吗？

**在 pod 容器中运行 shell**
可以通过 kubectl exec 命令在一个 pod 容器上运行 bash （或者其他形式
的 shell ）通过这种方式，可以随意浏览容器，而无须为每个要运行的命令执行kubectl  exec 。  

例如：  
`kubeclt exec -it zch-test -- /bin/ash`，然后就会进入容器内部了。

可以使用服务名称代替IP来访问服务器了，例如:`curl zch-svc:9189`或`./httpclnt zch-svc:9189`


**无法ping通服务IP的原因**  

curl 这个服务是工作的，但是却 ping 不通。这是因为服务的集群 IP 是一个虚拟 IP ，并且只有在与服务端口结合时才有意义。

## 5.2 连接集群外部的服务
我们 己经讨论了后端是集群中运行的一个或多个 pod 的服务。 但也存在希望通过 Kubernetes 服务特性暴露外部服务的情况。 不要让服务将连接重定向到集群中的 pod，而是让它重定向到外部 IP 和端口 。  

这样做可以让你充分利用服务负载平衡和服务发现。在集群中运行的客户端pod 可以像连接到内部服务一样连接到外部服务。

---

### 5.2.1 介绍服务endpoint
服务并不是和 pod直接相连的。
相反，有一种资源介于两者之间——它就是 Endpoint 资源。可以使用`kubectl describe svc svc_name`命令查看到endpoint。  

Endpoint资源就是一个暴露一个服务的IP地址和端口的列表，Endpoint资源和其他Kubernetes 资源一样，所以可以使用 kubectl get 来获取它的基本信息。缩写为ep。

例如：获取zch-svc服务的Endpoints：  
`kl get endpoints zch-svc`或`kl get ep zch-svc`


尽管在 spec服务中定义了 pod选择器但在**重定向传入连接**时不会直接使用它。相反，选择器用于构建 IP和端口列表，然后存储在Endpoint 资源中。当客户端连
接到服务时，服务代理选择这些 IP和端口对中的一个，并将传入连接重定向到在该位置监听的服务器。  

## 5.2.2 手动配置服务的endpoint
或许已经意识到这一点，服务的 endpoint 与服务解耦后，可以分别手动配置和更新它们。  

> 再强调一边，服务的spec中的选择器是用来构建endpoint的。

如果创建了不包含 pod选择器的服务， Kubernetes将不会创Endpoint资源（毕竟，缺少选择器，将不会知道服务中包含哪些 pod)。这样就需要创建 Endpoint 资源来指定该服务的 endpoint列表。  

要使用手动配置endpoint的方式创建服务，需要创建服务和Endpoint资源。  

**创建没有选择器的服务**  
创建一个不包含选择器的服务：  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: zch-svc2
  annotations: 
    spec: Service without selector
spec:
  ports:
  - port: 9000
```

定义一个名为zch-svc2的服务，它将接受9000端口上的传入连接。并没有为服务定义一个pod选择器。  

**为没有选择器的服务创建Endpoint资源**  

Endpoint是一个单独的资源并不 是服务的一个属性。由于创建的资源中并不包选择器，相关的Endpoints资源并没有自动创建，所以必须手动创建。  

需要额外手动创建Endpoint资源：svc-ep.yaml
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: zch-svc2
subsets:
  - addresses: 
    - ip: 10.244.1.45   #连接重定向到的pod的IP地址
    - ip: 10.244.2.40
    ports:
    - port: 9000    #连接重定向到的pod的IP端口
```
> Endpoint对象需要与服务具有相同的名称，并包含该服务的目标IP地址和端口列表。服务和Endpoint资源都发布到服务器后，这样服务就可以像具有pod选择器那样的服务正常使用。  
在服务创建后创建的容器将包含服务的环境变量，并且与其IP和Port对应的所有连接都将在服务端点之间进行负载均衡。  


如果稍后决定将外部服务迁移到Kubernetes中运行的pod，可以为服务添加选择器，从而对Endpoint进行自动管理。

反过来也是一样。将选择器从服务中移除，Kubernetes将停止更新Endpoints。这意味着服务的IP地址（即提供服务的pod的IP地址）可以保持不变，同时服务的实际实现却发生了改变。  

---

### 5.2.3 为外部服务创建别名

除了手动配置服务的Endpoint来代替公开外部服务方法，有一种更简单的方法，就是通过其完全限定域名(FQDN)访问外部服务。  

**创建ExternalName**  
要创建一个具有别名的外部服务的服务时，要将创建服务资源的一个type字段设置为ExternalName。

ExternalName类型的服务：  
```yaml
apiVersion: v1
kind: service
metadata:
  name: ext-svc
spec:
  type: ExternalName
  externalName: zch.http.com  #实际服务的完全限定域名
  ports:
  - port: 9100
```

服务创建完成后，pod可以通过`ext-svc.default.svc.cluster.local`域名（甚至是`ext-svc`）连接到外部服务。而不是使用服务的实际FQDN。  

这隐藏了实际的服务名称及其使用该服务的pod 的位置，允许修改服务定义，并且在以后如果将其指向不同的服务，只需简单地修改externalName属性，或者将类型重新变回Cluster IP并为服务创建Endpoint­ 无论是手动创建，还是对服务上指定标签选择器使其自动创建。

ExternalName服务仅在DNS级别实施——为服务创建了简单的CNAME DNS记录。因此，连接到服务的客户端将直接连接到外部服务，完全绕过服务代理。 出于这个原因， 这些类型的服务甚至不会获得集群IP。  

>注意CNAME记录指向完全限定的域名而不是数字 IP地址。

---

## 5.3 将服务暴露给外部客户端

到目前为止， 只讨论了集群内服务如何被pod使用；但是， 还需要向外部公开某些服务。 

有几种方式可以在外部访问服务：
- 将服务的类型设置我NodePort——每个集群节点都会在节点上打开一
个端口， 对于NodePort服务， 每个集群节点在节点本身（因此得名叫
NodePort)上打开一个端口，并将在该端口上接收到的流量重定向到基础服务。该服务仅在内部集群IP 和端口上才可访问， 但也可通过所有节点上的专用端口访问。

- 将服务的类型设置成LoadBalance, NodePort类型的一种扩展——这使得服务可以通过一个专用的负载均衡器来访问，这是由Kubernetes中正在运行
的云基础设施提供的。 负载均衡器将流量重定向到跨所有节点的节点端口。客户端通过负载均衡器的 IP连接到服务。

- 创建一个Ingress资源， 这是一个完全不同的机制， 通过一个IP地址公开多个服务——它运行在HTTP层上，因此可以比服务提供更多的功能。  

---

### 5.3.1 使用NodePort类型的服务

通过创建NodePort服务， 可以让Kubernetes在其所有节点上保留一
个端口（所有节点上都使用相同的端口号）， 并将传入的连接转发给作为服务部分的pod。  

这与常规服务类似（它们的实际类型是ClusterIP), 但是不仅可以通过服务的内部集群IP访问NodePort服务， 还可以通过任何节点的IP和预留节点端口访问NodePort服务。  

### 创建NodePort类型的服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zch-nodeport
spec:
  type: NodePort
  ports:
  - port: 9200 #服务的端口号
    targetPort: 9200 #里面提供服务的pod的端口号
    nodePort: 30000    #通过集群节点的30000端口可以访问该服务
  selector:
    type: serv
```
将类型设置为NodePort并指定该服务应该绑定到的所有集群节点的节点端
口。 （指定端口不是强制性的。 如果忽略它，Kubemetes将选择一个随机端口。）

>注意：当在GKE中创建服务时，kubect1打印出一个关于必须配置防火墙规则的警告。

### 查看 NodePort类型的服务
`kl get svc zch-nodeport`
看看EXTERNAL-IP列。 它显示nodes 表明服务可通过任何集群节点的IP地址访问。PORT（S）列显示了服务集群的内部端口和节点端口（30000），可以通过以下地址访问该服务

> 注意:经过实践 NodePort字段的端口号必须在30000~32767。

然后就可以根据节点IP地址和NodePort来访问提供服务的pod了。

---

### 5.3.2 通过负载均衡器将服务暴露出来


### 创建LoadBalance服务
在云提供商上运行的Kubernetes集群通常支持从云基础架构自动提供负载平衡器。 所有需要做的就是设置服务的类型为Load Badancer而不是NodePort。

和NodePort类似：  
```yaml
apiVersion: v1
kind: Service
metadata: 
  name: zch-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 9300
    targetPort: 9190
    NodePort: 30001
  selector:
    type: serv
```

服务类型设置为LoadBalancer而不是NodePort。如果没有指定特定的节点端口，Kubernetes将会选择一个端口。  

### 通过负载均衡器连接服务
创建服务后，云基础架构需要一段时间才能创建负载均衡器并将其IP地址写入服务对象。

创建服务后， 云基础架构需要一段时间才能创建负载均衡器并将其 IP 地址写入服务对象。 一旦这样做了， IP 地址将被列为服务的外部IP 地址：  

然后就可以通过EXTERNAL-IP来访问服务了。  

> 会话亲和性和Web浏览器:  
由于服务现在已暴露在外，可以尝试使用网络浏览器来访问它。但是可能会看到令人奇怪的东西——每次浏览器都会碰到同一个pod。难道是会话亲和性发送变化？，但是使用explain检查服务的亲和性依然设置为None。那么为什么不同的浏览器请求不会碰到不同的pod,  就像使用curl时那样？  
这是因为，览器使用keep-alive连接 ， 并通过单个连接发送所有请求，而curl每次都会打开一个新连接。服务在连接级别工作，所以当
首次打开与服务的连接时， 会选择一个随机集群， 然后将属于该连接的所有 网络数据包全部发送到单个集群。 即使会话亲和性设置为None, 用户也会始终使用相同的 pod (直到连接关闭）。

---

### 5.3.3 了解外部连接的特性

### 了解并防止不必要的网络跳数
当外部客户端通过节点端口连接到服务时随机选择的pod并不一定在接收连接的同一节点上运行。 可能需要额外的网络跳转才能到达pod, 但这种行为并不符合期望。

>例如；node2节点端口接收到了连接，然后服务将连接转发到了node1上的pod，这样就经过了跳转。

可以通过将服务配置为仅将外部通信重定向到接收连接的节点上运行的pod来阻止此额外跳数。这是通过在服务的spec部分中设置`external TrafficPolicy`字段来完成的：  
```yaml
spec:
  externalTrafficPolicy: Local
  ...
```

如果服务定义包含此设置， 并且通过服务的节点端口打开外部连接,则服务代理将选择本地运行的pod，如果没有本地pod存在， 则连接将挂起。因此，需要确保负载平衡器将连接转发给**至少具有一个pod的节点**。

>注意：使用Local外部流量政策的服务可能会导致跨pod的负载分布不均衡,例如：  
一个由两个节点node1、node2，node1上有1个pod，node2上有2个；如果负载平衡器在两个节点间均匀分布连接：就造成了node1上的pod占50%，node2上的2个pod各站25%，这样的并不均衡（最均衡应该各占30%）。

### 记住客户端IP是不记录的
通常，当集群内的客户端连接到服务时， 支持服务的pod可以获取客户端的IP地址。但是， 当通过节点端口接收到连接时，由于对数据包执行了源网络地址转换
(SNAT),  因此数据包的源IP将发生更改。

后端的pod无法看到实际的客户端IP，这对于某些需要了解客户端IP的应用程序来说可能是个问题，例如， 对于Web服务器， 这意味着访问日志无法显示浏览器的IP。

>上一节中描述的local外部流量策略会影响客户端IP的保留， 因为在接收连
接的节点和托管目标pod的节点之间没有额外的跳跃（不执行SNAT)。

---

## 5.4 通过Ingress暴露服务

### 为什么需要Ingress
一个重要的原因是每个 LoadBalancer 服务都需要自己的负载均衡器， 以及独有的公有IP 地址，而 Ingress 只需要一个公网 IP 就能为许多服务提供访问。当客
户端向 Ingress 发送 HTTP请求时，Ingress会根据请求的主机名和路径决定请求转发到的服务。

Ingress在网络栈（HTTP）的应用层操作，并且可以提供一些服务不能实现的功能， 诸如基于 cookie 的会话亲和性 (session affinity) 等功能。

### Ingress控制器是必不可少的
在介绍 Ingress对象提供的功能之前，必须强调**只有Ingress控制器在集群中运行，Ingress资源才能正常工作**。不同的Kubernetes环境使用不同的控制器实现，

---

### 5.4.1 创建Ingress资源
Ingress 是一个相对较新的Kubernetes功能，2020/02/24 先略过。

---

## 5.5 pod就绪后发出信号

之前的存活探针，通过确保异常容器慈东重启来保证应用程序的正常运行。与存活探针类似，Kubernetes还允许为容器定义准备就绪探针。

---

### 5.5.1 介绍就绪探针

### 就绪探针的类型
像存活探针一样，就绪探针有三种类型：  
- Exec 探针，执行进程的地方。容器的状态由进程的退出状态代码确定。
- HTTP GET 探针，向容器发送 HTTP GET 请求，通过响应的 HTTP 状态代码
判断容器是否准备好。
- TCP socket 探针，它打开一个 TCP 连接到容器的指定端口。如果连接己建立，则认为容器己准备就绪。

### 了解就绪探针的操作
与存活探针不同，如果容器未通过准备检查，则不会被终止或重新启动。 这是存活探针与就绪探针之间的重要区别。存活探针通过杀死异常的容器并用新的正常容器替代它们来保持 pod 正常工作，而就绪探针确保只有准备好处理请求的 pod 才可以接收他们（请求）。这在容器启动时最为必要， 当然在容器运行一段时间后也
是有用的。

如果一个容器的就绪探测失败，则将该容器从端点对象中移除（即就绪探针失败的pod从服务的endpoint中移除）。

### 了解就绪探针的重要性
就绪探针确保客户端至于正常的pod交互，并且永远不会知道系统存在问题。

---

### 5.5.2 向pod添加就绪探针
通过修改RC的pod模板来为现有的pod添加就绪探针

### 向pod template 添加就绪探针
可以通过`kubectl edit`命令来向已存在的RC中的pod模板添加探针。

`kubectl edit rc zch-rc`
例如加入exec类型探针：
```yaml
...
spec:
  ...
  template:
  ...'
    spec:
      containers:
      - name: zch
        image: 192.168.2.7:5000/go_serv:1.0
        readinessProbe:
          exec: 
            command:
            - ls
            - /httpserv
```

就绪探针将定期在容器内执行`ls /httpserv` 命令，如果存在httpserv文件，则就绪探针将成功；否则，它会失败。

注意：修改RC的pod模板并不会对现有的pod造成影响。只会影响后来新创建的pod。

---

### 5.5.3 了解就绪探针的实际作用
在实际应用中， 应用程序是否可
以（并且希望）接收客户端请求， 决定了就绪探测应该返回成功或失败。

应该通过删除 pod或更改 pod标签而不是手动更改探针来从服务中手动移除pod。

> 提示 如果想要从某个服务手中手动添加或删除pod，请将enable=true作为标签添加到pod，以及服务的标签选择器中。当想要从服务中移除pod时，删除标签。（这是一个很不错的技巧）。

### 务必定义就绪探针

两个关于就绪探针的要点：  
首先，如果没有将就绪探针添加到pod中，他们会立即成为服务端点，如果应用程序需要很长时间才能开始监听传入连接，则在服务启动但尚未准备好接收传入连接时，客户端请求将被转发到该pod。因此，客户端回看到“连接被拒绝”类型的错误。

> 提示 应该始终定义一个就绪探针，即使他只是向基准URL发送HTTP请求一样简单。

### 不要将停止pod的逻辑纳入就绪探针中

需要提及的另一件事情涉及 pod生命周期结束 (pod关闭）， 并且也与客户端出现连接错误相关。
当一个容器关闭时， 运行在其中的应用程序通常会在收到终止信号后立即停止接收连接。只要删除该容器， Kubernetes就会从所有服务中移除该容器。

---

## 5.6 使用headless服务来发现独立的pod

已经知道使用服务来提供稳定的IP 地址，从而允许客户端连接到支持服务的每个pod (或其他端点）。到服务的每个连接都被转发到一个随机选择的pod上。

但是如果客户端需要链接到所有的pod 呢？

要让客户端连接到所有pod, 需要找出每个pod的IP。一种选择是让客户端调用 Kubernetes API 服务器并通过 API 调用获取pod及其IP 地址列表，但由于应始终努力保持应用程序与 Kubernetes 无关， 因此使用 API服务器并不理想。

幸运的是，Kubernetes 允许客户通过 DNS 查找发现pod IP。 通常， 当执行服务的DNS 查找时， DNS服务器会返回单个IP一服务的集群 IP。但是， 如果告诉
Kubemetes,  不需要为服务提供集群 IP **(通过在服务 spec 中将 clusterIP 字段设置为None 来完成此操作）**， 则 DNS 服务器将返回pod IP而不是单个服务IP。

DNS服务器不会返回单个DNS 记录，而是会为该服务返回多个记录，每
个记录指向当时支持该服务的单个pod的IP。

客户端因此可以做一个简单的DNS记录查找并获取属于该服务所有pod的IP。客户端可以使用该信息连接到
其中的一个、 多个或全部。

---

### 5.6.1 创建headless服务
将服务的spec中的**clusterIP**字段设置为**None**会使服务成为**headless**服务，因为Kubernetes不会为其分配集群IP。

---
下面创建一个名为zch-headless的headless的服务。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: zch-headless
spec:
  clusterIP: None
  ports:
  - port: 9400
    targetPort: 9400
  selector:
    type: serv
```
在使用 kubectl create 创建服务之后，可以 通过kubectl get和kubectl describe来查看服务，你会发现它没有集群P并且它的后端 包含与pod选择器匹配的（部分） pod。部分”是因为pod包含就绪探针， 所以只有准备就绪的pod会被列出作 为服务的后端文件来确保至少有两个pod报告已准备就绪。

---

### 5.6.2 通过DNS发现pod
准备好pod后，现在可以尝试执行DNS查找以查看是否获得了实际的pod IP。需要从其中一个pod执行查找。

例如：创建一个pod，运行apline镜像容器，在该容器中使用命令`nslookup zch-headless`来根据DNS记录查看pod的IP。


> 注意 headless服务仍然提供跨 pod 的负载平衡，但是通过DNS轮询机制（DNS返回了Pod的IP，客户端直接就可以连接到该Pod）而不是通过服务代理。
---

### 5.6.3  发现所有的pod一包括未就绪的pod

只有准备就绪的pod 能够作为服务的后喘。 但有时希望即使 pod没有准备就绪。服务发现机制也能够发现所有匹配服务标签选择器的pod。

幸运的是， 不必通过查询KubemetesAPI服务器， 可以使用DNS查找机制来查找那些未准备好pod。要告诉Kubemetes无论pod的准备状态如何，希望将 所有pod 添加到服务中 。 必须将以下注解添加到服务中：  
```yaml
kind: Service
metadata:
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"

```
警告 就像说的那样， 注解名称表明了这是一个apa功能。KubernetesSevice 
API已经支待一个pulishNotReadyAddresses的新服务规范字段，它将替换tolerate-unready-endpoints注解。 在Kubernetes 1.9.0版本中， 这个字段还没有实现（这个注解决定了未准备好的 end points是否在DNS的记录中）。

---
## 5.7 排除故障服务
服务是Kubemetes的一个重要概念， 也是让许多开发人员感到困扰的根源。 许
多开发人员为了弄清楚无法通过服务IP或FQDN 连接到他们的pod的原因花费了
大量时间。 出于这个原因， 了解一下如何排除服务故障是很有必要的：  

如果无法通过服务访问pod，应该根据下面的列表进行排查：  
- 首先，从集群内部连接到服务的CLUSTER-IP，而不是从外部。
- 不要通过ping服务来判断服务是否可访问（请记住，服务的集群IP是虚拟IP，是无法ping通的）
- 如果已经定义了就绪指针，请确保它返回成功；否则该pod不会成为服务的一部分。
- 如果尝试通过FQDN或其中 一部分来访问服务（例如myservice.mynamespace.svc.cluster.local或myservice.mynamespace）,但并不起作用，请查看是否可以使用集群IP而不是FQDN来访问服务。
- 检查是否连接到服务公开的端口，而不是目标端口。
- 尝试直接连接到pod IP以确认pod正在接受正确端口上的连接（即pod提供的服务功能正常）。
- 如果甚至无法通过pod的IP 访问应用， 请确保应用不是仅绑定到本地主机。

