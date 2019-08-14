StorageClass
=================

如果我们后端存储使用NFS，那么我们就需要使用nfs-client自动配置程序（Provisioner），这个程序使用我们已经配置好的NFS服务器，来自动创建持久卷，也就是自动帮我们创建PV


* 自动创建的 PV 以${namespace}-${pvcName}-${pvName}这样的命名格式创建在 NFS 服务器上的共享数据目录中

* 而当这个 PV 被回收后会以archieved-${namespace}-${pvcName}-${pvName}这样的命名格式存在 NFS 服务器上。

## 创建NFS-Client


```
root@ks-master:~# git clone https://github.com/baishuchao/kubernetes.git
Cloning into 'kubernetes'...
remote: Enumerating objects: 137, done.
remote: Counting objects: 100% (137/137), done.
remote: Compressing objects: 100% (94/94), done.
remote: Total 395 (delta 47), reused 128 (delta 41), pack-reused 258
Receiving objects: 100% (395/395), 1.06 MiB | 192.00 KiB/s, done.
Resolving deltas: 100% (140/140), done.

root@ks-master:~# cd kubernetes/StorageClass/
root@ks-master:~/kubernetes/StorageClass# ll
total 24
drwxr-xr-x  2 root root 4096 Jul 19 11:38 ./
drwxr-xr-x 20 root root 4096 Jul 19 11:38 ../
-rw-r--r--  1 root root  185 Jul 19 11:38 nfs-client-class.yaml
-rw-r--r--  1 root root 1060 Jul 19 11:38 nfs-client-sa.yaml
-rw-r--r--  1 root root  847 Jul 19 11:38 nfs-client.yaml
-rw-r--r--  1 root root   33 Jul 19 11:38 README.md

root@ks-master:~/kubernetes/StorageClass# kubectl apply -f .

```


## 创建完成后查看下资源状态

```
root@ks-master:~/kubernetes/StorageClass# kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-587469c7f9-csqg6   1/1     Running   0          19m

root@ks-master:~/kubernetes/StorageClass# kubectl get sc
NAME         PROVISIONER      AGE
es-data-db   fuseim.pri/ifs   18m

```

## 创建demo测试

```
root@ks-master:~/kubernetes/StorageClass# kubectl apply -f nginx-demo.yaml 
statefulset.apps/nfs-web created

## 我们可以看到是不是也生成了3个 PVC 对象，名称由模板名称 name 加上 Pod 的名称组合而成，这3个 PVC 对象也都是 绑定状态了，很显然我们查看 PV 也可以看到对应的3个 PV 对象

root@ks-master:/data/k8s# ll
total 28
drwxrwxrwx 7 root  root  4096 Jul 19 11:45 ./
drwxr-xr-x 6 root  root  4096 Jul 19 11:15 ../
drwxrwxrwx 2 root  root  4096 Jul 19 11:45 default-www-nfs-web-0-pvc-04557632-303e-4fba-b6a9-11fdcaeca200/
drwxrwxrwx 2 root  root  4096 Jul 19 11:45 default-www-nfs-web-1-pvc-f5285f03-2a76-4f9c-b54c-bb32cb25666d/
drwxrwxrwx 3 kaifa kaifa 4096 Jul 19 11:25 logging-data-es-cluster-0-pvc-fb2c586b-fe32-4145-b1f6-f7e4bbef843c/
drwxrwxrwx 3 kaifa kaifa 4096 Jul 19 11:30 logging-data-es-cluster-1-pvc-d89e9db2-730a-4b69-82a3-c60cbbdbae53/
drwxrwxrwx 3 kaifa kaifa 4096 Jul 19 11:30 logging-data-es-cluster-2-pvc-55f98f3b-90e2-4504-90cb-de96e6b01eb5/
root@ks-master:/data/k8s# 

```

