---
title: Pod节点内无法访问ClusterIp
date: 2020-04-11 12:55:36
tags: K8s,分布式
---



1. 添加IPV4模块：

   ~~~bash
   cat > /etc/sysconfig/modules/ipvs.modules <<EOF
   modprobe -- ip_vs
   modprobe -- ip_vs_rr
   modprobe -- ip_vs_wrr
   modprobe -- ip_vs_sh
   modprobe -- nf_conntrack_ipv4
   EOF
   ~~~

2. 设置权限 

~~~bash
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
~~~



3. 修改kupe-proxy的模式：

~~~bash
kubectl edit cm kube-proxy -n kube-system
~~~

![img](https://www.lei32323.com/1337432-20200120154729046-415080258.png)

4. 重启kupe-proxy：

   ~~~bash
   kubectl get pod -n kube-system | grep kube-proxy |awk '{system("kubectl delete pod "$1" -n kube-system")}'
   ~~~

5. 查看日志：

~~~bash
kubectl logs -n kube-system kube-proxy-74vpx
~~~

![image-20200411204315655](https://www.lei32323.com/image-20200411204315655.png)

