**本章学习内容涵盖：**  
- 创建、启动和停止pod  
- 使用标签组织pod和其他资源
- 使用特定标签对所有pod执行操作
- 调度pod到指定类型的工作节点

接下来我们将更加详细地介绍所有类型的Kubemetes对象（或资源），以便你理解在何时、如何及为何要使用每一对象（资源）。其中pod是kubernetes中最为重要的核心概念，而其他对象仅仅是在管理、暴露pod

或被pod使用，所以接下来着重介绍pod这一概念。  

## 3.1介绍pos  
pod是一组并置的容器， 代表了Kubemetes中的基本构建模块。更多的是针对一组pod的容器进行部署和操作。 然而这并不意味着一个pod总是要包含多个容器一一实际上只包含一个，单独容器的pod也是非常常见的。当一个pod包含多个容器时， 这些容器总是运行于同一个工作节点上一－个pod绝不会跨越多个工作节点。  
如下图：  
![一个pod中的容器只能运行在同一个节点上](https://haochen233.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E5%BA%8A/%E4%B8%80%E4%B8%AApod%E4%B8%AD%E7%9A%84%E6%89%80%E6%9C%89%E5%AE%B9%E5%99%A8%E9%83%BD%E8%BF%90%E8%A1%8C%E5%9C%A8%E5%90%8C%E4%B8%80%E4%B8%AA%E8%8A%82%E7%82%B9%E4%B8%8A.png?Expires=1581991247&OSSAccessKeyId=TMP.hjZcV6CSzJbSMCFgKtHHLd6QsnwyKanaBBwDasakLa7ABzo7oeP7gBLG7ofAZtVnBeBSzPBb7weUyyouf4MkzoVCVFYeZ7CmsGvnSNrJHQyDT6qZeLud2Whig3EP2a.tmp&Signature=SDNVs78LIRQ0Qwwc4ouCJQf4kfs%3D)  

### 3.1.1 为何需要pod  
关于为何需要pod这种容器？为何不直接使用容器？为何甚至需要同时运行
多个容器？难道不能简单地把所有进程都放在一个单独的容器中吗？  

下面为解答：  
**为何多个容器比单个容器中包含多个进程要好**  
想象一个由多个进程组成的应用程序， 无论是通过ipc (进程间通信）还是本
地存储文件进行通信， 都要求它们运行于同一台机器上。容器被设计为每个容器只运行一个进程（除非进程本身产生子进程）。 如果在
单个容器中运行多个不相关的进程， 那么保持所有进程运行、 管理它们的日志等将会是我们的责任。同时这些进程都将记录到相同的标准输出中，而此时我们将很难确定每个进程分别记录了什么。  
所以，我们需要让每个进程运行于自己的容器中，而这就是Docker和Kubernetes期望使用的方式。  

### 3.1.2 了解pod

由于不能将多个进程聚集在一个单独的容器中，我们需要另一种更高级的结构来将容器绑定在一起，并将它们作为一个**单元**进行管理。在包含容器的pod下，我们可以将同时运行一些密切相关的进程，并为他们提供相同的环境，此时这些进程就好像运行在一个单独的容器中（即每个容器包含一个进程，这些容器又包含在pod中，所以可以将此抽象为，多个进程运行在一个单独的容器中。），同时又能保持一定的隔离（一个容器只含有一个进程）。实现了两全其美。  

**同一pod中容器的部分隔离**  
容器之间彼此是完全隔离的， 但此时我们期望的是隔离容器组（pod）， 而不是单个容器,并让每个容器组内的容器共享一些资源， 而不是
全部（换句话说， 没有完全隔离）。Kubemetes通过配置 Docker 来让一个 pod 内的所有容器共享相同的Linux 命名空间， 而不是每个容器都有自己的一组命名空间。  '

由千一个pod 中的所有容器都在相同的network 和UTS命名空间下运行，所以它们共享相同的主机名和网络接口。同样
地， 这些容器也都在相同的IPC 命名空间下运行， 因此能够通过IPC 进行通信。 在
最新的Kubernetes 和 Docker 版本中， 它们也能够共享相同的 PID 命名空间， 但是
该特征默认是未激活的。  

注意当同一个pod中的容器使用单独的PID命名空间时，在容器中执行ps aux就只会看到容器自己的进程。  

当涉及文件系统时，情况就有所不同。由于大多数容器的文件系统来自容器镜像， 因此默认情况下， 每个容器的文件系统与其他容器完全隔离。但我们可以使用名为 Volume的 Kubernetes 资源来共享文件目录。  

**容器如何共享相同的IP和端口空间**  
由于一个pod中的容器运行于相同的Network命名空间中， 因此它们共享相同的IP地址和端口空间。这意味着在同一 pod中的容器运行的
多个进程需要注意不能绑定到相同的端口号， 否则会导致端口冲突。但这只涉及同一pod中的容器（共享network namespace）。由于每个pod都有独立的端口空间。对于不同pod中的容器来说则永远不会遇到端口冲突问题。此外，
一个 pod中的所有容器也都具有相同的loopback网络接口， 因此容器可以通过localhost 与同一 pod中的其他容器进行通信。  


**介绍平坦pod间网络**  
Kubemetes 集群中的所有 pod都在同一个共享网络地址空间中，如下图：  
![k8s集群中所有pod都在同一个共享网络地址空间中](https://haochen233.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E5%BA%8A/k8s%E9%9B%86%E7%BE%A4%E4%B8%AD%E6%89%80%E6%9C%89pod%E9%83%BD%E5%9C%A8%E9%80%9A%E4%B8%80%E4%B8%AA%E5%85%B1%E4%BA%AB%E7%BD%91%E7%BB%9C%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4%E4%B8%AD.png?Expires=1581993454&OSSAccessKeyId=TMP.hjZcV6CSzJbSMCFgKtHHLd6QsnwyKanaBBwDasakLa7ABzo7oeP7gBLG7ofAZtVnBeBSzPBb7weUyyouf4MkzoVCVFYeZ7CmsGvnSNrJHQyDT6qZeLud2Whig3EP2a.tmp&Signature=PHp%2Bj2naoEf0SYp771kovoFNILE%3D)换句这也
表示它们之间没有 NAT (网络地址转换） 网关。即不需要网络地址转换（这就是平坦的网络），直接要实际的IP地址在pod间通信。  
不论是将两个 pod 安排在单一的还是不同的工作节点上，这些 pod内的容器都能够像在无NAT的平坦网络中一样相互通信， 就像局域网 (LAN) 上的计算机一样。 此时，每个 pod 都有自己的 IP 地址， 并且可以通过这个专门的网络实现 pod之间互相访问。  

总结下：pod
像是一个逻辑主机，其行为与现实中的物理主机或虚拟机非常类似。运行在同一个pod中的进程与运行在同一个物理机上的进程相似，不过每个进程都封装在一个容器中。

### 3.1.3  通过pod合理管理容器  

将 pod 视为独立的机器， 其中每个机器只托管一个特定的应用。 过去我们习惯于将各种应用程序塞进同一台主机， 但是pod不是这么干的。 由于pod 比较轻量，我们可以在几乎不导致任何额外开销的前提下拥有尽可能多的pod。与将所有内容填充到一个 pod 中不同， 我们应该将应用程序组织到多个 pod 中，而每个pod只包含紧密相关的组件或进程。  

**将多层应用分散到多个pod中**  
虽然我们可以在单个pod中同时运行前端服务器和数据库这两个容器， 但这种
方式并不值得推荐。那假如你非要把它们放在一起， 有错吗？某种程度上来说， 是的

如果前端和后端都在同一个容器中， 那么两者将始终在同一台计算机上运行。
如果你有一个双节点Kubemetes 集群， 而只有一个单独的pod, 那么你将始终只会
用一个工作节点， 而不会充分利用第二个节点上的计算资源 (CPU 和内存）。 因此
更合理的做法是将pod拆分到两个工作节点上， 允许 Kubemetes 将前端安排到一个
节点， 将后端安排到另一个节点， 从而提高基础架构的利用率。  

**基于扩缩容考虑而分割到多个pod 中**  
另一个不应该将应用程序都放到单一pod中的原因就是扩缩容。pod也是扩缩容的基本单位，对于Kubernetes来说，他不能横向扩缩单个容器，只能扩缩整个pod。如果你需要单独扩缩容器， 那么这个容器应该很明确地被部署在单独的pod中（即pod中只有单独一个容器）。  

**何时在pod中使用多个容器**  
将多个容器添加到单个pod的主要原因是应用可能由一个主进程和一个或多个辅助进程组成， 如图：  
![pod应该包含紧密的容器组](https://haochen233.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E5%BA%8A/pod%E5%BA%94%E8%AF%A5%E5%8C%85%E5%90%AB%E7%B4%A7%E5%AF%86%E8%80%A6%E5%90%88%E7%9A%84%E5%AE%B9%E5%99%A8%E7%BB%84.png?Expires=1581996375&OSSAccessKeyId=TMP.hjZcV6CSzJbSMCFgKtHHLd6QsnwyKanaBBwDasakLa7ABzo7oeP7gBLG7ofAZtVnBeBSzPBb7weUyyouf4MkzoVCVFYeZ7CmsGvnSNrJHQyDT6qZeLud2Whig3EP2a.tmp&Signature=FXWiPQYnfYoXI01z%2FiUjVQL%2FE1M%3D)
例如：pod中的主容器可以是一个仅仅服务于某个目录中的文件的web服务器，而另一个容器则定期从外部源下载内容并将其存储在Web服务器目录中。  

**决定何时在pod中使用多个容器**  
当决定是将两个容器放入一个 pod 还是
两个单独的 pod 时，我们需要问自己以下问题：
+ 他们需要一起运行还是可以在不同的主机上运行？
+ 它们代表的是一个整体还是相互独立的组件？
+ 它们必须一起进行扩缩容还是可以分别进行？
下图将有助于我们理解这一点：  
![容器不应该包含多个进程， pod 也不应该包含多个并不需要运行在同一主机上的容器](https://haochen233.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E5%BA%8A/%E5%AE%B9%E5%99%A8%E4%B8%8D%E5%BA%94%E8%AF%A5%E5%8C%85%E5%90%AB%E5%A4%9A%E4%B8%AA%E8%BF%9B%E7%A8%8B%EF%BC%8Cpod%E4%B9%9F%E4%B8%8D%E5%BA%94%E8%AF%A5%E5%8C%85%E5%90%AB%E5%A4%9A%E4%B8%AA%E5%B9%B6%E4%B8%8D%E9%9C%80%E8%A6%81%E8%BF%90%E8%A1%8C%E5%9C%A8%E5%90%8C%E4%B8%80%E5%8F%B0%E4%B8%BB%E6%9C%BA%E4%B8%8A%E7%9A%84%E5%AE%B9%E5%99%A8.png?Expires=1581996849&OSSAccessKeyId=TMP.hjZcV6CSzJbSMCFgKtHHLd6QsnwyKanaBBwDasakLa7ABzo7oeP7gBLG7ofAZtVnBeBSzPBb7weUyyouf4MkzoVCVFYeZ7CmsGvnSNrJHQyDT6qZeLud2Whig3EP2a.tmp&Signature=RKVchWL7nqoS%2BoSd2%2BZrLKppMq8%3D)  

尽管 pod 可以包含多个容器，但为了保持现在的简单性， 本章将仅讨论单容器
pod 有关的问题。

## 3.2 以YAML或JSON描述文件创建pod

pod和其他资源（对象）通常时通过向Kubernetes REST API提供JSON或YAML描述文件来创建的。此外还有其他更简单的创建资源的方法。比如之前的*`kubectl run`命令，但这些方法通常只允许你配置一组有限的属性。 另外，通过 YAML 文件定义所有的 Kubemetes 对象之后，还可以将它们存储在版本控制系统中，充分利用版本控制所带来的便利性。  

因此，为了配置每种类型资源的各种属性，我们 需要了解并理解 Kubemetes API 对象定义。 通过本书学习各种资源类型时，我们将会了解其中的大部分内容。  

### 3.2.1 检查现有pod的YAML描述文件

pod 定义由这么几个部分组成 ： 首先是 YAML 中使用的 Kubernetes API 版本和
YAML 描述的资源类型；其次是几乎在所有 Kubernetes 资源中都可以找到的三大重
要部分 ：  
- metadata 包括包括名称、命名空间、标签和关于该容器的其他信息。
- spec 包含pod内容的实际说明，例如pod的容器、卷和其他数据。  
- status 包括运行中的pod的当前信息，例如pod所处的条件、每个容器的描述和状态，以及内部IP和其他基本信息

由于status包含运行时数据等，所以在创建新的pod时，不需要提供status部分。  

这三部分展示了Kubernetes API对象的典型结构。其他对象（资源）也都具有相同的结构，这使得理解新对象相对来说更加容易。  

### 3.2.2 为pod创建一个简单的YAML描述文件


**指定容器端口**  
在pod定义中指定端口纯粹是展示性的（infomational）忽略它们对于客户端是
否可以通过端口连接到 pod 不会带来任何影响。如果容器通过绑定到地址 0.0.0.0 的端口接收连接，那么即使端口未明确列出在 pod spec 中， 其他 pod 也依旧能够连接到该端口 。但明确定义端口仍是有意义的，在端口定义下，每个使用集群的人都可以快速查看每个 pod 对外暴露的端口。明确定义端口还允许你为每个端口指定一个名称，这样一来更加方便我们使用。  

在准备manifest时，可以看Kubernetes的参考文档查看每个API对象支持哪些属性，也可以使用命令`kubectl explain 对象（资源）类型`。例如：  `kubectl explain pods` 来查看Pod支持哪些属性。  

然后根据可以深入了解各个属性的更多信息。即使用下面的命令模板 `kubectl explain 对象.属性`，例如：`kubectl explain pod.spec`。  

### 3.2.3 使用kubectl create来创建pod
使用 `kubectl create`命令从YAML文件创建pod。当然`kubectl create`命令会根据YAML或JSON文件的内容创建人任何资源（不仅仅只有pod）。例如：`kubectl create -f zch-manual.yaml` 会根据zch-manual.yaml文件，创建pod  

**得到运行中pod的完整定义**  

可以使用下面的命令来查看pod的完整描述文件，可以输出为yaml类型或json类型的。输出yaml类型的`kubectl get po -o yaml` ,输出json类型的`kubectl get po -o json`。  

**在 pod 列表中查看新创建的 pod**  
前面用过很多次了，命令为`kubectl get pod`。  


### 3.2.4 查看应用程序日志
容器化的应用通常会将日志记录到标准输出和标准错误输出流，  

用以下命令获取容器日志：`docker logs 容器id或容器名` 使用用`-tf`选项，可以加上时间和持续查看。  
但是Kubernetes提供了一种更为简单的方法。  

**使用kubectl logs命令获取pod日志**  
为了查看pod的日志（准确说是容器的日志），用`kubectl logs pod的名字`  。

如果该pod只包含一个容器， 那么查看这种在Kubernetes中运行的应用程序的日志则非常简单。  

> 注意： 每天日志文件达到10MB时，容器日志都会自动轮替。  

**获取多容器pod的日志时指定容器名称**  
如果我们的pod包含多个容器， 在运行kubectl logs命令时则必须通过包
含-c <容器名称＞选项来显式指定容器名称。例如：访问名为 `zch-manual`的pod的中的名为`serv`容器，命令如下：`kubectl logs zch-manual -c serv`。

> 注意：只能获取仍然存在的pod的日志。当一个pod被删除时，它的日志也会被删除。如果希望在pod删除之后仍然可以获取其日志， 我们需要设置中心化的、 集群范围的日志系统， 将所有日志存储到中心存储中。  

### 3.2.5 向pod发送请求
我们使用kubec七l expose命令创建了一
个service, 以便在外部访问该pod。 由于有 一整章专门介绍service, 因此本章并不打算使用该方法。 此外 ， 还有其他连接到pod以进行测试和调试的方法， 其中之一便是通过**端口转发**  。  

**将本地网络端口转发到pod中的端口**

如果想要不通过服务（service）的情况下与某个特定的pod进行通信（出于调试或其他原因），Kubernetes将允许我们配置端口转发到该pod可以通`kubectl port-forward`命令完成上述操作。  

将机器的本地端口9191转发到我们的zch-manual pod的端口为9190：`kubectl port-forward zch-manual 9191:9190` ,此时端口转发正在运行。可以通过本地端口连接到我们的pod。  

## 3.3 使用标签组织标签
部署实际应用程序时， 大多数用户最终将运行更多的pod。 随着pod数量的增加，将它们分类到子集的需求也就变得越来越明显了。  

例如， 对于微服务架构， 部署的微服务数量可以轻松超过20个甚至更多。 这
些组件可能是副本（部署同一组件的多个副本）和多个不同的发布版本(sable、
bea、canary等）同时运行。 这样一来可能会导致我们在系统中拥有数百个pod,果没有可以有效组织这些组件的机制， 将会导致产生巨大的混乱。  

很明显， 我们需要一种能够基于任意标准将上述pod组织成更小群体的方式，
这样一来处理系统的每个开发人员和系统管理员都可以轻松地看到哪个pod是什么。此外， 我们希望通过一次操作对属于某个组的所有pod进行操作， 而不必单独为每个pod执行操作。  

通过**标签**来组织pod和所有其他Kubernetes对象。  

### 3.3.1 介绍标签
标签是一种简单却功能强大的**Kubernetes**特性，不仅可以组织pod，也可以组织其他所有的Kubernetes资源。详细来讲，标签是可以附加到资源的任意键值对，用以选择具有该确切标签的资源（这是通过标签选择器完成的 ）。只要标签的key在资源内是唯一的，一个资源便可以拥有多个标签（即标签对一个资源来说不能重复）。通常我们创建资源时就会将标签附加到资源上，但之后我们也可以再添加其他标签，或者修改现有标签的值，而无须重新创建资源。  

可以通过给pod添加标签，可以得到一个更组织化的系统，以便我们理解。此时每个pod都有两个标签：  
- app：它指定pod属于哪个应用、 组件或微服务。
- rel：它显示在pod中运行的应用程序版本是stable、beta还是canary。  

如图：  
![使用pod标签组织微服务架构中的pod](https://haochen233.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E5%BA%8A/%E4%BD%BF%E7%94%A8pod%E6%A0%87%E7%AD%BE%E7%BB%84%E7%BB%87%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84%E4%B8%AD%E7%9A%84pod.png?Expires=1582015625&OSSAccessKeyId=TMP.hjZcV6CSzJbSMCFgKtHHLd6QsnwyKanaBBwDasakLa7ABzo7oeP7gBLG7ofAZtVnBeBSzPBb7weUyyouf4MkzoVCVFYeZ7CmsGvnSNrJHQyDT6qZeLud2Whig3EP2a.tmp&Signature=cagYalvWW%2Bx2%2B10lICXvSnxJ2Oo%3D)
，通过添加这两个标签基本上可以将pod组织为两个维度（基于应用的横向维度和基于版本的纵向维度）。  

每个可以访问集群的开发或运维人员都可以通过查看pod标签轻松看到系统的
结构，以及每个pod的角色。  

### 3.3.2 创建pod时指定标签
使用下面代码清单中的内容创建一个名为zch-manual-with-lables.yaml的新文件。  
```yaml
apiVersion: vl 
kind: Pod 
metadata:
  name: kubia-manual-v2 
  labels: 
    creaion method: manual
    env: alpine 
spec: 
  containers:
  - image: go_serv
    name:test_serv1  
    ports:
    - containerPort：5500
    protocol: TCP 
```

添加了标签，一个为创建方法是yaml手册，一个是环境为alpine。 

`kubectl get pods` 命令默认不会列出任何标签， 但我们可以使用 `--show­ labels` 选项来查看：  

如果你只对某些标签感兴趣， 可以使用 -L 选项指定它们并将它们分别显示在
自己的列中， 而不是列出所有标签。如下：`kubectl get po -L creation_method,env`  

### 3.3.3 修改现有pod的标签
标签也可以在现有 pod上进行添加和修改。使用下面的命令：`kubectl lable po zch-manual creation_method=manual`，为名为zch-manual的pod添加creation_method标签。  

还可以更改现有标签的值，需要使用在后面加上`--overwrite`选项，例如：`kubectl lable po zch-manual env=debug --overwrite`。  

还可以删除已有的标签,例如：删除env标签`kubectl label po zch-manual env-`,不需要使用`--overwrite`。

修改后在查看更新的标签`kubectl get po -L creation_method,env`。  

## 3.4 通过标签选择器列出子集
标签选择器允许我们选择标记有特定标签的pod子集， 并对这些pod执行操作。 可以说标签选择器是一种能够根据是否包含具有特定值的特定标签
来过滤资源的准则。

标签选择器根据资源的以下条件来选择资源：  
- 包含（或不包含）使用特定键的标签
- 包含具有特定键和值的标签
- 包含具有特定键的标签， 但其值与我们指定的不同  


### 3.4.1 使用标签选择器列出pod
列出所有标签creation_method为manual)的pod
命令`kubectl get po -l creation_method=manual`  

列出所有包含env标签的pod：`kubectl get po -l env`

列出所有不包含env标签的pod：  
`kubectl get po -l '!env'`  

注意：确保使用单引号''来圈引!env，因为!在bash中有特殊含义。  


同理还可以将pod与以下标签选择器进行匹配，例如：  
- creation_method!=manual选择带有creation_method标签，并且
值不等于manual的pod  

- env in (prod,devel)选择带有env标签且值为prod或devel的pod。（相当于值为集合里面的任意一个）  

- env notin （prod，devel）选择带有env标签值为prod或devel的pod。（相当于值不为集合里面的任意一个）  

### 3.4.2 在标签选择器中使用多个条件

可以使用逗号将选择器分隔开开，例如：`app=pc, rel=beta`  

标签选择器不仅帮助我们列出pod, 在对一个子集中的所有pod都执行操作时也具有重要意义。  


## 3.5 使用标签和选择器来约束pod调度

迄今为止， 我们创建的所有pod都是近乎随机地调度到工作节点上的。这恰恰是在Kubemetes集群中工作的正确方式。由于
Kubemetes将集群中的所有节点抽象为一个整体的大型部署平台， 因此对你的pod实际调度到哪个节点而言是无关紧要的。

我们不会特别说明pod应该调度到哪个节点上， 因为这将会使应用程序与基础
架构强耦合， 从而违背了Kubemetes对运行在其上的应用程序隐藏实际 的基础架构的整个构想。但如果你想对一个pod应该调度到哪里拥有发言权， 那就不应该直接指定一个确切的节点， 而应该用某种方式描述对节点的需求， 使Kubemetes选择一个符合这些需求的节点。这恰恰可以通过节点标签和节点标签选择器完成。  


### 3.5.1 使用标签分类工作节点
pod并不是唯一可以附加标签的Kubemetes资源。 标签可以附加到
任何Kubemetes对象上，包括节点。

运维团队向集群添加**新节点**时，
他们将通过附加标签来对节点进行分类， 这些标签指定节点提供的硬件类型 ，或者任何在调度pod时能提供便利的其他信息。  

设我们集群中的 一个节点刚添加完成， 它包含一个用于通用GPU计算的
GPU。我们希望向节点添加标签来展示这个功能特性， 可以通过将标签gpu = true添加到其中一个节点来实现，比如往新加的节点test1上添加gpu=true标签：`kubectl lable node test1 gpu=true`。  同以前一样使用**get -l**来查看:`kubectl get node -l gpu=true` 。

### 3.5.2 将pod调度到特定节点
假设我们想部署一个需要GPU来执行其工作的新pod。为了让调度器只
在提供适当GPU的节点中进行选择， 我们需要在pod 的YAML文件中添加一个**节点选择器**（nodeSelector）。  

创建一个zch_gpu.yaml的文件，然后`kubectl create -f zch-gpu.yaml`命令创建该pod。  
yaml文件清单如下：  
```yaml
apiVersion: v1
kind: pod
metadate:
  name: zch-gpu
spec:
  nodeSelector: 
    gpu: "true"
  containers:
  - image: zch/gpu
    name: zch
```  
我们只在**spec**部分加上了一个nodeSelector字段，当我们创建pod时，调度器将只在包含标签gpu =true的节点中选择一个节点。  

### 3.5.3 调度到一个特定节点

同样地，我们也可以将pod调度到某个确定的节点，由于每个节点都有一个
唯一标签， 其中键为kuberne七es.io/hostname, 值为该节点的实际主机名，即`seleteor为hostname=主机名`

但如果节点处于离线状态，通过hostname标签将nodeSelector设置为特定节点可能会导致pod不可调度。  

我们绝不应该考虑单个节点，而是应该通过标签选择器考虑符合特定标准的逻辑节点组。  

## 3.6 注解
除标签外，pod和其他对象还可以包含注解。注解也是键值对，所以它们本质上与标签非常相似。  

但与标签不同，注解并不是为了保存标识信息而存在的，它们不能像标签一样用于对象（资源）分组。  

当我们可以通过标签选择器选择对象时，就不存在注解选择器这样的东西。 

另一方面，注解 可以容纳更多的信息，并且主要用于工具使用。Kubemetes也
会将一些注解自动添加到对象，但其他的注解则需要由用户手动添加。

大量使用注解可以为每个pod或其他API对象添加说明，以便每个使用该集群
的入都可以快速查找有关每个单独对象的信息。例如，指定创建对象的人员姓名的
注解可以使在集群中工作的人员之间的协作更加便利。

### 3.6.1 查找对象的注解
让我们看一个Kubemetes自动添加注解到我们创建的pod的注解示例。为了查看注解，我们需要获取pod的完整YAML文件或使用 kubectl describe命令。例如：  
`kubectl get po zch-manual -o yaml`显示结果如下：  
```yaml
apiServer: v1
kind: pod
metadata:
  annotations:
    kubernetes.io/created-by:
      {"kind":"SerializedReference", "apiVersion":"v1",
      "reference":{"kind":"replicationController", "namespace":"default",...} ...#省略一些
      }
```
### 3.6.2 添加和修改注解

`kl annotate po zch-manual bes.com/someannotation="foo bar"` 往名为zch-manual的pod添加注解**bes.com/someannotation="foo bar"**。  

## 3.7使用命名空间对资源进行分组
首先回到标签的概念，我们已经看到标签是如何将 pod 和其他对象组织成组的。  

由于每个对象都可以有多个标签，因此这些对象组可以重叠。另外，当在集群中工作（例如通过 kubectl ）时，如果没有明确指定标签选择器，我们总能看到所有对象。  

### 3.7.2 发现其他命名空间及其pod
列出集群所有的命名空间：`kubectl get ns`。  

到目前为止，我们 只在 **default** 命名空间中进行操作。 当使用 `kubectl get` 命令列出资源时，我们从未明确指定命名空间，因此 kubectl 总是默认为**default**命名空间，只显示该命名空间下的对象。  

接下来可以使用 kubectl 命令指定
命名空间来列出只属于该命名空间的 pod ，如下所示为属于 kube- system 命名空间的pod:`kubectl get po --namespace kube-system`。

可以使用`-n`代替`--namespace`。

从`kube-system`命名空间的名称可以清楚地看到，这些资源与 Kubernetes系统本身是密切相关的。 通过将它们放在单独的命名空间中，可以保持一切组织良好。 如果它们都在默认的命名空间中，同时与我们自己创建的资源混合在一起，那
么我们很难区分这些资源属于哪里，并且也可能会无意中删除一些系统资源。  

namespace 使我们能够将不属于一组的资源分到不重叠的组中。如果有多个用户或用户组正在使用同一个 Kubernetes 集群，并且它们都各自管理自己独特的资源集合，那么它们就应该分别使用各自的命名空间。 这样一来，它们就不用特别担心无意中修改或删除其他用户的资源，也无须关心名称冲突。 如前所述，命名空间为资源名称提供了一个**作用域**。

### 3.7.3 创建一个命名空间
命名空间是一种和其他资源一样的 Kubernetes 资源。因此可以通过将 YAML 文件提交到 Kubernetes API 服务器来创建该资源。

文件**zch-prac-namespace.yaml**的内容如下：  
```yaml
apiVersion: v1
kind: Namespace    #kind值（资源类型）一般首字母大写
metadata:
  name: zch-prac
```

使用如下命令创建命名空间
`kubectl create -f zch-prac-namespace`。

我们还可以使用专用的 `kubectl create  namespace` 命令创建命名空间。而我们之所以选择使用 YAML 文件，只是为了强化 Kubemetes
中的所有内容都是一个 API 对象这一概念。可以通过向 API 服务器提交 YAML manifest 来实现创建、 读取、更新和删除。

### 3.7.4 管理其他命名空间中的对象
如果想要在刚创建的命名空间中创建资源（例如创建一个pod），可以选择在 metadata 宇段中添加一个 namespace: 指定namespace名字。  

除了直接写在yaml文件中，也可以在使用 `kubectl create `
命令创建资源时指定命名空间，例如：  `kubectl create -f test.yaml -n your_namespace`。

在列出、描述、修改、删除其他命名空间中对象时，需要给**kubectl**命令传递--namespace(或-n)选项。如果不指定命名空间，kubectl将在当前上下文中配置的默认命名空间中执行操作。而当前上下文的命名空间和当前上下文本身都可以通过`kubectl config`命令进行更改。

提示 要想快速切换到不同的命名空 间 ，可以设置以下别名：
```shell
 alias kcd=’ kubectl  co 口 fig set  con text  $  (kubectl  config  current -
context)  -- namespace ’ 
```
然后，可以使用 `kcd some - namespace` 在命名空间之间进行切换。  

### 3.7.5 命名空间提供的隔离
尽管命名空间将对象分隔到不同的组，只允许你对属于特定命名空间的对象进行操作，但实际上命名空间之间并不提供对**正在运行的对象**的**任何隔离**。

>你可能会认为当不同的用户在不同的命名空间中部署 pod 时，这些 pod
应该彼此隔离，并且无法通信。命名空间之间是否提供网络隔离取决于 Kubemetes 所使用的网络解决方案。当该解决方案不提供命名空间间的网
络隔离时－ ，如果命名空间 foo 中的某个 pod 知道命名空间 bar 中 pod 的 IP 地址，那它就可以将流量（例如 HTTP 请求）发送到另一个 pod。  

## 3.8 停止和移除pod

### 3.8.1 按名称删除pod
 
按名字删除pod：`kubectl delete po zch-manual`  
在删除 pod 的过程中，实际上我们在指示 Kubernetes 终止该 pod 中的所有容器。Kubernetes 向进程发送一个 SIFTERM阳信号并等待一定的秒数（默认为 30 ），使其正
常关闭。 如果它没有及时关闭，则通过 SIGKILL 终止该进程。  

提示 还可以通过指定多个空格分隔的名称来删除多个 pod （例如 ： `kubectl delete  po  podl  pod2` ）。

### 3.8.2 使用标签选择器删除pod
例如删除出所有type标签值为go的pod：`kubectl delete po -l type=go`。


### 3.8.3 通过删除整个命名空间来删除pod
这意味着，可以简单地删除整个命名空间（pod 将会
伴随命名空间 自动删除〉。 现在使用 以下命令删除zch-test命名空间：`kubectl delete ns zch-test`。  

### 3.8.4 删除命名空间中的所有pod，但保留命名空间

使用如下命令删除默认命名空间中的所有pod：  
`kubectl delete po --all`

但是使用`kubectl run `命令创建的pod，是由rc创建的并保持渴望的数量的，你删除一个一个，rc就会自动补充一个新的pod。所以想要删除该pod，必须删除该rc。  


### 3.8.5 删除命名空间中的（几乎）所有资源
通过使用单个命令删除当前命名空间中的所有资源，可以删除 ReplicationCcontroller和pod，以及service：  
`kubectl delete all --all`。  
命令中的第一个all指定正在删除所有资源类型（不是pod 或 rc 或ns等等单个类型），而--all选项指定将删除所有资源实例。  

> 注意使用 all 关键字删除所有内容并不是真的完全删除所有内容。一些资源（比如：secret）会被保留下来，并且需要被明确指定删除。

> 注意 kubectl delete  all  --all 命令也会删除名为 kubernetes 的Service,  但它应该会在几分钟后自动重新创建。  

