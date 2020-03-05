### 本章内容涵盖
- 在pod中使用宿主节点默认的Linux命名空间
- 以不同用户身份运行容器
- 运行特权容器
- 添加或禁用容器
- 定义限制pod行为的安全策略
- 保障pod的网络安全

---
## 13.1 在pod中使用宿主节点的Linux命名空间
pod中的容器通常在分开的Linux命名空间中运行。 这些命名空间将容器中的进程与其他容器中，或者宿主机默认命名空间中的进程隔离开来。  

例如，每一个pod有自己的IP和 端口空间，这是 因为它拥有自己的网络命名空间。类似地，每一个pod拥有自己的进程树，因为它有自己的PID命名空间。pod拥有自己的IPC命名空间，仅允许同一 pod内的进程通过进程间通信(InterProcess  Communication, 简称IPC)机制进行交流。

---
### 13.1.1 在pod中使用宿主节点的网络命名空间
部分pod（特别是系统pod）需要在宿主节点的默认命名空间中运行，以允许它们看到和操作节点级别的资源和设备。例如，某个pod可能需要使用宿主节点上的网络适配器，而不是自己的虚拟网络设备。这可以通过pod spec中的hostNetwork设置为true来实现。

在这种情况下，这个pod可以使用宿主节点的网络接口，而不是拥有自己独立的网络。这意味着这个pod没有自己的IP地址；如果这个pod中的某一进程绑定了某个端口，那么该进程将被绑定到宿主节点的端口上。

> 一个配置了hostNetwork: true 的pod使用宿主节点的网络接口，而不是它自己的。


例如，配置一个使用宿主节点的网络命名空间的pod：  
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: test2
spec:
  hostNetwork: true
  containers:
  - name: c1
    image: 192.168.2.7:5000/go_serv:2.0
    stdin: true
    tty: true
```

然后可以进入该pod查看IP地址，看是否为宿主节点的IP地址。

---
### 13.1.2 绑定宿主节点上的端口而不使用宿主节点的网络命名空间

一个与此有关的功能可以让 pod 在拥有自己的网络命名空间的同时，将端口绑定到宿主节点的端口上。这可以通过配置pod的spec.containers.ports字段中某个容器的某一端口的hostPort属性来实现。

不要混淆使用 hostPort 的 pod 和通过 NodePort 服务暴露的 pod。


对于一个使用hostPort的pod，到达宿主节点的端口的连接会被直接转发到pod的对应端口上。

然而对于NodePort服务中，到达宿主节点的端口的连接将被转发到随机选取的pod上，（这个pod可能在其他节点上）。

对于使用hostPort的pod，仅有运行了这类pod的节点会绑定对应的端口；而NodePort类型的服务会在所有的节点上绑定端口，即使这个节点上没有运行对应的pod。

> 注意，如果一个pod绑定了宿主节点上的一个特定端口，每个宿主机节点只能调度一个这样的pod实例，因为两个进程不能绑定宿主机上的同一个端口。调度器在调度pod时会考虑这一点。所以他不会把这样的两个pod调度到同一个节点上。

例如：需要在3个节点上部署这样的pod副本，只有3个副本能够成功部署（剩余一个pod保持pending状态）。

下面的pod的manifest，展示了将pod绑定到宿主节点默认网络命名空间的端口：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hpp1
spec:
  containers:
  - name: c1
    image: 192.168.2.7:5000/go_serv:2.0
    ports:
    - containerPort: 8080
      hostPort: 9090
      pootocol: TCP
```

在创建这个pod之后，可以通过它所在节点的9000端口访问该pod。

> hostPort 功能最初是用于暴露通过 DeamonSet 部署在每个节点上的系统服务的。最初，这个功能也用于保证一个 pod 的两个副本不被调度到同一节点上，但是
现在有更好的方法来实现这一需求。

---
### 13.1.3 使用宿主节点的PID和IPC命名空间
pod spec 中的 hostPID 和 host IPC 选项与 hostNetwork 相似。当它们被设
置为 true 时， pod 中的容器会使用宿主节点的 PID 和 IPC 命名空间，分别允许它
们看到宿主机上的全部进程，或通过 IPC 机制与它们通信。

下面这个pod的manifest，表示该pod会使用宿主节点的PID命名空间和IPC命名空间：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pid-ipc-pod
spec:
  hostPID: true
  hostIPC: true
  containers: 
  - name: c1
    image: 192.168.2.7:5000/go_serv:2.0
    stdin: true
    tty: true
```
pod通常只能看到自己内部的进程，但是这个pod可以看到宿主机的所有进程，不仅仅只有容器内的进程。并且设置了pod的spec.hostIPC: true，表明pod中的进程可以通过进程间通信与宿主机上的其他所有进程进行通信。

---
### 13.2 配置节点的安全上下文
除了让pod使用宿主节点的Linux命名空间，还可以在 pod 或其所属容器的描
述中通过 `security- Context` 选项配置其他与安全性相关的特性。这个选项可以
运用于整个 pod ，或者每个 pod 中单独的容器。

### 了解安全上下文中可以配置的内容
配置安全上下文可以允许你做很多事：  
- 指定容器中运行进程的用户（用户ID）
- 组织容器使用root用户运行（容器的默认运行用户通常在其镜像中指定，所以可能需要组织容器以root用户运行）
- 使用特权模式运行容器，使其对宿主节点的内核具有完全的访问权限。
- 与以上相反，通过添加或禁用内核功能，配置细粒度的内核访问权限
- 设置SELinux（Security Enhaced Linux， 安全增强型Linux）选项，加强对容器的限制
- 阻止进程写入容器的根文件系统

---
### 运行pod而不配置安全上下文
首先，运行一个没有任何安全上下文配置的 pod （不指定任何安全上下文选项），与配置了安全上下文的 pod 形成对照：  
`kl run pod-with-defaults --image 192.168.2.7:5000/go_serv:2.0 --restart Never --/bin/sleep 999999`


在这个pod中查看容器中的用户ID和组ID，以及它所属的用户组。
`kl exec pod-with-defualts -- id`

> 注意： 容器运行时使用的用户在镜像中指定。在Dockerfile中，这是通过使用USER命令来实现的。如果该命令被省略，容器将使用 root 用户运行

---
### 13.2.1 使用指定用户运行容器
为了使用一个与镜像中不同的用户 ID 来运行 pod ， 需要设置该 pod 的securityContext.runAsUser选项。

下面的pod的manifest，使用haochen用户来运行一个容器：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-as-user-haochen
spec:
  containers:
  - name: hc1
    image: 192.168.2.7:5000/go_serv:2.0
    stdin: true
    tty: true
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: uid（指定haochen用户的uid，而不是用户名）  
```

运行该pod，然后在容器中使用id命令，查看用户ID和组ID，以及它所属的用户组。


---
### 13.2.2 阻止用户以root用户运行
虽然容器与宿主节点基本上是隔离的，使用 root 用户运行容器中的进程仍然是
一种不好的实践。

例如，当宿主节点上的一个目录被挂载到容器中时，如果这个容器中的进程使用了 root 用户运行，它就拥有该目录的完整访问权限 ； 如果用非 root用户运行，则没有完整权限。

为了防止以上的攻击场景发生，可以进行配置，使得 pod 中的容器以非 root 用
户运行。配置pod.spec.containers.securityContext.runAsNonRoot:true

假如，现在有一个镜像的Dockerfile中指定了以root用户运行容器，但是下面的pod 使用非root用户运行容器：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-non-root
spec:
  containers:
  - name: non-root
    image: 192.168.2.7:5000/go_serv:2.0
    stdin: true
    tty: true
    command: ["/bin/sleep", "99999"]
    securityContext: 
      runAsNonRoot: true
```

部署这个pod后，它会被成功调度，但是不允许运行。虽然镜像中指定了使用root运行，但是pod中设置了阻止容器以root用户运行。所以就出现了这样的情况。

---
### 13.2.3 使用特权模式运行pod
有时 pod 需要做它们的宿主节点上能够做的任何事，例如操作被保护的系统设备，或使用其他在通常容器中不能使用的内核功能。

这种 pod 的一个样例就是 kube-proxy pod ，该 pod 需要修改宿主机的 iptables 规则来让 Kubernetes 中的服务规则生效。

为获取宿主机内核的完整权限，该pod需要在特权模式下运行。这可以通过将pod.spec.containers.securityContext.privilleged字段设置为true来实现。

下面创建一个带有特权容器的pod：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vip
spec:
  containers:
  - name: vip
    image: 192.168.2.7:5000/go_serv
    stdin: true
    tty: true
    securityContext:
      privilleged: true
```

道 Linux 中有一个叫作 ／dev 的特殊目录，该目录包含系统中所有设备对应的设备文件。 这些文件不是磁盘上的常规文件，而是用于与设备通信的特殊文件。

通过查看对比特权容器和非特权容器所能看到得设备列表的数量（`ls /dev`），就能够明显的体会到特权容器的作用。

---
### 13.2.4 为容器单独添加内核功能
已经介绍了一种给予容器无限力量的方法。 过去， 传统的UNIX 实现只区分特权和非特权进程， 但是经过多年的发展， Linux已经可以通过内核功能支待更细粒度的权限系统。

相比于让容器运行在特权模式下以给予其无限的权限，一个更加安全的做法是只给予它使用真正需要的内核功能的权限。k8s允许为特定的容器添加内核功能， 或禁用部分内核功能， 以允许对容器进行更加精细的权限控制， 限制攻击者潜在侵入的影响。

例如，一个容器通常不允许修改系统时间（硬件时钟的时间）。可以通过在容器中，执行`date +%T -d "12:00:00"`。但是这操作是没有权限的。

如果需要允许容器修改时间，可以在容器的capbility里 add 一项名为 `CAP_SYS_TIME` 的功能：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-add-settime-capbility
spec:
  containers:
  - name: stt
    image: 192.168.2.7:5000/go_serv
    stdin: true
    tty: true
    securityContext:
      capilities:
        add:
        - SYS_TIME
```

> 注意：Linux内核功能的名称通常以CAP_开头。但在 pod.spec中指定内核功能时，必须省略 CAP_ 前缀。

然后可以在该pod中修改系统时间。

> 警告：自行尝试时，请注意这样可能导致节点不可用。

> 提示可以在Linux手册中查阅Linux内核功能列表


---
### 13.2.5 在容器中禁用内核功能
可以禁用容器中的内核功能。例如，**默认情况下容器拥有 CAP_CHOWN 权限**，允许进程修改文件系统中文件的所有者。

为了阻止容器的此种行为。在容器的`securityContext.capbilities.drop`字段添加CAP_CHOWN选项（同样需要省略CAP_前缀）。
```yaml
...
...
securityContext:
  capbilities:
    drop:
    - CHOWN
```
禁用CHOWN内核功能后， 不允许在这个容器中修改文件所有者；

现在已经对容器安全上下文的大部分选项研究完毕。

---
### 13.2.6 阻止对容器根文件系统的写入
因为安全原因你可能需要阻止容器中的进程对容器的根文件系统进行写入，仅允许它们写入挂载的存储卷。

将容器的`securityContext.radOnlyRootFilesystem`设置为true来实现，阻止容器写入自己的根文件系统（对于挂载的卷的写入时允许的）。

```yaml
...
spec:
...
  securityContext:
    readOnlyRootFilesystem: true
...
```


> 如果容器的根文件系统是只读的， 你很可能需要为应用会写入的每一个目录（如日志、 磁盘缓存等）挂载存储卷。所以为了增强安全性， 请将在生产环境运行的容器的 readOnlyRootFilesystem选项设置为true。

---
### 设置pod级别的上下文
上面的例子都是对单独的容器设置安全上下文。这些选项中的一部分也可以从pod级别设定（通过pod.spec.securityContext属性）。 它们会作为pod中每一个容器的默认安全上下文， 但是会被容器级别的安全上下文覆盖。

---
### 容器使用不同用户运行时共享存储卷
k8s允许为p od中所有容器指定supplemental组，以允许它们无论以哪个用户ID运行都可以共享文件。通过一下两个属性设置:  
- fsGroup
- supplementalGroups

这两个属性都在pod.spec.securityContext.下

安全上下文中的fsGroup属性当进程在存储卷中创建文件时起作用，而supplementalGroups属性定义了某个用户所关联的额外的用户组。

---
## 13.3 限制pod使用安全相关的特性

---
### 13.3.1 PodSecurityPolicy（简写为psp） 资源介绍
PodSecurityPolicy是一种集群级别的（无命名空间）的资源，它定义了用户能否在pod中使用各种安全相关的特性。维护PodSecurityPolicy资源中配置策略的工作由集成在API服务器中的**PodSecurityPolicy准入控制插件**完成。

> 注意： 你的集群中不一定启用了 PodSecurityPolicy 准入控制插件会将pod与已经配置的PodSecurityPolicy进行校验，如果这个 Pod 符合集群中已有安全策略它会被接收并存入 etcd; 否则它会立即被拒绝。 这个插件也会根据安全策略中配置的默认值对 pod 进行修改。

---
### 了解 PodSecurityPolicy可以做的事
一个 PodSecurityPolicy 资源可以定义以下事项：  
- 是否允许pod 使用宿主节点的PID、 IPC、 网络命名空间
- pod允许绑定的宿主节点端口
- 容器运行时允许使用的用户ID
- 是否允许拥有特权模式容器的 pod
- 允许添加哪些内核功能， 默认添加哪些内核功能， 总是禁用哪些内核功能
- 允许容器使用哪些 SELinux 选项
- 容器是否允许使用可写的根文件系统
- 允许容器在哪些文件系统组下运行
- 允许pod 使用哪些类型的存储卷
  
### 检视一个PodSecurityPolicy样例
以下代码清单展示了一个 PodSecurityPolicy 的样例。 它阻止了 pod 使用宿主节点的PID、 IPC、网络命名空间， 运行特权模式的容器， 以及绑定大多数宿主节点的端口（除10 000-11000和13000-14000范围内的端口）。 它没有限制容器运行时使用的用户、 用户组和SELinux 选项。

```yaml
apiVersion: v1
kind: PodSecurityPolicy
metadata: 
  name: test
spec:
  hostPID: false     #容器不允许使用宿主节点的PID命名空间
  hostIPC: false     #容器不允许使用宿主节点的IPC命名空间
  hostNetwork: false #容器不允许使用宿主节点的网络命名空间
  hostPorts:          
  - min: 10000       #容器只能绑定宿主节点的10000-11000和13000-14000的端口（闭区间）
    max: 11000
  - min: 13000
    max: 14000
  privilleged: false  #容器不能在特权模式下运行
  readOnlyRootFilesystem: true #容器根文件系统只读
  runAsUser:          #容器可以以任意用户和用户组运行
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:              #可以使用任何SELinux选项
    rule: RunAsAny
  volumes:              #pod可以使用所有类型的存储卷
  - '*'
```

这个PodSecurityPolicy（准入控制准则）在集群中创建成功后，API服务器将不再允许之前样例中的特权pod。

类似地，集群中不能再部署使用宿主节点的PID、IPC、网络命名空间的pod了。

---
### 13.3.2 了解 runAsUser、fsGroup和supplementalGroups策略

前面的例子中的策略没有对容器运行时可以使用的用户和用户组施加任何限制，因为它们在runAsUser、fsGroup、supplementalGroups等字段中使用了runAsAny规则。如果需要限制容器可以使用的用户和用户组ID, 可以将规则改为**MustRunAs**,并指定允许使用的ID范围。

---
### 使用 MustRunAS 规则
下面的例子，指定容器运行时必须使用的用户和用户组ID
```yaml
runAsUser:
  rule: MustRunAs
  ranges:
  - min: 2
    max: 2
fsGroup:
  rule: MustRunAs
  ranges:
  - min: 2
    max: 10
  - min: 20
    max: 30
supplementalGroups:
  rule: MustRunAs
  ranges:
  - min: 2
    max: 10
  - min: 20
    max: 30
```

如果pod spec 试图将其中任意字段设置为该范围之外的值，这个pod将不会被API服务器接收。

> ！！！注意：修改策略对已经存在的pod无效，因为PodSecurityPolicy资源仅在创建和升级Pod时起作用。

---
### 部署runAsUser在指定范围之外的pod
那么API服务器会拒绝这个pod。这是非常显然的。

如果部署pod时没有指定runAsUser属性，但用户ID被注入到镜像的情况下（即Dockerfile中使用User命令指定了范围之外的pod。），会发生什么？

答案是API服务器不会拒绝。为什么呢，PodSecurityPolicy可以将硬编码覆盖到镜像中的用户ID（即，将PodSecurityPolicy中的ID覆盖镜像中的UID）。

---
### 在runAsUser字段中使用mustRunAsNonRoot 规则
正如其名，它将阻止用户部署以 root 用户运行的容器。在此种情况下， spec 容器中必须指定 runAsUser 宇段（如果不指定，默认以root运行），并且不能为 0 （ o 为 root 用户的 ID ），或者容器的镜像本身指定了用一个非 0 的用户 ID 运行（镜像中指定了）。

---
### 13.3.3 配置允许、默认添加、禁止使用的内核功能
容器可以运行在特权模式下（privillege），也可以通过对每个容器添加或禁用 Linux 内核功能来定义更细粒度的权限配置。以下三个字段会影响容器可以使用的内核功能：  
- allowedCapbilities
- defaultAddCapbilities
- requiredDropCapbilities

在PodSecurityPolicy资源中指定内核功能：  
```yaml
apiVerison: v1
kind: PodSecurityPolicy
spec:
  allowedCapbilities:       #允许容器添加SYS_TIME功能
  - SYS_TIME
  defaultAddCapbilituies:   #为每个容器自动添加CHOWN功能
  - CHOWN
  requiredDropCapbilities:  #要求容器禁用SYS_ADMIN和SYS_MODULE ...功能
  - SYS_ADMIN
  - SYS_MODULE
  ...
```

---
### 指定容器可以添加的内核功能
allowedCapabilities 字段用于指定 spec 容器的 securityContext.capbilities中可以添加哪些内核功能。比如。之前的一个例子中容器内添加了`SYS_TIME`内核功能。如果启用了PodSecurityPolicy访问控制插件。pod中将不能添加以上核心内核功能，除非在PodSecurityPolicy中指明允许添加。

---
### 为所有容器添加内核功能
defaultAddCapabilities 宇段中列出的所有内核功能将被添加到每个己部署的 pod 的每个容器中。 如果用户不希望某个容器拥有这些功能，必须在容器的spec 中显式地禁用它们。

---
### 禁用容器中的内核功能
requiredDropCapabilities。在这个字段中列出的内核功能会在所有容器中被禁用（PodSecurityPolicy访问控制插件会在所有容器的securityContext.capbilities.drop 字段中加入这些内核功能）。

---
### 限制pod可以使用的存储卷类型
PodSecurityPolicy 资源可以做到的是定义用户可以在 pod 中使用哪些类型的存储卷。在最低限度上， 一个 PodSecurityPolicy 需要允许 pod 使用以下类型的存储卷： emptyDir 、 configMap 、 secret 、 downwardAPI 、persistentVolumeClaim。

例如：  
```yaml
kind: PodSecurityPolicy
spec:
  volumes:
  - emptyDir
  - configMap
  - secret
  - downwardAPI
  - persistentVolumeClaim
```

如果有多个 PodSecurityPolicy 资源， pod 可以使用 PodSecurityPolicy 中允许使
用的任何一个存储卷类型（实际生效的是所有 volumes 列表的并集）。

---
### 13.3.5 对不同用户与组分配不同的PodSecurityPolicy
PodSecurityPolicy 是集群级别的资源。

对不同用户分配不同 PodSecurityPolicy是通过前一章中描述的 阻AC 机制实现的。这个方法是， 创建你需要的 PodSecurityPolicy 资源，然后创建 ClusterRole 资源并通过名称将它们指向不同的策略，以此便 PodSecurityPoIicy 资源中的策略对不同的用户或组生效。通过 Clu sterRo leBinding 资源将特定的用户或组绑定到 ClusterRole上，当 PodSecurityPolicy 访问控制插件需要决定是否接纳一个 pod 时，它只会考虑、创建 pod 的用户可以访问到的 PodSecurityPolicy 中的策略。

---
### 创建一个允许部署特权容器的 PodSecurityPolicy
```yaml
apiVersion: v1
kind: PodSecurityPolicy
metadata:
  name: privillege
spec:
  privilleged: true
  runAsUser:          
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:             
    rule: RunAsAny
  volumes:             
  - '*'
  ```

---
### 使用 RBAC 将不同的 PodSecurityPolicy 分配给不同用户

目前有两个psp，一个名为default不允许部署特权容器。一个名为privillege允许部署特权容器。

首先需要创建两个 ClusterRole ，分别允许使用其中一个策略。将第一个ClusterRole 命名为 psp-default 并允许其使用 default PodSecurityPolicy 资源。可以使用`kl create clusterrole `命令来操作：  
`kl create clusterrole psp-default --verb=use --resource=podsecuritypolicies --resource-name=default`

创建了一个名为psp-default的 clusterrole，指向default策略。

接下来再创建一个名为psp-privillege的clusterrole，指向privillege策略。

然后使用clusterrolebinding（而非rolebinding）来将clusterrole绑定到特定上。


要将psp-default ClusterRole绑定到所有已认证用户上，而非只有一个用户。这是必须的，否则没有用户可以创建pod。因为PodSecurityPolicy访问控制插件会因为没有找到任何策略而拒绝创建pod。所有已认证用户都属于system:authenticated组，因此你需要将该ClusterRole绑定到这个组：  
`kl craete clusterrolebinding psp-all-users --clusterrole=psp-default --group=system:authenticated`

然后需要将psp-privillege绑定到用户beta
`kl create clusterrolebinding psp-zch --clusterrole=psp-privillege --user=haochen`

haochen可以创建特权pod，而其他认证用户不可以。接下来看看事实还是否如此。

---
### 用kubectl 创建不同用户
首先，你需要使用如下命令，用 kubectl 的 config 子命令创建两个新用户：  
`kl config set-credentials alpha --username=alpha --password=password`  
`kl config set-credentials beta --username=beta --password=password`

---
### 使用不同的用户创建pod
可以尝试以alpha的身份认证，并创建一个特权pod。可以通过 --user选项向kubectl 传达你使用的用户凭据：  
`kl --user alpha create -f pod-privillege.yaml`  

与预期的一样，API服务器不允许alpha创建特权pod。

`kl --user beta create -f pod-privillege.yaml`
API服务器允许beta创建特权pod。

这里成功使用了RBAC，让访问控制插件对不同的用户使用不同的资源。

---
### 13.4 隔离pod的网络
我们来看一下如何通过限制 pod 可以与其他哪些 pod 通信，来确保 pod 之间的网络安全。

是否可以进行这些配置取决于集群中使用的网络容器插件。如果网络插件支持，可以通过 NetworkPolicy 资源配置网络隔离。

一个 NetworkPolicy 会应用在匹配它的标签选择器的 pod 上，指明这些允许访问这些 pod 的源地址，或这些 pod 可以访问的目标地址。这些分别由入向（ingress)和出向（egress ）规则指定。

这两种规则都可以匹配由标签选择器选出的 pod ，或者一个 namespace 中的所有 pod ，或者通过无类别域间路由（Classless Inter-Domain Routing,  CIDR）指定的 IP 地址段。

---
### 13.4.1 
在默认情况下， 某一命名空间中的 pod 可以被任意来源访问 。首先， 需要改变
这个设定。需要创建一个 default-deny NetworkPoli cy ，它会阻止任何客户端访
问中的 pod。

```yaml
apiVersion: v1
kind: NetWorkPolicy
metadata:
  name: default-deny
spec:
  podSelector:    #空的标签选择器匹配命名空间中的所以pod
```

在任何一个特定的命名空间中创建该NetworkPolicy之后，任何客户端都不能访问该命名空间中的pod

> 注意： 集群中的 CNI 插件或其他网络方案需要支持 NetworkPolicy ，否则
NetworkPolicy 将不会影响 pod 之间的可达性。

---
### 13.4.2 允许同一命名空间中的部分 pod 访问一个服务端 pod
仅允许部分pod访问其他特定pod的特定端口的NetWorkPolicy  
```yaml
apiVersion: v1
kind: NetworkPolicy
metadata:
  name: db-net-policy
spec:
  podSelector:      #这个networkpolicy确保了对具有app=databae标签的pod的访问安全性
    matchLabels:
      app: database
  ingress:          #它只允许来自具有app=webserv标签的pod的访问
  - from:
    - podSelector:
        matchLabels:
          app: webserv
  ports:
  - port: 5432     #允许对这个端口访问
```

NetworkPolicy 允许具有 app=webserv 标签的 pod 访问具有 app=database 的pod的访问，并且仅限访问5432端口。

>客户端通常通过Service 而非直接访问pod来访问服务端pod，但这对结果没有改变。NetworkPolicy在通过Service访问时仍然会被执行。

---
### 13.4.3 在不同k8s命名空间之间进行网络隔离
现在我们来看有另一个多个租户使用同一 Kubernetes集群的例子。每个租户有多个命名空间，每个命名空间中有一个标签指明它们属于哪个租户。例如，有一个租户haochen，它所在的命名空间中都有标签tenant:haochen。其中的一个命名空间中运行了一个微服务 `Shopping Cart`（购物车），它需要允许同一租户下所有命名空间的所有pod访问。显然，其他租户禁止访问这个微服务。

为了保障该微服务安全，可以创建如下的NetworkPolicy资源。  
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shoppingcart-netpolicy
spec:
  podSelector:
    matchLabels:
      app: shopping-cart    #该NetworkPolicy策略用于具有app=shopping-cart标签的pod
  ingress:
  - from: 
    - namespaceSelector:    #只有在具有tenant=haochen标签的命名空间中运行的pod可以访问该微服务
        matchLabels:
          tenant: haochen
  ports:
  - port: 9190
```

以上NetworkPolicy保证了只有在具有tenant=haochen的命名空间中运行的pod才可以访问该shopping-cart微服务。

如果 shopping-cart服务的提供者需要允许其他租户访问该服务，则可以创建一个新的NetworkPolicy资源，或者在之前的NetworkPolicy中加入一条入向的规则。

> 注意在多租户的k8s集群中，通常租户不能为它们的命名空间添加标签（或注释）。否则，它们可以规避基于namespaceSelector的入向规则。

---
### 13.4.4 使用 CIDR 隔离网络
除了通过在pod选择器或命名空间选择器定义哪些pod可以访问NetworkPolicy
资源中指定的目标pod, 还可以通过CIDR表示法指定一个IP段。

例如，为了允许IP在192.168.11到192.168.1.255范围内的客户端访问之前提到的shopping­-cart的pod, 可以在入向规则中加入如以下代码  
```yaml
ingress:
- from:
  - ipBlock:
      cidr: 129.168.1.0/24
```

### 13.4.5 限制 pod 的对外访问流量
例如：  
```yaml
spec:
  podSelector:
    matchLabels:
      app: webserver
  egress:
  - to:
      - podSelector:
          matchLabels: 
            app: database
```

以上的NetworkPolicy仅允许具有标签app=webserver的pod访问具有标签app=database的pod，除此之外不能访问任何地址（不论是其他pod，还是任何其他的IP，无论在集群内部还是集群外部）

---