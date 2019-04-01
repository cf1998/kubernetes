Table of Contents
=================


<!-- TOC -->

- [目录](#目录)
    - [Ingress 组件](#ingress-组件)
        - [1. Ingress 简介](#1-ingress-简介)
        - [2. 部署 Ingress](#2-部署-ingress)
        - [3. 使用ingress发布项目](#3-使用ingress发布项目)
        - [4.  验证是否成功](#4--验证是否成功)
        - [5. 配置TLS ingress](#5-配置tls-ingress)

<!-- /TOC -->


# 目录                                                                                                                                                                                                                                                                                                   　　　           
## Ingress 组件

### 1. Ingress 简介

> Kubernetes 提供了两种内建的云端负载均衡机制用于发布公共应用，一种是工作于传输层的Service资源，它实现的是“TCP负载均衡器”，另一种是Ingress资源，它实现的是“HTTP/HTTPS负载均衡器”。

- TCP负载均衡器
无论是iptables还是ipvs模型的service资源都配置于Linux内核中的Netfilter之上进行四层调度，是一种类型更改为通用的调度器，支持调度HTTP，MySQL等应用服务。不过，也正是由于工作于传输层从而使得它无法做到类似卸载HTTPS中的SSL会话等一类操作，也不支持基于URL的请求调度机制，而且，Kubernets也不支持为此类负载均衡器配置任何类型的健康状态检查机制。

- HTTP(s)负载均衡器
HTTP(s) 负载均衡器是应用负载均衡机制的一种，支持根据环境做出更好的调度决策。与传输层调度相比，它提供了诸如可自定义URL映射和TLS卸载等功能，并支持多种类型的后端服务器健康状态检查机制。


### 2. 部署 Ingress

```yaml
[root@ks-master ~]# wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
[root@ks-master ~]# kubectl  apply -f mandatory.yaml 
[root@ks-master ~]# kubectl get pods --all-namespaces |grep ingress
ingress-nginx       traefik-ingress-controller-gwcjk         1/1     Running   0         
ingress-nginx       traefik-ingress-controller-rphgz         1/1     Running   0         
ingress-nginx       traefik-ingress-controller-sd5l7         1/1     Running   0

```
> 注释：全部ks-master 上执行


### 3. 使用ingress发布项目

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana
  namespace: default
spec:
  rules:
  - host: grafana.baishuchao.com
    http:
      paths:
      - path: /
        backend:
          serviceName: grafana
          servicePort: 3000
```


### 4.  验证是否成功

通过浏览器访问 `grafana.baishuchao.com`

### 5. 配置TLS ingress

