### 本章内容涵盖
- 了解认证机制
- Service Accounts 是什么及使用的原因
- 了解基于角色（RBAC）的权限控制插件
- 使用角色和角色绑定
- 使用集群角色和集群角色绑定
- 解默认角色及其绑定


## 12.1 了解认证机制

目前有几个认证插件是直接可用的。它们使用下列方法获取客户端的身份认证：  
- 客户端证书
- 传入在 HTTP 头中的认证 token
- 基础的 HTTP 认证
- 其他

启动API服务器时，通过命令行选项可以开启认证插件。

---
### 12.1.1 用户和组
认证插件会返回（返回给API 服务器）已经证过用户的用户名和组（多个组）。k8s不会在任何地方存储这些信息。这些信息被用来验证用户是否被授权执行某个操作。

### 了解用户
k8s区分了两种连接到API服务器的客户端：  
- 真实的人（用户）
- pod（准确的说是运行在pod中的应用）

这两种类型的客户端都使用了上述的认证插件进行认证。用户应该被管理在外部系统中，但是pod使用一种称为`service accounts`的机制。该机制被创建和存储在集群中作为`serviceaccount`资源。相反没有该资源代表用户账户，也就意味着不能通过API服务器来创建、更新和删除用户。

---
### 了解组
正常用户和 ServiceAccount 都可以属于一个或多个组。己经讲过认证插件会连同用户名和用户 ID 返回组，而不是必须单独给用户赋予权限。  

由插件返回的组仅仅是表示组名称的字符串，但是系统内 置的组会有一些特殊的含义。  
- system : unauthenticated 组用于所有认证插件都不会认证客户端身份的请求。
- system : authenticated 组会自动分配给一个成功通过认证的用户。
- system:serviceaccounts 组包含所有在系统中的 Serv iceAccount 。
- system:serviceaccounts:<namespace>组包含了所有在特定命名空间中的ServiceAccount。


### 12.1.2 ServiceAccount介绍
每个pod都与一 个ServiceAccount相关联 它代表了运行在pod中应用程序的身份证明。token文件持有ServiceAccount的认证token。 应用程序使用这个 token连接API服务器时 身份认证插件会对ServiceAccount进行身份认证 并将ServiceAccount的用户名传回API服务器内部。ServiceAccount用户名的格式像下面这样：  
`system:serviceaccount:<namespace>:<service account name>`

API服务器将这个用户名传给己配置好的授权插件这决定该应用程序所尝试执行的操作是否被ServiceAccount允许执行 。

ServiceAccount只不过是一种运行在pod中的应用程序和API服务器身份认证 的一种方式。如前所述 应用程序 通过在请求中传递ServiceAccount token来实现这 一点。

---
### 了解ServiceAccount资源
ServiceAccount（简写为sa）就像Pod、Secret、 ConfigMp等一 样都是资源 它们作用在单独的命名空间 为每个命名空间自动创建一个默认的ServiceAccount。

当前命名空间只包含default ServiceAccount,  其他额外的ServiceAccount可以在需要时添加。 每个pod都与一个ServiceAccount相关联 但是多个pod可以使用同一个ServiceAccount。

pod不能跨命名空间使用serviceaccount，只能使用本命名空间的service1account

---
### ServiceAccount 如何和授权进行绑定
在pod的manifest 文件中，可以用指定账户名称的方式将一个ServiceAccount赋值给一个pod。如果不显式指定ServiceAccount的账户名称，则pod会只使用这个命名空间中默认的ServiceAccount。

可以通过将不同的 ServiceAccount 赋值给 pod 来控制每个pod可以访问的资源。当 API 服务器接收到一个带有认证 token 的请求时， 服务器会用这个token 来验证发送请求的客户端所关联的 ServiceAccount 是否允许执行请求的操作。 API 服务器通过管理员配置好的系统级别认证插件来获取这些信息。 其中一个现成的授权插件是基于角色控制的插件(RBAC)。

---
### 12.1.3 创建ServiceAccount
我们已经讲过每个命名空间都拥有一个默认的 ServiceAccount, 也可以在需要时创建额外的 ServiceAccount。但是为什么应该费力去创建新的 ServiceAccount而不是对所有的pod都使用默认的 ServiceAccount？

显而易见的原因是集群安全性。 不需要读取任何集群元数据的pod 应该运行在一个受限制的账户下， 这个账户不允许它们检索或修改部署在集群中的任何资源。需要检索资源元数据的pod应该运行在只允许读取这些对象元数据的ServiceAccount下。 反之， 需要修改这些对象的pod 应该在它们自己的 ServiceAccount 下运行， 这些 ServiceAccount允许修改 API对象。

---
### 创建ServiceAccount
得益于`Kubectl create serviceaccount`命令，，创建ServiceAccount非
常容易。让我们新创建一个名为zch的ServiceAccount。

使用kd命令查看创建的sa，看到已经创建了自定义的token密钥，并将它和ServiceAccount相 关联，然后可以使用kd命令查看token密钥里的数据（包含了和默认serviceaccount相同的条目（值当然不同）：CA证书、命名空间和token）。

---
### 了解ServiceAccount上的可挂载密钥
过使用kubectl describe命令查看Se1-viceAccount时，token会显示在可
挂载密钥列表中。下面让我们来解释一下这个列表代表什么。

在默认情况下，pod可以挂载任何它需要的密钥。但是 我们可以通过对ServiceAccount进行配置，让 pod 只允许挂载ServiceAccount 中列出的可挂载密钥。为了开启这个功能，ServiceAccount必须包含以下注解： kubernetes.io/enforce-mountable-secrets= "true"。

如果 ServiceAccount 被加上了这个注解，任何使用这个ServiceAccount的 pod只能挂载进 ServiceAccount的可挂载密——这些pod不能使用其他的密钥。

---
### 了解ServiceAccount的镜像拉取密钥
ServiceAccount 也可以包含镜像拉取密钥的list。镜像拉取密钥持有从私有镜像仓库拉取容器镜像的凭证。

例如：  
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-zch
imagePullSecrets:  
- name: my-dockerhub-secret
```

ServiceAccount的镜像拉取密钥和它的可挂载密钥表现有些轻微不同。和可挂载密钥不同的是，ServiceAccount 中的镜像拉取密钥不是用来确定一个pod可以使用哪些镜像拉取密钥的。添加到 ServiceAccount 中的镜像拉取密钥会自动添加到所有使用这个ServiceAccount的 pod中。向 ServiceAccount 中添加镜像拉取密钥可以不必对每个pod 都单独进行镜像拉取密钥的添加操作。

---
### 12.1.4 将ServiceAccount分配给pod
在创建另外的 ServiceAccount 之后，需要将它们赋值给 pod。通过在 pod定义
文件中的 spec.serviceAccountName 字段上设置 ServiceAccount的名称来进行
分配。

> 注意 pod的ServiceAccount必须在pod1创建时进行设置，后续不能被修改

---
### 创建使用自定义ServiceAccount的pod
创建一个使用非默认ServiceAccount的pod，例如：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test1
spec:
  serviceAccountName: zch
  containers:
  - name: cc1
    image: 192.168.2.7:5000/go_serv:2.0
    stdin: true
    tty: true
```

---
### 使用自定义额ServiceAccount和API服务器进行通信


---
### 通过基于角色的权限控制加强集群安全 

在Kubernetes1.8.0版本中，RBAC授权插件升级为GA(通用可用性），并且在很多集群上默认开启（通过kubeadm部署的集群）。

RBAC会阻止未授权的用户查看和修改集群状态。除非你授予默认的ServiceAccount额外的特权，否则默认的ServiceAccount不允许查看集群状态，更不用说以任何方式去修改集群状态。要编写和Kubernetes API服务器通信的APP，需要了解如何通过RBAC的资源管理授权。

---
### 12.2.1介绍RBAC授权插件
Kubernetes API 服务器可以配置使用一个授权插件来检查是否允许用户请求的动作执行。 因为API 服务器对外暴露了 REST 接口， 用户可以通过向服务器发送HTTP 请求来执行动作， 通过在请求中包含认证凭证来进行认证（认证 token、用户名和密码或者客户端证书）。

---
### 了解动作
REST 客户端发送 GET、 POST、 PUT、DELETE 和其他类型的 HTTP请求到特定的 URL 路径上， 这些路径表示特定的REST资源。在k8s中，这些资源时Pod、Service、Secret等等。以下是k8s请求动作的一些例子：  
- 获取pod
- 创建服务
- 更新密钥

这些动词（获取、更新、创建：get、create、update）映射到客户端请求的HTTP方法（GET、POST、PUT）上。名词（pod、service、secret）显然是映射到k8s上的资源。

认证动词和HTTP方法之间的映射关系：  
| HTTP方法 | 单一资源的动词 | 集合的动词 |
| :---: | :---: | :---: |
| GET、HEAd | get（以及watch用于监听） | list（以及watch） |
| POST | create | n/a |
| PUT | update | n/a |
| PATCH | patch | n/a |
| DELETE | delete | deletecollection |

---
>除了可以对全部资源类型应用安全权限， RBAC 规则还可以应用于特定的资源实例（例如， 一个名为myservice 的服务），并且后面你会看到权限也可以应用于non-resource  (非资源） URL路径， 因为并不是API 服务器对外暴露的每个路径都映射到一个资源（例如 /api路径本身或服务器健康信息在的路径 /healthz)。


---
### 了解RBAC插件
顾名思义， RBAC 授权插件将用户角色作为决定用户能否执行操作的关键因素。主体（可以是一个人、一个ServiceAccount，或者一组用户或ServiceAccount）和一个或多个角色相关联每个角色被允许在特定的资源上执行特定的动词。

如果一个用户有多个角色， 他们可以做任何他们的角色允许他们做的事情。

如
果用户的角色都没有包含对应的权限， 例如， 更新密钥， API 服务器会阻止用户3个 “他的”对密钥执行PUT或PATCH请求。

通过 RBAC 插件管理授权是简单的， 这一切都是通过创建四种RBAC 特定的Kubernetes资源来完成的。

---
### 12.2.2 介绍RBAC资源
RBAC 授权规则是通过四种资源来进行配置的， 它们可以分为两个组：  
- Role（角色）和ClusterRole（集群角色），他们指定了在资源上可以执行哪些动词。

- RoleBinding（角色绑定）和ClusterRoleBinding（集群角色绑定），他们将上述角色绑定到特定的用户、组或ServiceAccount上

> 角色决定了可以做什么操作、角色绑定决定了**谁**可以做这些操作。

角色和集群角色， 或者角色绑定和集群角色绑定之间的区别在于角色和角色绑定是**命名空间的资源**，而集群角色和集群角色绑定是**集群级别的资源**（不是命名空间的）。

Role（角色）授予权限，同时RoleBinding将Role绑定到主体上。

>是什么角色，就有角色对应的权限

多个角色绑定可以存在于单个命名空间中（对于角色也是如此）。 同样地， 可以创建多个集群绑定和集群角色。

尽管角色绑定是在命名空间下的， 但它们也可以引用不在命名空间下的集群角色。

在研究RBAC资源是怎样通过API服务器影响你可以执行什么操作之前， 需要确定RABC在集群中已经开启。首先， 确保使用的Kubemees在1.6版本以 上，并且RBAC插件是唯一的配置生效的授权插件。（可以同时并行启用多个插件，如果其中一个插件允许执行某个操作，那么这个操作就会被允许）。

### 12.2.3 使用Role和RoleBinding
Role资源定义了哪些操作可以在哪些资源上执行

下面定义一个role，它允许用户获取并列出default命名空间中的服务：  
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: zch-prac # Role所在的命名空间如果没指定则使用当前的命名空间。
  name: svc-reader
rules:
- apiGroups: [" "] #Serrvice是核心apiGroup的资源，所以没有apiGroup名就是“”
  verbs: ["get", "list"] #获取独立的Service (通过名字）并且列出所有允许的服务
  resources: ["services"] # 这条规则和服务相关（必须使用复数的名字！）
```

> 警告：在指定资源时必须使用复数的形式。

在角色定义中，需要为定义包含的每个规则涉及的资源指定apiGroup。如果你允许访间属于不同API组的资源，可以使用多种规则。

在上面Role的manifest中，你允许访问所有服务资原，但是也可以通过额外的resourceNames字段指定服务实例的名称来限制对服务实例的访问。

---
### 创建角色
`kl create 0f svc-reader.yaml -n zch-prac`
在命名空间zch-prac中创建该role。

这个角色允许你在pod中列出zch-prac命名空间中的服务。仅仅创建Role还不够，需要将角色绑定到命名空间中的ServiceAccount上。

---
### 绑定角色到 ServiceAccount
必须将Role绑定到一个主体。

通过创建一个RoleBinding资源来实现将角色绑定到主体。

运行下面的命令将角色绑定到`default`ServiceAccount：  
`kl create roleBinding test --role=svc-reader --serviceaccount=zch-prac:default -n zch-prac` 

该命令会创建一个RoleBinding资源，它将svc-reader角色绑定到命名空间zch-prac中的`default`ServiceAccount上。

注意： 如果要绑定一个角色到一个 user （用户）而不是 ServiceAccount 上， 使用--user 作为参数来指定用户名。 如果要绑定角色到组，可以使用 --group 参数。

---
可以查看创建的RoleBinding的yaml格式：  
`kl get rolebinding test -n zch-prac -o yaml`


但是只有被该RoleBinding绑定的serviceaccount运行的pod才能获取和列出zch-prac中的services。还可以将该RoleBinding绑定到另一个命名空间中的ServiceAccount，则使用该ServiceAccount的pod就可以获取和列出zch-prac中的services。

---
### 12.2.4 使用ClusterRole和ClusterRoleBinding

Role 和 RoleBinding 都是命名空间的资源，这意味着它们属于和应用在一个单一的命名空间资源上。但是，如我们所见， RoleBinding 可以引用来自其他命名空间中的 ServiceAccount 。

一个常规的角色只 允许访 问 和角色在同一命名空间中的资源。 如果你希望允许跨不同命名空间 访问资源，就必须要在每个命名空间中创建一个 Role 和RoleBinding 。 如果你想将这种行为扩展到所有的命名空间（集群管理员可能需要 ），需要在每个命名空间中创建相同的Role 和 RoleBinding 。 当创建一个新的命名空间时，必须记住也要在新的命名空间 中创建这两个资源。

Cluster Role 是一种集群级资源，它允许访问没有命名空间的资源和非资源型的URL，或者作为单个命名空间内部绑定的公共角色，从而避免必须在每个命名空间中重新定义相同的角色。

---
### 允许访问集群级别的资源
允许pod列出集群中的PersistentVolume。首先创建一个pv-reader的ClusterRole：  
`kl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes`

> 注意:使用RoleBinding引用一个ClusterRole，不会授予集群级别的资源的权限，必须始终使用ClusterRoleBinding来对集群级别的资源进行授权访问。

创建ClusterRoleBinding：  
`kl create clusterrolebinding crb --clusterrole=pv-reader --serviceaccount=zch-peac:sa-t`


>在授予集群级别的资源访问权限时，必须使用一个 ClusterRole
和一个 ClusterRoleBinding 。

>记住一个 RoleBinding 不能授予集群级别的资源访问权限，即使它引用了
一个 ClusterRoleBinding。

---
### 允许访问非资源型的 URL
API 服务器也会对外暴露非资源型的 URL。访问这些 URL 也必须要显式地授予权限－否则， API 服务器会拒绝客户端的请求。 通常， 这个会通过 system : discovery  ClusterRol e 和相同命名的 ClusterRoleBinding 帮你自动完成

用下面命令查看cr可以发现很多clusterrole
`kl get clusterrole -n zch-prac`

kd clusterrole system:discovery可以发现ClusterRole引用的是 URL 路径而不是资源。

> 注意：对于非资源型 URL ，使用普通的 HTTP 动词，如post 、 put 和 patch,而不是 create 或 update 。 动词需要使用 小写的形式指定。

和集群级别的资源一样，非资源型的 URL ClusterRole 必须与 ClusterRo leBinding结合使用。把它们 和 RoleBinding 绑定不会有任何效果。

查看system:discovery clusterrole对应的clusterrolebinding。

---
### 使用ClusterRole 来授权访问指定命名空间中的资源

ClusteRole不是必须一直和集群级别的ClusteRoleBinding捆绑使用。 它们也可以和常规的有命名空间 RoleBinding进行捆绑。

让我们了解另一个名为view 的ClusteRole。

查看view：  
`kl get clusterrole view -o yaml`

这个ClusterRole有很多规则,这些规则允许 get 、list和 watch 资源，这些资源是有命名空间的， 即使我们正在了解的是个ClusterRole。这个 ClusterRole到底做了什么？

这取决于它是和ClusterRoleBinding 还是和RoleBinding 绑定。如果如果你创建了一个ClusterRoleBinding 并在它里面引用了 ClusterRole，在被绑定中列出的主体可以在所有命名空间中查看指定的资源。

相反， 如果你创建的是一个RoleBinding, 那么在绑定中列出的主体只能查看在RoleBinding 命名空间中的资源。


>即ClusterRoleBinding 和 ClusterRole 授予跨所有命名空间的资源权限。

>指向一个Cluste「Role的RoleBinding只授权获取在RoleBinding命名空间中的资源。

---
### 总结Role、ClusterRole、 Rolebinding和ClusterRoleBinding的组合

| 访问的资源 | 使用的角色类型 | 使用的绑定类型 |
| :---: | :---: | :---: |
| 集群级别的资源（no、 pv...） | ClusterRole | ClusterRoleBinding |
| 非资源型URl | ClusterRole | ClusterRoleBinding |
| 在任何命名空间中的资源（和跨所有命名空间的资源） | ClusterRole | ClusterRoleBinding |
| 在具体命名空间中的资源（在多个命名空间中重用这个ClusterRole） | ClusterRole | RoleBinding |
| 在具体命名空间中的资源（Role需要在每个命名空间中定义） | Role | RoleBinding |

---
### 了解默认的 ClusterRole 和 ClusterRoleBinding

### 用view ClusterRole 允许对资源的只读访问
它允许读取一个命名空间中的大多数资源， 除了Role、 RoleBinding和Secret。

---
###  用editClusterRole允许对资源的修改

接下来是ed江ClusterRole, 它允许你修改一个命名空间中的资源， 同时允许读取和修改Secret。 但是 它也不允许查看或修改Role和RoleBinding, 这是为了防止权限扩散。

### 用admin ClusterRole赋予一个命名空间全部的控制权

一个命名空间中的资源的完全控制权是由admin ClusterRole赋予的。有这个
ClusterRole的主体可以读取和修改命名空间中的任何资源，**除了ResourceQuota和命名空间资源（ns）本身**

edit和adminClusterRole 之间的主要区别是能否在命名空间中查看和修改Role和RoleBinding

> 注意为了防止权限扩散 API服务器只允许用户在已经拥有一个角色中列出的所有权限的情况下（及该用户有所有权限），创建和更新这个角色。

---
### 12.2.6 理性地授予授权权限
显然，将所有的ServceAccounts赋予c luster-admin ClusterRole是一个坏主意。和安全问题一样， 我们最好只给每个人提供他们工作所需要的权限，一个单独权限 也不能多（最小权限原则）

---
### 为每个pod创建特定的ServiceAccount

一个好的 想法是为每 一个pod(或一 组pod的副本）创建一个特定ServiceAccount, 并且把它和一个定制的Role (或ClusterRole)通过RoleBindng联系起来。

### 假设你的应用会被入侵

你的目标是减少入侵者获得集群控制的可能性 。现在复杂的应用程序会包含很多的漏洞， 你应该期望不必要的用户最终获得ServceAccount 的身份认证token。因 此你应该始终限制ServceAccount, 以防止它 们造成任何实际的伤害