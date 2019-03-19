Table of Contents
=================
<!-- TOC -->

- [目录](#目录)
    - [基于二进制部署kubernetes v1.13.4](#基于二进制部署kubernetes-v1134)
        - [1. kubernetes概述](#1-kubernetes概述)
            - [1.1 什么是kubernetes](#11-什么是kubernetes)
            - [1.2 kubernetes的特点](#12-kubernetes的特点)
            - [1.3 kubernetes能做什么](#13-kubernetes能做什么)
            - [1.4 kubernetes核心组件](#14-kubernetes核心组件)
            - [1.5 kubernetes二进制安装包下载](#15-kubernetes二进制安装包下载)
        - [2. kubernetes基础环境部署](#2-kubernetes基础环境部署)
            - [2.1 kubernetes部署节点环境说明](#21-kubernetes部署节点环境说明)
            - [2.2 kubernetes部署前准备](#22-kubernetes部署前准备)
            - [2.3 kubernetes节点系统优化](#23-kubernetes节点系统优化)
            - [2.4 安装Docker-CE](#24-安装docker-ce)
            - [2.5 配置Docker镜像加速](#25-配置docker镜像加速)
        - [3. 创建 CA 证书和秘钥](#3-创建-ca-证书和秘钥)
            - [3.1 安装 cfssl 工具集](#31-安装-cfssl-工具集)
            - [3.2 创建根证书 (CA)](#32-创建根证书-ca)
            - [3.3 生成 CA 证书和私钥](#33-生成-ca-证书和私钥)
            - [3.4 分发证书文件](#34-分发证书文件)
        - [4. 部署 kubectl 客户端工具](#4-部署-kubectl-客户端工具)
            - [4.1 下载和分发 kubectl 二进制文件](#41-下载和分发-kubectl-二进制文件)
            - [4.2 创建 admin 证书和私钥](#42-创建-admin-证书和私钥)
            - [4.3 创建 kubeconfig 文件](#43-创建-kubeconfig-文件)
            - [4.4 分发 kubeconfig 文件](#44-分发-kubeconfig-文件)

<!-- /TOC -->




# 目录
## 基于二进制部署kubernetes v1.13.4 

### 1. kubernetes概述

#### 1.1 什么是kubernetes

> Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。通过

**Kubernetes你可以：**
- 快速部署应用
- 快速扩展应用
- 无缝对接新的应用功能
- 节省资源，优化硬件资源的使用

> 注：K8s是将Kubernetes的8个字母“ubernete”替换为“8”的缩写。

#### 1.2 kubernetes的特点

- 可移植: 支持公有云，私有云，混合云，多重云（multi-cloud）
- 可扩展: 模块化, 插件化, 可挂载, 可组合
- 自动化: 自动部署，自动重启，自动复制，自动伸缩/扩展

#### 1.3 kubernetes能做什么

- 多个进程（作为容器运行）协同工作。（Pod）
- 存储系统挂载
- 分发敏感文件
- 应用健康检测
- 应用实例的复制
- Pod自动伸缩/扩展
- 务发现
- 负载均衡
- 滚动更新
- 资源监控
- 日志访问
- 调试应用程序
- 提供认证和授权

#### 1.4 kubernetes核心组件

**Kubernetes主要由以下几个核心组件组成:**
- etcd：保存了整个集群的状态；
- apiserver：提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- controller manager：负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler：负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- kubelet：负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
- Container runtime：负责镜像管理以及Pod和容器的真正运行（CRI）；
- kube-proxy：负责为Service提供cluster内部的服务发现和负载均衡；

**除了核心组件，还有一些推荐的Add-ons：**

- kube-dns：负责为整个集群提供DNS服务
- Ingress Controller：为服务提供外网入口
- Heapster：提供资源监控
- Dashboard：提供GUI
- Federation：集群联邦提供跨可用区的集群
- Fluentd-elasticsearch：提供集群日志采集、存储与查询


#### 1.5 kubernetes二进制安装包下载

安装包下载地址： <https://github.com/kubernetes/kubernetes>

需要下载的安装包如下图：

![](./images/install_tag.png)

### 2. kubernetes基础环境部署

#### 2.1 kubernetes部署节点环境说明

|            主机名             |   IP Address   |   service   |  
| :------------------: | :-----------------: | :---------------------------------: |  
|     ks-master    |    10.10.11.21    |   docker、etcd、api-server、scheduler、controller-manager、kubelet、flannel   | 
|      ks-node1       |   10.10.11.20    |  docker、etcd、kubelet、proxy、flannel    |
|  ks-node2 |  10.10.11.19|    docker、etcd、kubelet、proxy、flannel  |


**master节点**

> Master节点上面主要由四个模块组成，APIServer，schedule,controller-manager,etcd

> APIServer: APIServer负责对外提供RESTful的kubernetes API的服务，它是系统管理指令的统一接口，任何对资源的增删该查都要交给APIServer处理后再交给etcd，如图，kubectl>(kubernetes提供的客户端工具，该工具内部是对kubernetes API的调用）是直接和APIServer交互的。

>schedule: schedule负责调度Pod到合适的Node上，如果把scheduler看成一个黑匣子，那么它的输入是pod和由多个Node组成的列表，输出是Pod和一个Node的绑定。 kubernetes目前提>供了调度算法，同样也保留了接口。用户根据自己的需求定义自己的调度算法。

>controller manager: 如果APIServer做的是前台的工作的话，那么controller manager就是负责后台的。每一个资源都对应一个控制器。而control manager就是负责管理这些控制器的，比如我们通过APIServer创建了一个Pod，当这个Pod创建成功后，APIServer的任务就算完成了。

>etcd：etcd是一个高可用的键值存储系统，kubernetes使用它来存储各个资源的状态，从而实现了Restful的API。

**node节点**

> 每个Node节点主要由三个模板组成：kublet, kube-proxy

> kube-proxy: 该模块实现了kubernetes中的服务发现和反向代理功能。kube-proxy支持TCP和UDP连接转发，默认基Round Robin算法将客户端流量转发到与service对应的一组后端pod。服务发现方面，kube-proxy使用etcd的watch机制监控集群中service和endpoint对象数据的动态变化，并且维护一个service到endpoint的映射关系，从而保证了后端pod的IP变化不会对访问者造成影响，另外，kube-proxy还支持session affinity。

> kublet：kublet是Master在每个Node节点上面的agent，是Node节点上面最重要的模块，它负责维护和管理该Node上的所有容器，但是如果容器不是通过kubernetes创建的，它并不会管理。本质上，它负责使Pod的运行状态与期望的状态一致。

#### 2.2 kubernetes部署前准备

**配置所有节点主机名**

```
# 配置主机名
[root@ks-master ~]# hostnamectl set-hostname ks-master
[root@ks-node1 ~]# hostnamectl set-hostname ks-node1
[root@ks-node2 ~]# hostnamectl set-hostname ks-node2

# 配置hosts记录

cat <<EOF > /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.10.11.21  ks-master
10.10.11.20  ks-node1
10.10.11.19  ks-node2
EOF

# 配置免密钥登陆

[root@ks-master ~]# ssh-keygen    
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:fIPG7HsiNiDfQ3e0eITt4tB9mR6JcHNF0di90R1F9rc root@ks-master
The key's randomart image is:
+---[RSA 2048]----+
|             .oBX|
|              ooB|
|         o   .  =|
|       +o.* .  .o|
|       .SOo= + E |
|  . . oo=.B.*    |
|   o + +.+ o .   |
|    . * o.. .    |
|     . +.o       |
+----[SHA256]-----+

[root@ks-master ~]# ssh-copy-id ks-master
[root@ks-master ~]# ssh-copy-id ks-node1
[root@ks-master ~]# ssh-copy-id ks-node2

```

#### 2.3 kubernetes节点系统优化

```
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭Swap
swapoff -a 
sed -i 's/.*swap.*/#&/' /etc/fstab

# 禁用Selinux
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config  

# 报错请参考下面报错处理
modprobe br_netfilter   
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
sysctl -p /etc/sysctl.d/k8s.conf
ls /proc/sys/net/bridge

# 内核优化
echo "* soft nofile 204800" >> /etc/security/limits.conf
echo "* hard nofile 204800" >> /etc/security/limits.conf
echo "* soft nproc 204800"  >> /etc/security/limits.conf
echo "* hard nproc 204800"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf

```
> 注：kubernetes所有节点执行


#### 2.4 安装Docker-CE

```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum makecache fast
yum install -y --setopt=obsoletes=0 \
  docker-ce-18.06.1.ce-3.el7

systemctl start docker
systemctl enable docker
```
> 注：kubernetes所有节点执行

#### 2.5 配置Docker镜像加速

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://95d2qlt5.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 3. 创建 CA 证书和秘钥


> 为确保安全，kubernetes 系统各组件需要使用 x509 证书对通信进行加密和认证。

> CA (Certificate Authority) 是自签名的根证书，用来签名后续创建的其它证书。

#### 3.1 安装 cfssl 工具集

```
sudo mkdir -p /opt/k8s/cert && cd /opt/k8s
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
mv cfssl_linux-amd64 /opt/k8s/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
mv cfssljson_linux-amd64 /opt/k8s/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /opt/k8s/bin/cfssl-certinfo

chmod +x /opt/k8s/bin/*
export PATH=/opt/k8s/bin:$PATH
```
> 注：所有操作在ks-master上执行

#### 3.2 创建根证书 (CA)

> CA 证书是集群所有节点共享的，只需要创建一个 CA 证书，后续创建的所有证书都由它签名。

**创建配置文件**

> CA 配置文件用于配置根证书的使用场景 (profile) 和具体参数 (usage，过期时间、服务端认证、客户端认证、加密等)，后续在签名其它证书时需要指定特定场景。

```
cd /opt/k8s/work
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```
> 注：所有操作在ks-master上执行

- signing：表示该证书可用于签名其它证书，生成的 ca.pem 证书中 CA=TRUE；
- server auth：表示 client 可以用该该证书对 server 提供的证书进行验证；
- client auth：表示 server 可以用该该证书对 client 提供的证书进行验证；

**创建证书签名请求文件**

```
cd /opt/k8s/work
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "4Paradigm"
    }
  ]
}
EOF
```
> 注：所有操作在ks-master上执行

- CN：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)，浏览器使用该字段验证网站是否合法；
- O：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；
- kube-apiserver 将提取的 User、Group 作为 RBAC 授权的用户标识；


#### 3.3 生成 CA 证书和私钥

```
cd /opt/k8s/work
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls ca*
```
> 注：所有操作在ks-master上执行

#### 3.4 分发证书文件

将生成的 CA 证书、秘钥文件、配置文件拷贝到所有节点的 /etc/kubernetes/cert 目录下：

首先执行此脚本加载变量：[environment.sh](common/environment.sh)

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh # 导入 NODE_IPS 环境变量
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "mkdir -p /etc/kubernetes/cert"
    scp ca*.pem ca-config.json root@${node_ip}:/etc/kubernetes/cert
  done
```
> 注：所有操作在ks-master上执行

### 4. 部署 kubectl 客户端工具

> kubectl 是 kubernetes 集群的命令行管理工具，kubectl 默认从 ~/.kube/config 文件读取 kube-apiserver 地址、证书、用户名等信息，如果没有配置，执行 kubectl 命令时可能会出错：

```
[root@ks-master bin]# kubectl get pods
The connection to the server 127.0.0.1:8443 was refused - did you specify the right host or port?
```

#### 4.1 下载和分发 kubectl 二进制文件

```
cd /opt/k8s/work
wget https://dl.k8s.io/v1.13.4/kubernetes-client-linux-amd64.tar.gz
tar -xzvf kubernetes-client-linux-amd64.tar.gz


# 分发到所有使用 kubectl 的节点：

cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp kubernetes/client/bin/kubectl root@${node_ip}:/opt/k8s/bin/
    ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
  done
```


#### 4.2 创建 admin 证书和私钥

> kubectl 与 apiserver https 安全端口通信，apiserver 对提供的证书进行认证和授权。

> kubectl 作为集群的管理工具，需要被授予最高权限。这里创建具有最高权限的 admin 证书。

**创建证书签名请求：**

```
cd /opt/k8s/work
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "4Paradigm"
    }
  ]
}
EOF
```

- O 为 system:masters，kube-apiserver 收到该证书后将请求的 Group 设置为 system:masters；
- 预定义的 ClusterRoleBinding cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予所有 API的权限；
- 该证书只会被 kubectl 当做 client 证书使用，所以 hosts 字段为空；

**生成证书和私钥:**

```
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
  -ca-key=/opt/k8s/work/ca-key.pem \
  -config=/opt/k8s/work/ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin

ls admin*
```

#### 4.3 创建 kubeconfig 文件

> kubeconfig 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书；

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/k8s/work/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kubectl.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials admin \
  --client-certificate=/opt/k8s/work/admin.pem \
  --client-key=/opt/k8s/work/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=kubectl.kubeconfig

# 设置上下文参数
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=kubectl.kubeconfig
  
# 设置默认上下文
kubectl config use-context kubernetes --kubeconfig=kubectl.kubeconfig
```

- --certificate-authority：验证 kube-apiserver 证书的根证书；
- --client-certificate、--client-key：刚生成的 admin 证书和私钥，连接 kube-apiserver 时使用；
- --embed-certs=true：将 ca.pem 和 admin.pem 证书内容嵌入到生成的 kubectl.kubeconfig 文件中(不加时，写入的是证书文件路径)；

#### 4.4 分发 kubeconfig 文件

> 分发到所有使用 kubectl 命令的节点：

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "mkdir -p ~/.kube"
    scp kubectl.kubeconfig root@${node_ip}:~/.kube/config
  done
```

