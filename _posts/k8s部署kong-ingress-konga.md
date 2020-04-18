---
title: k8s使用kong-ingress（安装konga+postgres数据库）
date: 2020-04-18 20:30:36
tags: K8s,分布式
---

### 安装konga

具体konga.yaml  如下  :  

~~~yaml
---
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    description: Kong UI
  name: kong
---
apiVersion: v1
kind: Service
metadata:
  name: konga
  namespace: kong
  labels:
    app.kubernetes.io/name: konga
    app.kubernetes.io/instance: konga
spec:
  type: NodePort 
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      nodePort: 8080
      name: http
  selector:
    app.kubernetes.io/name: konga
    app.kubernetes.io/instance: konga
---
apiVersion: v1
kind: ConfigMap
metadata:
  name:  konga-config
  namespace: kong
  labels:
    app.kubernetes.io/name: konga
    app.kubernetes.io/instance: konga
data:
  KONGA_SEED_USER_DATA_SOURCE_FILE: "/etc/konga-config-files/userdb.data"
  KONGA_SEED_KONG_NODE_DATA_SOURCE_FILE: "/etc/konga-config-files/kong_node.data"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name:  konga-config-file
  namespace: kong
  labels:
    app.kubernetes.io/name: konga
    app.kubernetes.io/instance: konga
data:
  userdb.data: |-
    module.exports = [
        {
            "username": "admin",
            "email": "admin@test.com",
            "firstName": "Admin",
            "lastName": "Administrator",
            "admin": true,
            "active" : true,
            "password": "admin@123"
        }
    ]
  kong_node.data: |-
    module.exports = [
        {
            "name": "kong",
            "type": "default",
            "kong_admin_url": "http://kong-admin:8001",
            "health_checks": false
        }
    ]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: konga
  namespace: kong
  labels:
    app.kubernetes.io/name: konga
    app.kubernetes.io/instance: konga
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: konga
      app.kubernetes.io/instance: konga
  template:
    metadata:
      labels:
        app.kubernetes.io/name: konga
        app.kubernetes.io/instance: konga
    spec:
      containers:
        - env:
            - name: "DB_ADAPTER"
              value: "postgres"
            - name: "DB_URI"
              value: "postgresql://kong:kong@postgres/kong"
          name: konga
          image: pantsel/konga:next
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 1337
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          envFrom:
            - configMapRef:
                name: konga-config
          volumeMounts:
          - mountPath: /etc/konga-config-files
            name: config-file-volume
          resources:
            requests:
              cpu: 500m
              memory: 256Mi
      volumes:
      - configMap:
          defaultMode: 420
          name: konga-config-file
        name: config-file-volume

~~~

>~~~yaml
>- env:
>    - name: "DB_ADAPTER"
>      value: "postgres"
>    - name: "DB_URI"
>      value: "postgresql://kong:kong@postgres/kong"
>~~~
>
>这里主要配置连接postgres的信息
>
>DB_URI中的`postgres`为k8s中service的name



执行

~~~bash
kubectl apply -f ingress-kong/konga.yaml
~~~

查看konga pod

~~~bash
[root@m kongs]# kubectl get pods -n kong 
NAME                            READY   STATUS      RESTARTS   AGE
ingress-kong-8494d44445-zfjl4   2/2     Running     1          163m
konga-747d8bcf6b-xv6tb          1/1     Running     0          158m
~~~

konga的svc是NodePort类型，映射端口为8080

~~~bash
[root@m kongs]# kubectl get svc -n kong 
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
kong-admin                ClusterIP   10.101.24.223   <none>        8001/TCP                     3h43m
kong-proxy                NodePort    10.98.221.80    <none>        80:38960/TCP,443:33166/TCP   3h43m
kong-validation-webhook   ClusterIP   10.100.68.169   <none>        443/TCP                      3h43m
konga                     NodePort    10.99.31.6      <none>        80:8080/TCP                  3h40m
~~~

浏览器访问http://:8080； 点击APPLICATION->CONNECTIONS->ACTIVATE 可看到界面规则信息，直接调的kong-admin api获取



打开页面后

![image-20200418211245746](https://wanglei.club/image-20200418211245746.png)