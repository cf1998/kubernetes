Table of Contents
=================
<!-- TOC -->

- [目录](#目录)
    - [基于 kubeadm 部署 kubernetes v1.14.0](#基于-kubeadm-部署-kubernetes-v1140)
        - [1. kubernetes 环境概述](#1-kubernetes-环境概述)
            - [1.1 kubernetes环境介绍](#11-kubernetes环境介绍)
            - [1.2 kubernetes核心组件](#12-kubernetes核心组件)
        - [2. kubernetes部署前准备](#2-kubernetes部署前准备)
            - [2.1 系统初始化](#21-系统初始化)
            - [2.2 设置国内 yum 源](#22-设置国内-yum-源)
            - [2.3 安装Docker-CE](#23-安装docker-ce)
            - [2.4 配置Docker镜像加速](#24-配置docker镜像加速)
        - [3. 部署ETCD高可用集群](#3-部署etcd高可用集群)
            - [3.1 安装 cfssl 工具集](#31-安装-cfssl-工具集)
            - [3.2 下载和分发 etcd 二进制文件](#32-下载和分发-etcd-二进制文件)
            - [3.3 创建 etcd 证书和私钥](#33-创建-etcd-证书和私钥)
            - [3.4 创建 etcd 的 systemd unit 模板文件](#34-创建-etcd-的-systemd-unit-模板文件)
            - [3.5 为各节点创建和分发 etcd systemd unit 文件](#35-为各节点创建和分发-etcd-systemd-unit-文件)
            - [3.6 启动 etcd 服务](#36-启动-etcd-服务)
            - [3.7 检查启动结果](#37-检查启动结果)
            - [3.8 验证服务状态](#38-验证服务状态)
            - [5.8 查看当前的 leader](#58-查看当前的-leader)
        - [4. 开始部署kubernetes](#4-开始部署kubernetes)
            - [4.1 安装kubeadm、kubectl、kubelet](#41-安装kubeadmkubectlkubelet)
            - [4.2 安装Master节点](#42-安装master节点)
            - [4.3 部署网络组件](#43-部署网络组件)
            - [4.3 加入节点](#43-加入节点)

<!-- /TOC -->

# 目录
## 基于 kubeadm 部署 kubernetes v1.14.0 

### 1. kubernetes 环境概述
#### 1.1 kubernetes环境介绍
> kubeadm是一个python写的项目，代码在这里，用来帮助快速部署Kubernetes集群环境，如果通过kubeadm默认安装，etcd则为单点，不建议用于生产环境，接下来我们自定义config.yaml让etcd与kubernetes集群分离，实现etcd高可用。

|            主机名             |   IP Address   |   service   |  
| :------------------: | :-----------------: | :---------------------------------: |  
|     ks-master    |    10.10.12.143    |   docker、etcd、api-server、scheduler、controller-manager、kubelet、flannel   | 
|      ks-node1       |   10.10.12.142    |  docker、etcd、kubelet、proxy、flannel    |
|  ks-node2 |  10.10.12.141|    docker、etcd、kubelet、proxy、flannel  |


#### 1.2 kubernetes核心组件

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




### 2. kubernetes部署前准备

#### 2.1 系统初始化

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
10.10.12.143  ks-master
10.10.12.142  ks-node1
10.10.12.141  ks-node2
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

**kubernetes节点系统优化**

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


#### 2.2 设置国内 yum 源

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

**安装必备工具**

```
$ yum install -y epel-release 
$ yum install -y net-tools wget vim  ntpdate
```

#### 2.3 安装Docker-CE

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

#### 2.4 配置Docker镜像加速

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

### 3. 部署ETCD高可用集群


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



etcd 是基于 Raft 的分布式 key-value 存储系统，由 CoreOS 开发，常用于服务发现、共享配置以及并发控制（如 leader 选举、分布式锁等）。kubernetes 使用 etcd 存储所有运行数据。

etcd 集群各节点的名称和 IP 如下：

- ks-master：10.10.12.143
- ks-node1：10.10.12.142
- ks-node2：10.10.12.141
> 注：所有操作在ks-master上执行

#### 3.2 下载和分发 etcd 二进制文件

```
cd /opt/k8s/work
wget https://github.com/coreos/etcd/releases/download/v3.3.10/etcd-v3.3.10-linux-amd64.tar.gz
tar -xvf etcd-v3.3.10-linux-amd64.tar.gz


# 分发二进制文件到集群所有节点

cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp etcd-v3.3.10-linux-amd64/etcd* root@${node_ip}:/opt/k8s/bin
    ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
  done

```

> 注：所有操作在ks-master上执行


#### 3.3 创建 etcd 证书和私钥

**创建证书签名请求：**

```
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.10.12.142",
    "10.10.12.143",
    "10.10.12.141"
  ],
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

- hosts 字段指定授权使用该证书的 etcd 节点 IP 或域名列表，这里将 etcd 集群的三个节点 IP 都列在其中；


**生成证书和私钥：**

```
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
    -ca-key=/opt/k8s/work/ca-key.pem \
    -config=/opt/k8s/work/ca-config.json \
    -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
ls etcd*pem
```

> 注：所有操作在ks-master上执行

**分发生成的证书和私钥到各 etcd 节点：**

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "mkdir -p /etc/etcd/cert"
    scp etcd*.pem root@${node_ip}:/etc/etcd/cert/
  done

```
> 注：所有操作在ks-master上执行

#### 3.4 创建 etcd 的 systemd unit 模板文件

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > etcd.service.template <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=${ETCD_DATA_DIR}
ExecStart=/opt/k8s/bin/etcd \\
  --data-dir=${ETCD_DATA_DIR} \\
  --wal-dir=${ETCD_WAL_DIR} \\
  --name=##NODE_NAME## \\
  --cert-file=/etc/etcd/cert/etcd.pem \\
  --key-file=/etc/etcd/cert/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-cert-file=/etc/etcd/cert/etcd.pem \\
  --peer-key-file=/etc/etcd/cert/etcd-key.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls=https://##NODE_IP##:2380 \\
  --initial-advertise-peer-urls=https://##NODE_IP##:2380 \\
  --listen-client-urls=https://##NODE_IP##:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://##NODE_IP##:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --auto-compaction-mode=periodic \\
  --auto-compaction-retention=1 \\
  --max-request-bytes=33554432 \\
  --quota-backend-bytes=6442450944 \\
  --heartbeat-interval=250 \\
  --election-timeout=2000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

> 注：所有操作在ks-master上执行

- WorkingDirectory、--data-dir：指定工作目录和数据目录为 ${ETCD_DATA_DIR}，需在启动服务前创建这个目录；
- --wal-dir：指定 wal 目录，为了提高性能，一般使用 SSD 或者和 --data-dir 不同的磁盘；
- --name：指定节点名称，当 --initial-cluster-state 值为 new 时，--name 的参数值必须位于 --initial-cluster 列表中；
- --cert-file、--key-file：etcd server 与 client 通信时使用的证书和私钥；
- --trusted-ca-file：签名 client 证书的 CA 证书，用于验证 client 证书；
- --peer-cert-file、--peer-key-file：etcd 与 peer 通信使用的证书和私钥；
- --peer-trusted-ca-file：签名 peer 证书的 CA 证书，用于验证 peer 证书；

#### 3.5 为各节点创建和分发 etcd systemd unit 文件

**替换模板文件中的变量，为各节点创建 systemd unit 文件：**

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 3; i++ ))
  do
    sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" etcd.service.template > etcd-${NODE_IPS[i]}.service 
  done
ls *.service
```

> 注：所有操作在ks-master上执行

* NODE_NAMES 和 NODE_IPS 为相同长度的 bash 数组，分别为节点名称和对应的 IP；

**分发生成的 systemd unit 文件：**

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp etcd-${node_ip}.service root@${node_ip}:/etc/systemd/system/etcd.service
  done
```
> 注：所有操作在ks-master上执行


[etcd.service](../common/etcd.service)



#### 3.6 启动 etcd 服务

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "mkdir -p ${ETCD_DATA_DIR} ${ETCD_WAL_DIR}"
    ssh root@${node_ip} "systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd " &
  done
```

> 注：所有操作在ks-master上执行

- 必须创建 etcd 数据目录和工作目录;
- etcd 进程首次启动时会等待其它节点的 etcd 加入集群，命令 systemctl start etcd 会卡住一段时间，为正常现象。

#### 3.7 检查启动结果

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "systemctl status etcd|grep Active"
  done

# 执行结果
>>> 10.10.12.143
   Active: active (running) since Tue 2019-03-19 15:55:22 CST; 1h 26min ago
>>> 10.10.12.142
   Active: active (running) since Tue 2019-03-19 15:55:11 CST; 1h 26min ago
>>> 10.10.12.141
   Active: active (running) since Tue 2019-03-19 15:55:11 CST; 1h 26min ago


```

> 注：所有操作在ks-master上执行

#### 3.8 验证服务状态

**部署完 etcd 集群后，在任一 etc 节点上执行如下命令：**

```
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ETCDCTL_API=3 /opt/k8s/bin/etcdctl \
    --endpoints=https://${node_ip}:2379 \
    --cacert=/opt/k8s/work/ca.pem \
    --cert=/etc/etcd/cert/etcd.pem \
    --key=/etc/etcd/cert/etcd-key.pem endpoint health
  done

# 执行结果

>>> 10.10.12.143
https://10.10.12.143:2379 is healthy: successfully committed proposal: took = 1.997671ms
>>> 10.10.12.142
https://10.10.12.142:2379 is healthy: successfully committed proposal: took = 1.289626ms
>>> 10.10.12.141
https://10.10.12.141:2379 is healthy: successfully committed proposal: took = 1.663937ms
```
> 注：所有操作在ks-master上执行


#### 5.8 查看当前的 leader

```
source /opt/k8s/bin/environment.sh
ETCDCTL_API=3 /opt/k8s/bin/etcdctl \
  -w table --cacert=/opt/k8s/work/ca.pem \
  --cert=/etc/etcd/cert/etcd.pem \
  --key=/etc/etcd/cert/etcd-key.pem \
  --endpoints=${ETCD_ENDPOINTS} endpoint status 

# 结果

+--------------------------+------------------+---------+---------+-----------+-----------+------------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
| https://10.10.12.143:2379 | a5819d9a1318a6fd |  3.3.10 |   20 kB |     false |         5 |         15 |
| https://10.10.12.142:2379 | 41f7138582b5f2c2 |  3.3.10 |   20 kB |      true |         5 |         15 |
| https://10.10.12.141:2379 | 97f11d52a9bffebb |  3.3.10 |   20 kB |     false |         5 |         15 |
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
```

> 注：IS LEADER 为true，则为leader

> 注：所有操作在ks-master上执行



### 4. 开始部署kubernetes 

#### 4.1 安装kubeadm、kubectl、kubelet

```
$ yum install -y kubelet kubeadm kubectl kubernetes-cni
$ systemctl enable kubelet && systemctl start kubelet
```

> 注：所有操作在所有节点执行上执行

#### 4.2 安装Master节点

**查看所需要的docker image**

```shell
[root@ks-master ~]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.14.0
k8s.gcr.io/kube-controller-manager:v1.14.0
k8s.gcr.io/kube-scheduler:v1.14.0
k8s.gcr.io/kube-proxy:v1.14.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```

> 注：如果要pull如上镜像需要搭个梯子的。。。。。所以接下来的操作我们需要更改镜像仓库进行pull

**部署 master**

[config.yaml](./config.yaml)


```
[root@ks-master ~]# kubeadm init --config config.yaml

初始化成功后输出信息：
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.10.12.143:6443 --token b99a00.a144ef80536d4344 --discovery-token-ca-cert-hash sha256:8c
```

**在ks-master上执行初始化命令**

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```


**查看状态**

```
$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}    
etcd-1               Healthy   {"health":"true"}
```

#### 4.3 部署网络组件

**修改系统设置，创建 flannel 网络**

```
[root@devops-101 ~]# sysctl net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-iptables = 1
[root@devops-101 ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds created
```

> flannel 默认会使用主机的第一张网卡，如果你有多张网卡，需要通过配置单独指定。修改 kube-flannel.yml 中的以下部分

```
containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=enp0s3            #指定内网网卡
```

> 执行成功后，Master并不能马上变成Ready状态，稍等几分钟，就可以看到所有状态都正常了。

```
[root@ks-master ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY     STATUS    RESTARTS   AGE
kube-system   coredns-78fcdf6894-8sd6g             1/1       Running   0          14m
kube-system   coredns-78fcdf6894-lgvd9             1/1       Running   0          14m
kube-system   etcd-devops-101                      1/1       Running   0          13m
kube-system   kube-apiserver-devops-101            1/1       Running   0          13m
kube-system   kube-controller-manager-devops-101   1/1       Running   0          13m
kube-system   kube-flannel-ds-6zljr                1/1       Running   0          48s
kube-system   kube-proxy-bhmj8                     1/1       Running   0          14m
kube-system   kube-scheduler-devops-101            1/1       Running   0          13m
[root@ks-master ~]# kubectl get nodes
NAME         STATUS    ROLES     AGE       VERSION
ks-master   Ready     master    14m       v1.14.0
```

#### 4.3 加入节点

**在所有节点执行**

```
$kubeadm join 10.10.12.143:6443 --token b99a00.a144ef80536d4344 --discovery-token-ca-cert-hash sha256:51985223a369a1f8c226f3ccdcf97f4ad5ff201a7c8c708e1636eea0739c0f05
```


**稍等片刻查看节点状态**

```
[root@ks-master ~]# kubectl get nodes
NAME        STATUS   ROLES    AGE     VERSION
ks-master   Ready    master   3d18h   v1.14.0
ks-node1    Ready    <none>   3d18h   v1.14.0
ks-node2    Ready    <none>   3d18h   v1.14.0
```

> 到此为止，kubernetes集群部署完毕，其他组件部署请参考其他文档

