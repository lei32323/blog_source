---
title: 安装K8s成功之后，重启虚拟机采坑指南
date: 2020-03-06 20:14:36
tags: K8s,分布式
---

是这样的，最近有些小伙伴通过kubeadm成功安装好了K8s。但是“好景不长”，第二天再次打开虚拟机打算玩会K8s的时候，发现问题出现了，再也用不起来了，欲哭无泪，咋办？咋办？怎么办？
下面我们就针对大家可能出现的问题说说对应的解决方案。

## 01 终极解决方案

把虚拟机删除掉，然后重新安装虚拟机，根据之前的安装文档重头开始。
what？老师，你在逗我吧？这也算解决方案？
哈哈，开个玩笑。其实也未尝不可，毕竟多安装一次能够更加熟练一些，如果大家真的打算使用这个终极解决方案，那么安装的过程中一定要注意的是：

（1）以后关闭虚拟机要优雅关机

（2）安装docker之后，所有节点设置docker默认为开机启动

`systemctl start docker && systemctl enable docker`
（3）安装kubelet之后，所有节点设置kubelet默认为开机启动

`systemctl start kubelet && systemctl enable kubelet`
其实上面这些操作主要是希望虚拟机再次开起的时候，K8s能够正常跑起来。

## 02 倒数第二终极解决方案
在每台节点上都执行一下，kubeadm reset，也就是打算重新搭建K8s集群了，这时候回到文档的kubeadm init开始往下操作即可。
这个方案相比第一个要好一些，因为不需要从头开始，只需要从kubeadm init初始化开始，相对还是美滋滋的。

## 03 输入kubectl get …，发现报错
错误信息类似于这种：
`The connection to the server 192.168.8.51:6443 was refused - did you specify the right host or port?`

如何解决
首先看一下你执行kubectl命令的节点，默认一定要在master上，worker上执行无效[虽然也可以配置，但是默认是无效的哦]
其次，重新启动一下docker，并且设置为默认开机启动：systemctl restart docker && systemctl enable docker

## 04 kubectl get nodes，变成NotReady
（1）`kubectl get pods -n kube-system`

检查当前的网络插件是否正常，比如calico，如果不正常或者没有安装，参照课程文档中的方式正确安装网络插件calico

（2）重新启动kubelet并且设置为默认开机启动

可以先查看一下kubelet的日志：
`journalctl -u kubelet -f`
然后重新启动每个节点中的kubelet服务：
`systemctl restart kubelet && systemctl enable kubelet`

（3）文档中的设置hostname和修改hosts文件一定要记得做，然后重启虚拟机

不然不能通过主机名访问到彼此，也会导致NotReady。
`sudo hostnamectl set-hostname 主机名`，记得用sudo权限哦，不然重启后主机名称可能会变。
永久修改hostname：vi /etc/sysconfig/network，最后增加一行，比如hostname=m这种。