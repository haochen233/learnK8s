## 一、安装部分
首先准备2到3台centos虚拟机机器。  

可以用yum在线安装kubenetes、docker或者1离线安装。  

1. 离线安装：  

    1. 离线安装docker：  
    到国内开源软件镜像网站（如阿里云）下载docker-ce、docker-ce-cli、containerd.io这三个的rpm包。然后使用sudo yum localinstall package_name 命令安装这几个包。  

    2. 离线安装Kubernetes：  
    同样去阿里云等镜像网站下载。下载 kubectl、kubeadm、kubelet、Kubernetes-cni、cri-tools等五个rpm包。然后使用sudo yum localinstall package_name 命令安装这几个包。

2. 在线安装

    1. 在线安装kubernetes  
    需要在/etc/yum.repos.d/目录下新建Kubernetes.repo文件，配置该文件将Kubernetes仓库添加到yum的库。该文件的具体配置为：  
        ```repo
        [kubernetes]
        name=Kubernetes
         baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/       kubernetes-el7-x86_64/
         enabled=1
         gpgcheck=1
        repo_gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
        ```
        yum更新下缓存：`yum clean all ; yum -y makecache`  
    
    2. 安装docker
        。。。
    

----------------------
## 配置Kubernetes及docker所需的环境

1. 启动docker：  
    systemctl enable docker ; systemctl restart docker
      
    #设置docker开机启动，并且启动docker
2. 启动kubelet：  
    systemctl enable kubelet ; systemctl start kubelet  

    #设置开机启动kubelet，并且立即启动  
3. 关闭swap，不然会报错，临时关闭`swapoff -a`,建议永久关闭 注释掉/etc/fstab 文件中有swap的那一行。  

4. 关闭防火墙，查看防火墙状态`systemctl status firewalld`,禁止防火墙并且立即关闭`systemctl disable firewalld ; systemctl stop firewalld`

5. 修改内核参数net.bridge.bridge-nf-call-iptables
若为0则不对bridge数据处理，若为1，则处理。  
临时修改：`net.bridge.bridge-nf-call-iptables = 1`,`net.bridge.bridge-nf-call-ip6tables = 1`
永久修改:在 /etc/sysctl.d/目录下建立k8s.conf文件然后，在文件中输入  
`net.bridge.bridge-nf-call-iptables = 1`  
`net.bridge.bridge-nf-call-ip6tables = 1`  
然后保存。

6. 禁用selinux：  
    临时禁用： `setenforce 0`  
    永久禁用：修改/etc/selinux/config 文件将其中的SELINUX=enforcing ，改为SELINUX=permissive

6. 配置docker：  
    由于Dockerhub的服务器在国外，所以下载镜像特别慢，但是可以配置加速器，比如我们可以使用阿里云的Docker官方镜像加速器。配置链接如下：[阿里云镜像加速器](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)。在 /etc/目录下建立**docker**目录，然后在/etc/docker/ 目录下新建daemon.json文件，输入以下内容：  
    ```json
    {
        "registry-mirrors": ["https://v16stybc.mirror.aliyuncs.com"]
    }
    ```
    然后配置好文件后，需要重启docker，` systemctl daemon-reload ; systemctl restart docker`  

7. 修改Cgroup Driver，防止安装集群时报错，还是修改 /etc/docker/daemon.json 文件。  
新增`exec-opts": ["native.cgroupdriver=systemd`  
daemon.json 文件改成了如下：  
    ```json
    {
    "registry-mirrors": ["https://v16stybc.mirror.aliyuncs.com"],
    "exec-opts": ["native.cgroupdriver=systemd"]
    }
    ```
    然后再重新加载docker，` systemctl daemon-reload ; systemctl restart docker`



