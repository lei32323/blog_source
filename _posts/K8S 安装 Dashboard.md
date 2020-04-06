---
title: K8S 安装 Dashboard
date: 2020-04-06 21:14:36
tags: K8s,分布式
---

# K8S 安装 Dashboard

### 下载 Dashboard yaml 文件

wget http://pencil-file.oss-cn-hangzhou.aliyuncs.com/blog/kubernetes-dashboard.yaml



打开下载的文件添加一项：`type: NodePort`，暴露出去 Dashboard 端口，方便外部访问。

~~~yaml
# ------------------- Dashboard Service ------------------- #
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort    # 新增
  ports:

   - port: 443
     targetPort: 8443
       selector:
         k8s-app: kubernetes-dashboard
~~~



### 部署

~~~bash
kubectl create -f  kubernetes-dashboard.yaml
~~~

~~~bash
kubectl get pods --all-namespaces -o wide | grep dashboard

kube-system   kubernetes-dashboard-5f59745d46-2rtwg        1/1     Running   0          10m
~~~

 >需要修改下载kubernetes-dashboard.yaml中的镜像 地址
 >
 >`registry.cn-hangzhou.aliyuncs.com/wanglei_k8s/kubernetes-dashboard-amd64:v1.10.1`



### 创建简单用户

创建 `dashboard-adminuser.yaml` 文件，加入以下内容：

~~~yaml
 vim dashboard-adminuser.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
~~~



执行脚本

~~~bash
kubectl apply -f dashboard-adminuser.yaml
~~~



查看生成的token 

~~~bash
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kubernetes-dashboard-admin-token | awk '{print $1}')

~~~

~~~text
Name:         kubernetes-dashboard-admin-q9rk7
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard-admin
              kubernetes.io/service-account.uid: b164245a-774e-11ea-a3e8-000c296a240b

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6IndhbmdsZWktdG9rZW4tcTlyazciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoid2FuZ2xlaSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImIxNjQyNDVhLTc3NGUtMTFlYS1hM2U4LTAwMGMyOTZhMjQwYiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OndhbmdsZWkifQ.FYPpLI8mDpWSf1cEky-y2w-THQhrwqG8ddrOGDvQGf3yOvVeYm0FzTUkbvWV72fNEWXuQHaLZIwTJtn0u7Ay-6cAqvppSVwt7Q608g8MGTGZ1UHx98hiyiwjTweLqOllaoduF9phse7-UoeuwVW7FgzVoPFo-OF7rkBs1JPY9ROiIUaCgF0nq2btSInM2B6OVApoo5-ihHs1w6ChBHBEpHB_7y55eDzth_utziCHFpOA4NG8OsMzCLYpZ-rOjClPxGxVur_4Jb6WGVL0eRr9Z-5w9WmuiQ9-ffaqVt550MciclPNAOVdSvk2FGVHZ47x251Z9ytBzEokchyxcJC52A
~~~



### 登录 Dashboard

​	查看 Dashboard 端口号

~~~bash
 kubectl get svc --all-namespaces
~~~

~~~bash
NAMESPACE     NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes             ClusterIP   10.96.0.1      <none>        443/TCP                  35h
kube-system   kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP,9153/TCP   35h
kube-system   kubernetes-dashboard   NodePort    10.111.92.24   <none>        443:31696/TCP            14m
~~~

   访问 Dashboard 

~~~text
https://192.168.0.105:31696/
~~~



输入token 

