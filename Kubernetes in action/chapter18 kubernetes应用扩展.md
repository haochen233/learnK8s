### 本章内容涵盖
- 在 Kubernetes 上添加自定义对象
- 为自定义对象添加控制器
- 添加自定义 API 服务器
- 使用 Kubernetes 服务目录完成自助服务配置
- 红帽（ Red Hat）容器平台 OpenShift 介绍
- Deis Workflow 与 Deis Helm

---
本章是本书的最后一章。 作为总结，我们介绍如何自定义 API 对象，并为这些对象添加控制器。除此之外，我们还会了解基于 Kubernetes 的优秀平台即服务(platform-as-a-Service,  PaaS ）解决方案。

---
## 18.1 定义自定义API对象

---
### 18.1.1 CustomResourceDefinition
开发者只需向 Kubernetes API 服务器提交 CRD 对象， 即可定义新的资源类型。成功提交 CRD 之后， 我们就能够通过 API 服务器提交 JSON 清单或 YAML 清单的方式创建自 定义资源，以及其他 Kubernetes 资源实例。

> 注意： 在 Kubernetes 1.7 之前的版本中 ，需要通过 ThirdPartyResource 对象的方式定义自定义资源。 ThirdPartyResource 与 CRD 十分相似，但在 Kubernetes 1.8 中被CRD取代。

开发者可以通过创建 CRD 来创建新的对象类型。 但是，如果创建的对象无法在集群中解决实际问题，那么它就是一个无效特性。 通常， CRD 与所有 Kubernetes核心资源都有一个关联控制器 （ 一个基于自定义对象有效实现目标的组件）（比如rs控制器、deploy控制器，ep控制器） 。

因此，只有在部署了控制器之后，开发者才能真正知道CRD所具有的功能远不止添加自定义对象实例而已。

---
### CRD 范例介绍

如果你想让自己的 Kubernetes 集群用户不必处理 pod、服务以及其他 Kubernetes资源， 甚至只需要确认网站域名 以及网站中的文件 （ HTML、 css 、 PNG， 等等〉就能以最简单的方式运行静态网站。这时候，你需要一个Git存储库当作这些文件的来源（挂载gitrepo类型的卷）。

当用户创建网站资源实例时，你希望 Kubernetes 创建一个新的 web 服务器pod，并通过服务将它公开。

> 每个网站资源实例，都会产生一个服务和一个HTTP服务器pod。

虚构的资源大致如下：  
```yaml
kind: Website
metadata:
  name: w
spec:
  gitRepo: https://github.com/haochen233/website-example.git

```
当然现在不能使用Website资源，kubernetes并不识别它。

---
### 创建一个CRD对象
要想k8s接收你自定义的资源实例，必须向k8s apiserver 提交CRD

例如：  
```yaml
apiVersion: apiextentions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.extentions.example.com
spec:
  sope: Namespaced  #期望Website这种资源属于某个命名空间下的
group: extensions.example.com
version: v1         #定义API集群和网站资源版本
names:
  kind: Website
  singular: wesite  #单数形式
  plural: websites  #复数形式
```

理解API组和API组的版本：  
> 如你所见，在创建 Deployment 对象时，你需要将 apiVersio口设置为 apps/v1，其中“/”前的部分为API组，后面的为API组的版本名。

在上面的清单中我们指定了API组为extensions.example.com，版本名为v1，所以后面使用Website资源创建实例时，需要指定字段apiVersion为`apiVersion extentions/example.com/v1`。

---
创建好CRD后，就可以创建该自定以资源了。

---
可以通过`kl get `命令查看实例
包括一系列操作删除、创建等等。

---
### 18.1.2 使用自定义控制器自动定制资源
为了让你的网站对象运行一个通过服务暴露的 web 服务器 pod，你就需要构建
和部署一个网站控制器（用来创建对应资源的，如rs控制器创建pod）。

控制器将创建deploy资源，而不是直接创建非托管pod ，这样就能确保pod 既能被管理，还能在遇到节点故障时继续正常工作。

当准备将控制器部署到生产环境中时， 最好的方法是在 Kubernetes 内部运行控制器，就像所有其他核心控制器那样。要在 Kubernetes 中运行控制器，可以通过Deployment 资源进行部署。

控制器是一个双容器pod，其中一个是主容器，另一个是porxy。主容器通过proxy间接访问API服务器，进行对Website对象的操作（创建或删除等等）

在Deployment过程中部署了 一个双容器pod副本。其中一个容器用于运行你的控制器，而另一个容器则是用于与API服务器进行简单通信的ambasador。

这个pod在其特殊的服务账户 下运行，因此需要在部署控制器之前创建一个服务账户：website-controller。

如果集群启用了基于角色的访问控制（RBAC），kubernetes将不允许控制器查看网站资源或创建deploy或svc。需要使用集群角色绑定，将website-controller绑定到cluster-admin的集群角色上。

---
### 18.1.3 验证自定义对象
可能你已经发现， 你还没有在网站CRD中指定任何类型的验证模式。 这会导致你的用户可以在网站对象的YAML中包含他们想要的任何字段。 由于API服务器并不会验证YAML的内容(apiversion、kind 和metadata等常用字段除外），用户创建的Website对象就有可能是无效对象。

既然如此， 我们是否可以向控制器添加验证并防止无效的对象被API服务器接收？事情并没有这么简单。

因为API服务器首先需要存储对象， 然后向客户端(kubectl)返回成功响应， 最后才会通知所有监听器（包括你的控制器）。

显然，你希望的是API服务器在验证对象时立即拒绝无效对象。

在Kubernetes1.8版本中， 自定义对象的验证作为alpha特性被引入。 如果想要让API服务器验证自定义对象，需要在API服务器中启用CustomResourceValidation特性，并
在CRD中指定一个JSON schema。

---
### 18.1.4 为自定义对象提供自定义API服务器
如果你想更好地支待在Kubernetes 中添加自定义对象， 最好的方式是使用你自己的API服务器，并让它直接与客户端 进行交互。

---
### API服务器聚合
在Kubernetes 1.7版本中， 通过API服务器聚合，可以将自定义API服务器与主KubernetesAPI服务器进行集成。

从此，客户端可以被连接到聚合APL并直接将请求转发给相应的API服务器。这样一来，客户端甚至无法察觉幕后有多个API服务器正在处理不同的对象。

通过聚合后，这些**被拆分**的API服务器会作为单独服务器被暴露（只会暴露一个主服务器）。

对你来说，可以创建一个专门负责处理你的网站对象的API服务器，并使它参照Kubernetes API核心服务器验证对象的方式来验证你的网站（自定义对象）。这样你就不必再创建CRD来表示这些对象， 就可以**直接将网站对象类型实现到你的自定义API服务器中。**

通常来说，每个API服务器会负责存储它们自己的资源。它可以运行自己的etcd实例（或整个etcd集群），也可以通过创建CRD实例将其资源存储在核心API服务器的etcd存储中。在这种情况下，就需要先创建一个CRD对象，然后才能创建CRD实例，正如在之前所做的那样。

---
### 注册一个自定义 API 服务器

如果你想要将自定义API服务器 添加到集群中，可以将其部署为一个pod并 通过 Service暴露。下 一步，**为了将它集成到主API服务器中**，需要部署一个描述APIService资源的YAML列表。

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.extensions.example.com
spec:
  group: extensions.example.com
  version: v1alpha1
  priority: 150
  service:              #定制API服务器暴露的服务
    name: website-api  
    namespace: defualt   
```

在创建上述的APIService资源后，被发送到主API服务器的包含**extensions.example.com** API组任何资源的客户端请求，会和v1alpha1版本号一起被转发到通过website-api Service公开的自定义API服务器pod（通过服务将请求发到运行apiserver的pod上）。

---
注册！
注册！
注册！

重要的事说三遍，自定义apiserver已经以pod形式在运行了，字段中的service是自定义apiserver被暴露的服务。

---
## 18.2 使用kubernetes服务目录扩展kubernetes
第一个通过API服务器 **aggregation**加入Kubernetes的附加 API服务器 是**服务目录**API服务器。服务目录是Kubemetes社区的热门话题之一。

目前，对于使用服务的pod（这里的服务与service资源无关），有人需要部署提供服务的pod、一个Service资源，可能还需要一个可以让客户端pod用来同服务器进行身份认证 的密钥。 通常是与部署客户端pod相同的一个用户， 或者如果一个团队专
门部署这些类型的通用服务。

那么用户需要提交一个票据， 并等待团队提供服务。这意味着用户需要为服务的所有组件创建文件， 知道如何正确配置， 并手动部署，或者等待其他团队来完成。

但是， Kubernetes显然应该是一个易于使用的自助服务系统。理想情况下，如果用户的应用需要特定的服务（例如， 需要后端数据库的web 应用程序）， 那么他只需要对Kubernetes说：＂嘿， 我需要一个PostgreSQL数据库。 请告诉我在哪里，以及如何连接到它。”想要快速实现这一功能， 你就需要使用Kubernetes 服务目录。

---
### 18.2.1 服务目录介绍
顾名思义 ， 服务目录就是列出所有服务的目录。 用户可以浏览目录并自行设置目录中列出的服务实例， 却无须处理服务运行所需的Pod、 Service、 ConfigMap和 其他资源。 这听起来与自定义网站资源很相似。

服务目录并不会为每种 服务类型的API 服务器添加自定义资源， 而是将以下四种通用API资源引入其中：  
- 一个ClusterServiceBroker, 描述一个可以提供服务的（外部）系统
- 一个ClusterServiceClass, 描述一个可供应的服务类型
- 一个Servicelnstance, 已配置服务的一个实例
- 一个ServiceBinding, 表示一组客户端(pod)和Servicelnstance之间的绑定

---

简而言之， 集群管理员为会每个服务代理创建一个ClusterServiceBroker资源。

而这些服务代理需要在集群中提供它们的服务。 接着， Kubernetes从服务代理获取它可以提供的**服务列表**， 并为它们中的每个**服务**创建一个**ClusterServiceClass**资源。

当用户调配服务时，首先需要创建一个**Servicelnstance**资源，然后创建一个**ServiceBinding**以将该**Servicelnstance** 绑定到它们的pod。

---
### 18.2.2 服务目录API服务于控制器管理器介绍
与核心Kubernetes类似的是， 服务目录也是由三个组件组成的分布式系统：  
- 服务目录API服务器
- 作为存储的etcd
- 运行所有控制器的控制器管理器

之前所介绍的四个与服务目录相关的资源是通过将 YAML/ JSON清单发布到API 服务器来创建的。 随后， API 服务器会将它们存储到自己的etcd实例中， 或者使用主 API 服务器中的CRD作为替代存储机制（在这种情况下不需要额外的etcd实例）。

使用这些资源的控制器正在控制器管理器中运行。 显然， 它们与服务目录API服务器交互的方式， 与其他核心Kubernetes控制器与核心API 服务器交互的方式相同。 这些控制器本身不提供所请求的服务， 而是将其留给外部服务代理， 再由代理通过在服务目录API 中创建ServiceBroker资源进行注册。

---
### Service代理和OpenServiceBroker API
集群管理员可以在服务目录中注册一个或多个外部ServiceBroker。同时代理都必须实施OpenServiceBroker API 。

通过OpenServiceBroker API，服务目录（整个系统）可以通过APi与ServiceBroker进行通信。这个简单的REST API 能够提供以下功能：
- 使用GET/v2/catalog
- 配置服务实例
- 更新服务实例
- 绑定服务实例
- 解绑服务实例
- 取消服务实例配置


---
### 在服务目录中注册代理
集群管理员可以通过向服务目录API发布ServiceBroker资源清单来注册代理（通过代理才能访问到外部服务）。

例如：  
```yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: ClusterServiceBroker
metadata:
  name: db-broker  #代理名称
spec:
  url: http://database-osbapi.myorganization.org #服务目录与代理连接处

```

在管理员创建ClusterServiceBroker资源后， Service Catalog Controller Manager中的控制器就会连接到资源中指定的URL, 并且检索此代理可以提供的服务列表 。在检索服务列表后， 就会为每个服务创建一个ClusterServiceClass资源。每个ClusterServiceClass资源都描述了 一种可供应的服务。

---
### 罗列集群中的可用服务
`kl get clusterserviceclasses`

---
### 18.2.4 提供服务于使用服务

提供服务实例：  
需要做的是创建 一个Servicelnstance 资源。  
```yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: ServiceInstance
metadata:
  name: my-db   #为实例命名
spec:
  clusterServiceClassName: postgres-database #需要的serviceclass和方案
  clusterServicePlanName: free
  parameters:
    init-db-args: ----data-checksums  #传递给代理的其他参数
```

可以查看实例，看是否已经成功提供服务。

---
### 绑定服务实例
想要在pod中使用 配置的Servicelnstance,需要创建ServiceBindingresource

---
### 18.2.5 解出绑定与取消配置
一旦你不再需要服务绑定， 可以按照删除其他资源的方式将其删除：
`kl delete servicebinding ...`

只会解绑，实例还在，可以创建新的服务绑定。

---
### 18.2.6 服务目录给我们带来了什么

如你所知，服务提供者可以通过在任何 Kubernetes 集群中注册代理，在该集群中暴露服务，这就是服务目录的最大作用。

总而言之，服务代理让你能够在 Kubemetes 中轻松配置和暴露服务。

---
## 18.3 基于kubernetes搭建的平台
基于 Kubernetes 构建的最著名的 PaaS 系统包括 Deis Work.flow 和 Red Hat 的
OpenShift。