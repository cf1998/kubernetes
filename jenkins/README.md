# Jenkins 部署

## 创建namespace

* pv是通过NF网络存储创建
* 容器的 /var/jenkins_home 目录通过pvc进行挂载并持久化存储


```
root@ks-master:~# kubectl get namespaces ops

```

## 通过编排文件安装Jenkins

```
root@ks-master:~# git clone https://github.com/baishuchao/kubernetes.git
root@ks-master:~# cd kubernetes/jenkins/
root@ks-master:~/kubernetes/jenkins# kubectl apply -f .
```

