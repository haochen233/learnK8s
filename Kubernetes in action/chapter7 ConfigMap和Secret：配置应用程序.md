### 本章内容涵盖
- 更改容器的主进程
- 将命令行选项传递给应用程序
- 设置暴露给应用程序的环境变量
- 通过**ConfigMap**配置应用程序
- 通过**Secret**传递敏感配置信息

---

## 7.1 配置容器化应用程序
开发一款新应用程序的初期， 除了将配置嵌入应用本身， 通常会以命令行参数的形式配置应用。 随着配置选项数量的逐渐增多， 将配置文件化。

另一种通用的传递配置选项给容器化应用程序的方法是借助环境变量。应用程序主动查找某一特定环境变量的值， 而非读取配置文件或者解析命令行参数。

为何环境变量的方案会在容器环境下如此常见？常直接在Docker容器中采用配置文件的方式是有些许困难的， 往往需要将配置文件打入容器镜像， 抑或是挂载包含该文件的卷。显然每次修改完
配置之后需要重新构建镜像。除此之外，任何拥有镜像访问权限的人可以看到配置文件中包含的敏感信息，如证书和密钥。相比之下，挂载卷的方式更好，然而在容器启动之前需确保配置文件已写入相应的卷中。

---

你可能会想到采用gitRepo卷作为配置源。 这并不是一个坏主意， 通过它可以保持配置的版本化，并且能比较容易地按需回滚配置。然而有一种更加简便的方法能将配置数据置于Kubernetes的顶级资源对象中，并可与其他资源定义存入同一 Git仓库或者基于文件的存储系统中。

无论你是否在使用ConfigMap 存储配置数据， 以下方法均可被用作配置你的应用程序：  
- 向容器传递命令行参数
- 为每个容器设置自定义环境变量
- 通过特殊类型的卷将配置文件挂载到容器中

## 7.2 向容器传递命令行参数

### 7.2.1 在Docker中定义命令与参数
首先，需要阐明的是，容器中运行的完整指令有两部分组成：命令和参数。

### 了解ENTRYPOINT与CMD
Dockerfile中的两种指令分别定义命令与参数这两个部分：  
- ENTRYPOINT 定义容器启动时被调用的可执行程序
- CMD 指定传递给ENTRYPOINT的参数

        借助ENTRYPOINT指令， 仅仅用CMD指定所需的默认参数。 这样， 镜像可以直接运行， 无须添加任何参数

记住docker run时指定的参数会覆盖CMD指定的参数。

所以有了ENTRYPOINT和CMD可以用两种方式启动容器：  
1. `docker run <image>`不添加参数
2. `docker run <image> args`指定参数
---

### 了解shell与exec形式的区别
上述两条指令均支持以下两种形式：  
- shell形式——如`ENTRYPOINT node app.js`
- exec形式——如ENTRYPOINT ["node", "app.js"]。

两者的区别是否是在shell中被调用。一个是直接运行node进程，另一个是从shell中启动，所以这个从shell中启动优点多余，推荐数组格式的ENTRYPOINT。

### 7.2.2 在Kubernetes中覆盖命令和参数

在Kubernetes中定义容器时，镜像的ENTRYPOINT和CMD均可以被覆盖，只需要在容器定义中设置属性（字段）command和args的值。

例如，自定义command和args的pod：  
```yaml
kind: Pod
spec: 
  containers:
  - image: 192.168.2.7:5000/go_serv
    command: ["/bin/command"]
    args: ["arg1", "arg2", "arg3"]
```

> 注意：command和args字段在pod创建后无法被修改。

少量参数值的设置可以使用上述的数组表示。多参数值情况下可以采用如下标记：  
```yaml
args:
- foo
- bar
- "15"
```
> 提示，字符串值无需用引号标记，数组需要。

---

## 7.3为容器设置环境变量

容器应用化通常会使用环境变量作为配置源。Kubernetes允许为pod中的每一个容器都指定自定义的环境变量合集。

> 注意与容器的命令和参数设置相同，环境变量列表无法在pod创建后被修改。

### 7.3.1 在容器定义中指定环境变量
```yaml
kind: Pod
spec:
  containers:
  - image: 192.168.2.7:5000/go_serv:1.0
    env:
    - name: readme
      value: "hello alpine linux"
```

环境变量被设置在pod的容器定义中，并非是pod级别。

### 7.3.2 在环境变量值中引用其他环境变量
类似linux中使用$env_name 显示env_value一样。

```yaml
kind: Pod
spec:
  containers:
  ...
    env:
    - name: HELLO
      value: helllo
    - name: WORLD
      value: $(HELLO)world
```

### 7.3.3 了解硬编码环境变量的不足之处
为了能在多个环境下复用pod的定义，需要将配置从pod的定义描述中解耦出来。幸运的是，你可以通过一种叫作ConfigMap的资源对象完成解耦，用**valueFrom**字段替代**value**字段使ConfigMap成为环境变量值的来源。

## 7.4 利用ConfigMap解耦配置

如果将 pod 定义描述看作是应用程序源代码，显然需要将配置移出 pod 定义。

### 7.4.1 ConfigMap介绍
Kubernetes 允许将配置选项分离到单独的资源对象 ConfigMap 中，本质上就是一个键/值对映射，值可以是短字面量，也可以是完整的配置文件。

应用无须直接读取ConfigMap,甚至根本不需要知道其是否存在。映射的内容通过环境变量或者卷文件的形式传递给容器。而并非直接传递给容器。命令行参数的定义中可以通过`$(ENV_VAR)`语法引用环境变量，因而可以达到将ConfigMap的条目当做命令行参数传递给进程的效果。


>总的来说，pod通过环境变量与ConfigMap卷使用ConfigMap。

当然，应用程序同样可以通过 Kubernetes Rest API按需直接读取ConfigMap的内容。不过除非是需求如此， 应尽可能使你的应用保持对 Kubernetes 的无感知。

pod是通过名称引用 ConfigMap 的，因此可以在多环境下使用相同的 pod 定义描述。  

### 7.4.2 创建ConfigMap
用指令`kubectl create configmap` 创建 ConfigMap。

### 使用指令kubectl 创建ConfigMap

例如,如下manifest：  
```yaml
kubectl create configmap myconfigmap --from-literal=START='19:00:00' --from-literal=END='22:00:00'
```
通过上面的命令创建一个名为myconfigmap的ConfigMap，有两条映射（START=19:00:00;END=22:00:00），使用多次--from-literal 参数可以创建多条映射。

---

然后，可以使用`kubectl get configmap myconfigmap -o yaml | less`查看我们创建的configmap的yaml格式的定义。

当然编写这个yaml文件也是很容易的。

### 从文件内容创建ConfigMap条目

命令为：`kubectl create configmap my-configmap --from-file=config-file.conf`

运行上述命令时，kubectl会在当前目录下查找 config-file.conf文件,并将文件内容存储在 ConfigMap中义config-file.conf为键名的条目下。

>注意：使用查看configMap时，可能会看到有的条目第一行最后的管道符，它表示后续的条目值是多行字面量。

同样可以多次使用--from-file选项来增加多个条目

### 从文件夹创建ConfigMap
可以引入某一文件夹中的所有文件：  
`kubectl create configmap my-config --from-file=/path/dir/`  
会为文件夹中的每个文件单独创建条目，仅限于文件名可合法作为ConfigMap键名的文件。


### 合并不同选项
创建 ConfigMap 时可以混合使用这里提到的所有选项：  
> kubectl create configmap my-cf
>- `-from-file=data.json #单独的文件，默认使用文件名作为键名`
>- `--from-file=second_name=zch.conf #键名自定义为second_name`
>- `--from-file=dir/ # 完整的文件夹`
>- `--from-literal=FIELD1='haha' # 字面量`


### 7.4.3 给容器传递 ConfigMap 条目作为环境变量

即将`pod.spec.containers.env.value`字段替换为valueFrom字段，例如：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cf
spec:
  containers:
  - image: 192.168.2.7:5000/go_serv:1.0
    name: test
    env:
    - name: ZCH_ENV
      valueFrom:
        configMapKeyRef:
          name: mycmp
          key: START-TIME
```
在这定义了一个环境变量名为ZCH_ENV，值引用名为mycmp的ConfigMap中的START-TIME键对应的值。


---

### 在 pod 中引用不存在的 ConfigMap
Kubemetes 会正常调度 pod 并尝试运行所有的容器。 然而**引用不存在的 ConfigMap**
的容器会启动失败，其余容器能正常启动。但是，如果之后创建这个缺失的ConfigMap，失败的容器会自动启动，无须重新创建pod。

> 注意：可以标记对 ConfigMap 的引用是可选的（设直 configMapKeyRef.optional: true)。设置为对ConfigMap的引用是可选的，即使ConfigMap不存在，也能够正常启动。  

这个例子展示了如何将配置从 pod 定义中分离。这样能使所有的配置项较为集
中（甚至多个 pod 也是如此〉，而不是分散在各处（或者冗余复制于多个 pod 定义
清单）

简单地说将配置集中了起来。

---
### 7.4.4 一次性传递 ConfigMap 的所有条目作为环境变量

如果ConfigMap包含不少条目，为每个条目单独设置环境变量的过程是单调乏味且容易出错的。幸运的是， 1.6版本的 Kubernetes 提供了暴露 ConfigMap 的所有条目作为环境变量的手段。

如果一个ConfigMap包含多个条目（键值对）。可以通过`spec.containers.envFrom`字段将所有条目暴露作为环境变量。

例如：  
```yaml
spec:
  containers:
  - image: go_serv:1.0
    envFrom:
    - prefix: CONFIG_ #所有环境变量均包含前缀CONFIG_
      configMapRef:
        name: my-config-map
```

---
### 7.4.5 传递ConfigMap条目作为命令行参数
如何将ConfigMap中的值作为参数值传递给运行在容器中的主进程，在字段`pod.spec.containers.args`中无法直接引用ConfigMap的条目。但是，可以先用ConfigMap初始化某个环境变量；然后再在参数字段中引用该环境变量。

例如下面的manifest：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - image: go_serv:1.0
    name: cnt
    env:
    - name: ARG1
      valueFrom:
        configMapKeyRef:
        - name: my-cmp
          key: ARG1
    args: ["$(ARG1)"]
```

引用my-cmp的key为ARG的条目的值作为环境变量ARG1的值，然后参数引用环境变量ARG1的值。  

这样就间接引用ConfigMap的条目作为命令行的参数。

---

### 7.4.6 使用configMap卷将条目暴露为文件

环境变量或者命令行参数值作为配置值通常适用于变量值较短的场景。

由于ConfigMap 中可以包含完整的配置文件内容，当你想要将其暴露给容器时，可以借助configMap卷的特殊卷格式。

尽管这种方法主要适用于传递较大的配置问价给容器，同样可以用于传递较短的变量值。
例如：  
```yaml
...
spec:
  volumes:
  - name: vol1
    configMap: 
    - name: cmp
  containers:
  - image: go_serv:1.0
    name: c1
    volumeMounts:
    - name: vol1
      mountPath: /mnt/config
      readOnly: true
```

---
### 创建ConfigMap
从文件创建一个configMap，将文件名作为条目名。  
`kubectl ceate configmap mycmp --from-file=cf.conf`
---

### 在卷内使用ConfigMap的条目

创建包含ConfigMap条目内容的卷只需要创建一个引用ConfigMap名称的卷并挂载到容器中。

例如：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  volumes:
  - name: config
    configMap:  # configMap类型的卷
      name: my-cmp #引用my-cmp的ConfigMap文件名字。
  containers:
  - image: go_serv
    name: cnt
    volumeMounts:
    - name: config #卷名
      mountPath: /mnt #卷挂载的容器路径
      ReadOnly: true
```

进入容器检查是否已经挂载configMap卷。
挂载目录会有所有条目的符号链接文件。
---

### 卷内暴露指定的ConfigMap条目

可以创建仅包含ConfigMp中部分条目的configMap卷。

通过指定pod.spec.volumes.congifMap.items字段来实现。
```yaml
spec:
  volumes:
  - name: vol1
    configMap: 
      name: cmp
      items:
      - key: E
        path: mypath/E.conf   #path作用是将E条目，重命名为E.conf，且加入了mypath路径
  containers:
  - image: go_serv:1.0
    name: c1
    volumeMounts:
    - name: vol1
      mountPath: /mnt/config
      readOnly: true

```
---

### 挂载某一文件夹会隐藏该文件加中已存在的文件。

例如挂载文件夹是/etc, 该文件夹通常包含不少重要文件。由于/etc 下的所有文件不存在，容器极大可能会损坏。如果你希望添加文件至某个文件夹如/etc, 绝不能采用这种方法。

---

### ConfigMap独立条目作为文件被挂载且不隐藏文件夹中的其他文件

volumeMounts额外的subPath字段可以被用作挂载卷中的某个独立文件或者是文件夹，无须挂载完整卷。

假设拥有一个包含文件myconfig.conf的confgMap卷，希望能将其添加为/etc文件夹下的文件someconfig.conf。通过属性subPath可以将该文件挂载的同时又不影响文件夹中的其他文件。

例如，由文件config.conf创建my-cmp的configmap文件；以my-cmp及其他创建的configMap卷名为cf-vol，然后里面有config.conf，只需要将config.conf添加到到/etc/my.conf文件

>pod挂载configMap（卷）的指定条目（文件）至特定文件。
```yaml
spec:
  containers:
    ...
    volumeMounts:
    - name: cf-vol
      mountPath: /etc/my.conf
      subPath: config.conf
```

即configMap卷将条目暴露为文件后，然后此时configMap有多个文件，只使用其中的一个，就用`spec.containers.volumeMounts.subPath`字段。

---

### 为configMap卷中文件权限设置

configMap卷中的所有文件默认权限是644，可以通过`volumes.configMap.defaultMode`字段来设置权限。

---

### 7.4.7 更新应用配置且不重启应用程序

使用环境变量或者命令行参数作为配置源的弊端在于无法在
进程运行时更新配置。将CnfigMap暴露为卷可以达到配置热更新的效果， 无须重
新创建pd或者重启容器。

ConfigMap被更新后，卷中引用它的所有文件也会相应更新，进程发现文件被更改之后进行重载，Kubernetes同样支持文件更新后手动通知容器。

>警告 更新ConfigMap之后对应文件的更新耗时会出人意料的长。
---
### 修改ConfigMap
尝试使用`kubectl edit configmap my-cmp`
然后就可以进行编辑，好后进行保存，等待文件更新。

---
### 了解文件自动更新的过程
被挂载的configMap卷中的文件是..data文件夹中文件的符号链接，而..data文件夹同样是符号链接，当ConfigMap更新后，Kubernetes会创建这样一个文件夹，写入所有文件并重新将..data符号链接至新文件夹，通过这种方式一次性修改了所有文件。

---
### 挂载已存在文件夹的文件不会被更新


>涉及到更新configMap卷需要提出一个警告：如果挂载的是容器中的单个文件而不是完整的卷， ConfigMap更新之后对应的文件不会被更新！

---
### 了解更新ConfigMap的影响
容器的一个比较重要的特性是其不变性，从同一镜像启动的多个容器之间不存在任何差异。


>关键点在于应用是否支持重载配置。ConfigMap更新之后创建的pod会使用新配置， 而之前的pod依旧使用旧配置， 这会导致运行中的不同实例的配置不同。这也不仅限于新pod, 如果pod中的容器因为某种原因重启了， 新进程同样会使用新配置。因此， 如果应用不支持主动重载配置， 那么修改某些运行pod所使用的ConfigMap并不是一个好主意。

如果应用支持主动重载配置， 那么修改ConfigMap的行为就算不了什么。有一点仍需注意， 由千 configMap 卷中文件的更新行为对于所有运行中示例而言不是同步的， 因此不同 pod中的文件可能会在长达一分钟的时间内出现不一致的清况。

---
## 7.5使用Secret给容器传递敏感数据

到目前为止传递给容器的所有信息都是比较常规的非敏感数据。

但是有的配置通常会包含一些敏感数据， 如证书和私钥， 需要确保其安全性。  

---
### 7.5.1 介绍Secret
为了存储与分发此类信息， Kubernetes提供了一种称为 Secret 的单独资源对象。 Secret 结构与 ConfigMap类似， 均是键／值对的映射。 Secret 的使用方法也与ConfigMap 相同，可以：  
- 将Secret条目作为环境变量传递给容器。
- 将Secret条目暴露为卷中的文件

Kubernetes 通过仅仅将Secret分发到需要访问Secret的 pod所在的机器节点来保障其安全性。

另外Secret是存储在节点的内存中的，永不写入物理存储。

>对千主节点本身（尤其是etcd), Secret通常以非加密形式存储， 这就需要保障
主节点的安全从而确保存储在Secret 中的敏感数据的安全性。 这种保障不仅仅是对
etcd存储的安全性保障， 同样包括防止未授权用户对API 服务器的访问， 这是因为
任何人都能通过创建 pod 并将Secret挂载来获得此类敏感数据。  

---

>从 Kubernetes 1.  7 
开始， etcd会以加密形式存储Secret, 某种程度提高了系统的安全性。

正因为如此，
从 Secret 与 ConfigMap中做出正确选择是势在必行的， 选择依据相对简单：  
- 采用ConfigMap存储非敏感的文本配置数据
- 采用Secret 存储天生敏感的数据，通过键来引用。如果一个配置文件同时包含敏感和非敏感数据，则该文件应该被存储在Secret中。

### 7.5.2 默认令牌 Secret 介绍

首先来分析一种默认被挂载至所有容器的 Secret,使用`kubectl describe po pod_name`来查看一个pod，往往可以看到以下的清单：  
```yaml
volumes: 
  default-token-8wxdr:
    type:     Secret (a volume populated by a Secret)
    secretName: dedefault-token-8wxdr
```

每个pod都会被自动挂载上一个secret卷，这个卷引用的是前面`kubectl describe`命令输出的叫作`dedefault-token-8wxdr`的Secret。

由于Secret也是资源对象，所以也可以用`kubectl get secrets`命令来查看Secret 列表。

同样可以使用kubectl describe多了解一下这个Secrets

---
可以看到Secret包含三个条目-ca.crt、 namespace与token,包含了从pod内部安全访问KubemetesAPI服务器所需的全部信息。

---
`kubectl  describe  pod`命令会显示secret卷被挂载的位置

> 意default-tokenSecret默认会被挂载至每个容器。 可以通过设置pod定
义中的automountServiceAccountToken字段为false, 或者设置pod使用的服务账户中的相同字段为false来关闭这种默认行为。

---
### 7.5.3 创建Secret
 通过命令`kubectl create secret`创建。


比如本目录下有两个文件http.key和http.cert，里面分别写点内容，然后可以这样生成Secret：  
`kubectl create secret generic mysct --from-file=http.key --from-file=http.cert`

还可以使用`kuebctl create secret tls secret_name`来生成。不过有很多选项与之前创建configmap和上面generic类型是不一样的。使用`kubectl create secret tls -h`来查看选项。

---
### 7.5.4 对比 ConfigMap 与 Secret

分别查看secret和configmap的yaml格式，可以发现secret条目的内容会以Base64格式编码，而configmap条目直接以纯文本显示。

---
这种区别导致在处理YAML和JSON格式的Sere
时会稍许有些麻烦，需要在设置和读取相关条目时对内容进行编解码。

### 为二进制数据创建Secret
采用Bae4编码的原因很简单 。Secret的条目可以涵盖二进制数据，而不仅仅是纯文本。Base64编码可以将二进制数据转换为纯文本 ，以YAML或JSON格式展示。


### string Data字段介绍

Kubernetes允许通过Secret的stringData字段设置条目的纯文本值.
例如：  
```yaml
kind: Secret
apiVersion: v1
stringData: 
  owner: plain text
data:
  https.cert: fafawfafawetqwrf...
  https.key: wrtgwea3rtgawetaw..

```

stringData字段是只写的，可以被用来设置条目值，所以通过`kubectl get secret sct_name -o yaml`获取secret的yaml格式时，不会显示stringData字段。相反，stringData字段中的所有条目被会Base64编码之后 展示在data字段下。

---
### 在pod中读取Secret条目
通过secret卷将secret暴露给容器之后，secret条目的值会被解码，以真实形式（纯文本或二进制）写入对应的文件。通过环境变量暴露Sere条目亦是如此。
在这两种情况下，应 用程序均无须主动解码，可直接读取文件内容或者查找环境变
量。

---
### 7.5.5 在pod中使用Secret
将含有证书和密钥的secret卷挂载至pod中的web_server容器中，例如：  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: serv
spec:
  volumes:    #挂载三个不同类型的卷
  - name: vol1
    emptyDir: {}
  - name: vol2
    configMap: 
      name: mycmp
  - name: vol3
    secret:
      secretName: sct
  containers:
  - image: go_serv:1.0
    name: web_serv
    volumeMounts:
    - name: vol1  
      mountPath: /mnt/storage  #把emptyDir挂载到/mnt/storage
    - name: vol2
      mountPath: /mnt/config  #把configmap挂载到/mnt/config目录
      readOnly: true
    - name: vol3
      mountPath: /mnt/secret   #把secret卷挂载到/mnt/secret
      readOnly: true
    stdin: true
    tty: true
```



### 通过环境变量暴露Secret条目
和configmap。类似

```yaml
env:
- name: ENV!
  valueFrom:
    secretKeyFrom:
      name: sct
      key: http.key
```
环境变量ENV1的值使用sct的http.key 条目

---
### 将所有secret条目暴露为环境变量
```yaml
...
envFrom:
- perfix: SECRET_   #所有环境变量名前加SECRET_
  secretMapRef:
  - name: sct
```