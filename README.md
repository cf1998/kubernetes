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









