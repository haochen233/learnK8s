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



