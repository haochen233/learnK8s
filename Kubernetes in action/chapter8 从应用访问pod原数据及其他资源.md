### 本章内容涵盖
- 通过 Downward API 传递信息到容器
- 探索Kubernetes REST API
- 把授权和服务器验证交给kubectl proxy
- 从容器内部访问API服务器
- 理解 ambassador 容器模式
- 使用 Kubernetes 客户端库

---
## 8.1 通过Downward API传递元数据

>我们已经了解到如何通过环境变量或者configMap和secret卷向应用传递配置数据。 这对于pod调度、 运行前预设的数据是可行的。
但是对于那些不能预先知道的数据， 比如pod的IP、 主机名或者是pod自身的名称(当名称被生成，如RS或RC生成的名称)呢？此外，对
于那些已经在别处定义的数据， 比如pod的标签和注解呢？


### 8.1.1 了解可用的元数据
Downward API 可以给在pod中运行的进程暴露pod的元数据。 目前我们可以给容器传递以下数据：  
- pod的名称
- pod的IP
- pod 所在的命名空间
- pod运行节点的名称
- pod运行所归属的服务账户的名称（未知）
- 每个容器请求的CPU和内存使用量
- 每个容器可以使用的CPU和内存的限制
- pod的标签
- pod的注解

除了还没有讲到的服务账户、CPU和内存的请求和限制概念，其他都无须进一步解释。

先了解服务账户是pod访问API服务器时用来进行身份验证的账户。

---
列表中的大部分项目既可以通过环境变量也可以通过downwardAPI卷传递给容器，但是标签和注解只可以通过卷暴露。部分数据可以通过其他方式获取，但是DownwardAPI提供了一种更加便捷的方式。

### 8.1.2 通过环境变量暴露元数据
通过环境变量的方式将pod和容器的元数据传递到容器中。
例如：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dw
spec:
  containers:
  - name: c1
    image: 192.168.2.7:5000/go_serv:1.0
    stdin: true
    tty: true
    resources:
      requests:    #资源请求
        cpu: 15m
        memory: 3000Ki
      limits:      #资源限制
        cpu: 100m
        memory: 4Mi
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
         fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name:  SERVICE_ACCOUNT
      valueFrom: 
        fieldRef: 
          fieldPath:  spec.serviceAccountName
    - name:  CONTAINER_CPU  REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef: 
          resource:  requests.cpu 
          divisor:  lm #基数单位，千分之一核 
    - name:  CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef: 
          resource:  limits.memory
          divisor:  lKi #基数单位为1Ki，则该环境变量显示的值就位4Mi/1Ki = 4096。
      
```
当我们的进程在运行时， 它可以获取所有我们在pod的定义文件中设定的环境变量。

### 8.1.3 通过downwardAPI 卷来传递元数据
如果更倾向于使用文件的方式而不是环境变量的方式暴露元数据。可以定义一个downward卷并挂载到容器中。由于不能通过环境变量暴露，所以必须使用downwardAPI卷来暴露pod标签或注解。

与环境变量一样，需要显示地指定元数据字段来暴露给进程


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dw
  labels:
    type: serv
  annotations:
    key1: v1
    key2: |
      v2
      v3
      v4
spec: 
  containers:
  - image: go_serv:1.0
    name: c1
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    volumeMounts:
    - name: dwvol
      mountPath: /mnt/downward
  volumes:
  - name: dwvol
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          filedPath:  metadata.name
      - path: "labels"
        fieldRef:
          filedPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "cpu-req-millicores"
        resourceFieldRef:
          containerName: c1
          resource: requests.cpu
          divisor: 1m
      - path: "Mem-limit"
        resourceFieldRef:
          containerName: c1
          resource: limits.memory
          divisor: 1
```

我们没有通过环境变量传递元数据而是定义了一个叫作 downward 的
卷， 并且通过 /etc/downward目录挂载到我们的容器中。 卷所包含的文件会通过卷定义中的 downwardAPI.iterms属性来定义。

### 修改标签和注解
可以在pod运行时修改标签和注解。如我们所愿，当标签和注解被修改后，Kubernetes会更新存有相关信息的文件，从而使pod可以获取最新的数据。这也解释了为什么不能通过环境变量的方式暴露标签和注解，在环境变量方式下，一旦标签和注解被修改，新的值将无法暴露。

---
### 在卷的定义中引用容器级的元数据
当暴露容器级的元数据时，如容器可使用的资源限制或者资源请求（containers.resources.requests或limits），使用`volumes.downwardAPI.items.resourceFieldRef`字段，一定要指定容器名称：`resourceFieldRef.containerName`字段。

使用卷的方式来暴露容器的资源请求和使用限制比环境变量的方式稍显复杂，但是好处是，可以传递一个容器的资源到另一个容器（处于同一个pod）。

---
### 何时使用 Dowanward API 方式
不过通过Downward API 的方式获取的元数据是相当有限的， 如果需要获取更多的元数据， 需要使用直接访问 Kubernetes API服务器的方式。

---
## 8.2 与Kubernetes API服务器交互

---
### 8.2.1 探究Kubernetes REST API
先尝试直接访问 API服务器， 可以通过运行`kubectk cluster-info`命令来得到服务器的URl

因为服务器使用 HTTPS 协议并且需要授权， 所以与服务器交互并不是一件简单的事情。

幸运的是，我们可以执行kubectl proxy命令，通过代理与服务器交互，而不是自行来处理验证过程。

---
### 通过通过 Kubectl proxy 访问 API 服务器
`kubectl proxy`命令启动一个代理服务来接收来自你本机的HTTP 连接并转发至 API服务器，时处理身份认证， 所以不需要每次请求都上传认证凭证。它也可以确保我们直接与真实的API服务器交互， 而不是一个中间入（通过每次验证服务器证书的方式）。

运行代理：  
`kubectl proxy`
输出了

    Starting to serve on 127.0.0.1:8001

我们无需传递其他任何参数，因为kubec旦已经知晓所需的所有参数(API服务器URL、认证凭证等）。 一旦启动， 代理服务器就将在本地端口8001接收连接请求。

>`注意使用：后台进程运行kubectl proxy`,否则会一直卡在starting to serve on 127.0.0.1:8001

### 通过kubectl proxy研究 Kubernetes API
API服务器会返回一些路径：这些路径对应了我们创建Pod、Service这些资源时定义的API组和版本信息

>注意 这些没有列入API组的初始资原类型现在一般被认为归属于核心的
API组。

### 研究批量API组的REST endpoint
来研究下Job资源API，从路径内容/apis/batch下的内容开始

例如：  
`curl http://localhost:8001/apis/batch/v1`

API服务器返回了在batch/Vl目录下API 组中的资源
类型以及 RESTendpoint清单。除了资源的名称和相关的类型，API 服务器也包含了一些其他信息，比如资源是否被指定了命名空间、名称简写。资源对应可以使用的动词列表等。

### 列举集群中所有的Job实例
通过在/apis/batch/v1/jobs 路径运行一个GET请求，可以获取集群中所有Job清单。

`curl http://localhost:8001/apis/batch/v1/jobs`

如果集群中没有部署Job资源那么 items 数组将是空的。

### 从pod内部与API服务器进行交互
我们已经知道如何从本机通过使用 kubectl proxy与 API 服务器进行交互。现在我们来研究从一个 pod 内部访问它，这种情况下通常没有kubec口可用。因此，想要从 pod 内部与 API 服务器进行交互， 需要关注以下三件事情：  
- 确定API服务器的位置
- 确保是与API服务器进行交互，而不是冒名者
- 通过服务器的认证，否则将不能查看任何内容以及进行任何操作

### 运行一个pod来尝试与API服务器进行通信
运行一个能够执行curl命令的容器

---
### 发现API服务器地址

首先，需要找到Kbemetes API服务器的 IP 地址和端口。 这一点比较容易做到，
因为 一个名为kubernetes的服务在默认的命名空间被自动暴露。

使用命令:`kubectl get svc`会发现一个名为Kubernetes的服务。可以看到这个服务的IP和端口。

---
在容器内通过查询KUBERNETES_SERVICE_HOST和KUBERNETES_SERVICE_PORT这两个环境变量就可以获取 API服务器的IP地址和端口。

知道了HTTPS的443默认端口，就可以通过HTTPS协议来访问服务器：  
`curl https://kubernetes`,但是会有证书校验，简单的做法可以使用-k或-insecure选项来跳过这个验证环节。

---
但是我们应该使用更长（正确）的途径。我们应该通过使用 curl 检查证书的方式验证 API服务器的身份，而不是盲目地相信连接的服务是可信的。

>提示在真实的应用中， 永远不要跳过栓查服务器证书的环节。 这样做会导致你的应用验证凭证暴露给采用中间人攻击方式的攻击者。

---
### 验证服务器身份
我们知道
名为 defalut-token-8wxdr的Secret会被自动创建并挂载到每个容器的/var/run/secrets/kubernetes.io/serviceaccount
目录下。 让我们查看目录下的文件， 再次看一下Secret 的内容。

关注下ca.crt文件，该文件中包含了CA的证书。

用来对 Kubernetes API服务器证书进行签名为了验证正在交互的API服务器，我们需要检查服务器的证书是否是由 CA签发。curl允许使用-cacert选项来指定CA 证书， 我们来尝试重新访问 API 服务器：
`curl --cacert ca.crt https//kubernetes`

未完。


