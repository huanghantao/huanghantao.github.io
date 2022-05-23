---
title: sealos重新创建k8s集群后coredns一直pending问题
date: 2022-05-23 10:02:30
tags:
---

版本：

```bash
sealos version
Version: 3.3.9-rc.11
Last Commit: 49e79d2
Build Date: 2022-02-17T06:17:34Z

cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)

kubectl version
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.8", GitCommit:"7061dbbf75f9f82e8ab21f9be7e8ffcaae8e0d44", GitTreeState:"clean", BuildDate:"2022-03-16T14:10:06Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.8", GitCommit:"7061dbbf75f9f82e8ab21f9be7e8ffcaae8e0d44", GitTreeState:"clean", BuildDate:"2022-03-16T14:04:34Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
```

在一个纯净的虚拟机里面使用`sealos`部署`k8s`成功了。后面使用如下命令对集群进行了清理：

```bash
sealos clean --all
```

然后使用如下命令进行安装：

```bash
sealos init --passwd 'xxxxxxxx' --master 10.120.117.8  --node 10.120.116.2 --pkg-url /root/kube1.22.8.tar.gz --version v1.22.8
```

发现集群`nodes`一直处于`notReady`的状态，并且`coredns`也启动不了：

```bash
kubectl get nodes
NAME      STATUS        ROLES                  AGE     VERSION
master    NotReady      control-plane,master   5m21s   v1.22.8
worker1   NotReady      <none>                 4m56s   v1.22.8

kubectl -n kube-system get pods
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-78d6f96c7b-5kpql   0/1     Pending   0          2m53s
calico-node-lfm5c                          1/1     Running   0          2m45s
calico-node-npssm                          1/1     Running   0          2m54s
coredns-78fcd69978-2jmjj                   0/1     Pending   0          2m53s
coredns-78fcd69978-clrv5                   0/1     Pending   0          2m53s
etcd-master                                1/1     Running   2          3m3s
kube-apiserver-master                      1/1     Running   2          3m10s
kube-controller-manager-master             1/1     Running   2          3m3s
kube-proxy-5h65c                           1/1     Running   0          2m54s
kube-proxy-xj7cw                           1/1     Running   0          2m45s
kube-scheduler-master                      1/1     Running   2          3m10s
kube-sealyun-lvscare-worker1               1/1     Running   2          2m44s
```

这是因为`sealos clean`的时候不会卸载`containerd`，但是会删除/etc/cni这个目录，重新部署的时候会创建这个目录，但是`containerd`现在忽略了这个`event`，所以要重启`containerd`重新加载`cni plugin`。

具体的讨论在[这里](https://github.com/labring/sealos/issues/834)。
