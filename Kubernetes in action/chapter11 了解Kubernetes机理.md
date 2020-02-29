本章内容涵盖：  
- Kubernetes集群包含哪些组件
- 每个组件的作用以及他们是如何工作的
- 运行的pod是如何创建一个部署对象
- 运行的pod是什么
- pod至今的网络如何工作
- Kubernetes服务如何工作
- 如何保证高可用性


## 11.1 了解架构
Kubernetes集群分为两部分：  
- Kubernetes 控制平面（control plane）
- （工作）节点

---
控制平面的组件：  
- etcd 分布式持久化存储
- API服务器
- 调度器
- 控制器管理器

---
工作节点上运行的组件：  
- Kubelet
- kubelet代理服务（kube-proxy）
- 容器运行时

---
附加组件：  
- Kubernetes DNS 服务器
- 仪表盘
- Ingress 控制器
- Heapster（容器集群监控）
- 容器网络接口插件

---
### 11.1.1 Kubernetes组件的分布式特性

检查控制平面组件的状态，API服务器对外暴露了一个名为ComponentStatus的API资源（简称CS）。

查看控制平面**组件**的状态：  
`kubectl getc componentstatus`

---
### 组件间如何通信
Kubernetes 系统组件间只能通过API服务器通信，他们不会直接通信。