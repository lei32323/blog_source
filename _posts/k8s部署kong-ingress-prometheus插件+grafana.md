---
title: k8s部署kong-ingress+prometheus插件+grafana
date: 2020-04-19 12:55:36
tags: K8s,分布式
---



# 安装Prometheus 插件



先查看部署的kong是否有metrics接口   

获取kong metrics

~~~bash
CLUSTER_IP=`kubectl -n kong get svc kong-admin --output=jsonpath={.spec.clusterIP}`
curl http://${CLUSTER_IP}:8444/metrics
~~~



## konga已经关联数据库

如果已经关联数据库了，那就直接在konga的界面就可以直接安装了

![image-20200419140311972](https://www.lei32323.com/image-20200419140311972.png)



选择prometheus插件进行安装

![image-20200419140350108](https://www.lei32323.com/image-20200419140350108.png)



这里可以直接点击ADD PLUGIN ,让所有的consumer使用(如果你需要指定consume，需要填写对应的comsumer ID)

![image-20200419140455663](https://www.lei32323.com/image-20200419140455663.png)



这个时候 发现已经创建了一个plugin  （默认是所有的Route都能使用）

![image-20200419140644208](https://www.lei32323.com/image-20200419140644208.png)



到这里就已经安装完成了。



你也可以对指定的route安装Prometheus插件



在Routes中， 选择需要安装插件的Route

![image-20200419140854957](https://www.lei32323.com/image-20200419140854957.png)



选择Prometheus插件进行安装

![image-20200419140942993](https://www.lei32323.com/image-20200419140942993.png)



![image-20200419141003917](https://www.lei32323.com/image-20200419141003917.png)



![image-20200419141028905](https://www.lei32323.com/image-20200419141028905.png)





这时你会发现已经添加了Prometheus 

![image-20200419141056731](https://www.lei32323.com/image-20200419141056731.png)

> 这里的插件只针对当前的选择的route，并不是全局的



在Plugins菜单中会发现有一个插件已经APPLY_TO到某一个Route了

![image-20200419141231544](https://www.lei32323.com/image-20200419141231544.png)



## konga 没有关联数据库

如果没有管理数据库的话，那么就不能通过konga安装了，被kong-admin服务禁止，这时需要通过yaml进行手动安装



加载prometheus plugin，新增annotations

```
# kubectl -n kong edit deployments. ingress-kong
spec:
  template:
    metadata:
      annotations:
        prometheus.io/port: "9542"
        prometheus.io/scrape: "true"
    spec:
      containers:
      - env:
        - name: KONG_PLUGINS
          value: ...,prometheus
```

声明为global，每个请求都被prometheus跟踪

创建`prometheus-plugin.yaml`

```
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  labels:
    global: "true"
  name: prometheus
plugin: prometheus
```

执行`kubcectl apply -f prometheus-plugin.yaml ` 



# 安装Prometheus Pod

下载`prometheus-namespace.yaml`

创建prometheus的namespace 

~~~bash
kubectl apply -f prometheus-namespace.yaml
~~~



下载`prometheus.yaml`文件

找到 `- targets: ['localhost:9090']` 这一行，添加 Kong 管理节点，添加后如下：

![image-20200419143031903](https://www.lei32323.com/image-20200419143031903.png)

> 这里的8444就是kong-admin的服务



### 安装 nfs服务器

因为当前`prometheus.yaml`中使用了nfs，所以我们需要提前准备好nfs的服务器

> nfs(network file system)网络文件系统，是FreeBSD支持的文件系统中的一种，允许网络中的计算机之间通过 TCP/IP网络共享资源

找到一台可以安装nfs的机器

1. 安装nfs

   ~~~bash
   yum install -y nfs-utils
   ~~~

2. 创建nfs目录 (需要根据prometheus.yaml中配置的PersistentVolume的path)

   ~~~bash
   mkdir -p /nfs/prometheus/data 
   mkdir -p /nfs/grafana/data    #因为下面的grafana 也需要nfs所以这里一起创建了
   ~~~

   ![image-20200419144309565](https://www.lei32323.com/image-20200419144309565.png)

3. 授予权限

   ~~~bash
   chmod -R 777 /nfs
   ~~~

4. 编辑export文件

   ~~~bash
   vi /etc/exports
      /nfs *(rw,no_root_squash,sync)
   ~~~

5. 使export文件生效

   ~~~bash
   exportfs -r
   ~~~

6. 查看生效

   ~~~bash
   exportfs
   ~~~

7. 启动rpcbind,nfs服务

   ~~~bash
   systemctl restart rpcbind && systemctl enable rpcbind 
   systemctl restart nfs && systemctl enable nfs
   ~~~

8. 查看rpc服务的注册情况

   ~~~bash
   rpcinfo -p localhost
   ~~~

   ![image-20200419143843283](https://www.lei32323.com/image-20200419143843283.png)

9. Showmount测试

   ~~~bash
   showmount -e 当前nfs的机器ip
   ~~~

   ![image-20200419143757694](https://www.lei32323.com/image-20200419143757694.png)

10. 在k8s的集群中都要安装nfs

    ~~~bash
    yum -y install nfs-utils
    ~~~

    ~~~bash
    systemctl start nfs && systemctl enable nfs
    ~~~

> **这里你的nfs 要么把端口都要放开，要么直接把防火墙关掉，否则其他机器是连不上的nfs服务的**



### 运行prometheus.yaml

运行之前，还要把`prometheus.yaml`中的nfs服务器地址换成我们刚才的nfs服务器

![image-20200419144402627](https://www.lei32323.com/image-20200419144402627.png)



直接运行`kubecetl apply -f  prometheus.yaml  ` 就可以了



查看`kubectl get pods -n ns-monitor -o wide`  pod创建情况

![image-20200419144603057](https://www.lei32323.com/image-20200419144603057.png)



如果出现问题，使用

~~~bash
kubectl describe pod -n ns-monitor prometheus-5c9b8f5d68-wngb7
~~~

查看问题，再解决问题





接下来我们查看service 试着访问一下看看

~~~bash
kubectl get svc -n ns-monitor
~~~

![image-20200419144856600](https://www.lei32323.com/image-20200419144856600.png)

查看对外端口为60699 ，上面查看Pod 安装到了master01节点上，

那我们直接再浏览器上访问`http://192.168.0.200:60699/` 就能看到primetheus界面了

![image-20200419145247245](https://www.lei32323.com/image-20200419145247245.png)



选择Metrics名称 就能看到我们的一开始调用8444接口查看到的指标信息

![image-20200419145353806](https://www.lei32323.com/image-20200419145353806.png)

# 安装Grafana Pod

下载`grafana.yaml`文件

同样这里需要把nfs 的服务器地址更换到上面的nfs服务器地址

![image-20200419145630533](https://www.lei32323.com/image-20200419145630533.png)

如果上面没有创建grafana的nfs路径的话，也需要再nfs服务器中创建



执行

~~~bash
kubectl apply -f grafana.yaml
~~~





查看pod安装情况

~~~Bash
kubectl get pods -n ns-monitor -o wide 
~~~

![image-20200419145957071](https://www.lei32323.com/image-20200419145957071.png)



记住当前grafana安装在worker01节点



查看grafana的service信息

~~~bash
kubectl get svc -n ns-monitor
~~~

![image-20200419145911268](https://www.lei32323.com/image-20200419145911268.png)

当前对外的映射端口为40492



访问worker01节点的ip +40492 

~~~bash
http://192.168.0.205:40492/
~~~

首次登陆grafana  用户名和密码都是admin   



# 使用Grafana

## Grafana连接prometheus

选择菜单的data sources

![image-20200419150415575](https://www.lei32323.com/image-20200419150415575.png)



添加数据源

![image-20200419150440064](https://www.lei32323.com/image-20200419150440064.png)



![image-20200419150453625](https://www.lei32323.com/image-20200419150453625.png)





配置prometheus的连接地址

![image-20200419150523772](https://www.lei32323.com/image-20200419150523772.png)



> 因为我这里都是在K8s的集群中，这里直接配置了prometheus的service的名称
>
> 如果你也和我一样的话，请使用命令` kubectl get svc -n ns-monitor ` 查看你的service名称 和内部端口号
>
> ![image-20200419150716736](https://www.lei32323.com/image-20200419150716736.png)

最后点击``save&Test`  测试下连接是否正常

![image-20200419150758526](https://www.lei32323.com/image-20200419150758526.png)

说明正常。。



## 导入prometheus的dashboard

github中已经有对应的kong->prometheus->grafana的 dashboard，我们只需要导入就可以使用了

https://github.com/lei32323/kong-plugin-prometheus/blob/master/grafana/kong-official.json

找到grafana的Dashboard Manage

![image-20200419151027838](https://www.lei32323.com/image-20200419151027838.png)

选择Import 

选择刚才从giithub中下载的json

![image-20200419151102281](https://www.lei32323.com/image-20200419151102281.png)





最后选择grafana的 Dashboard 就能查看 到导入的dashboard了

![image-20200419151219176](https://www.lei32323.com/image-20200419151219176.png)



最后的效果图就是这样了

![image-20200419151301422](https://www.lei32323.com/image-20200419151301422.png)



[上面例子使用的Yaml都在 https://github.com/lei32323/kubenetes-demo/tree/master/kong](https://github.com/lei32323/kubenetes-demo/tree/master/kong)

