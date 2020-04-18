---
title: k8s使用kong-ingress（安装postgres数据库）
date: 2020-04-18 20:31:36
tags: K8s,分布式
---



## 执行前置工作

如果对[Helm](https://github.com/helm/helm)比较熟悉，可以试一下[Kong via Helm](https://hub.kubeapps.com/charts/stable/kong)，更关心部署细节所以这里用的是[Kong via Manifest Files](https://docs.konghq.com/install/kubernetes/)，相关文件位于仓库[Kong/kong-dist-kubernetes](https://github.com/Kong/kong-dist-kubernetes/)中，用git获取：

```
git clone https://github.com/Kong/kong-dist-kubernetes.git
cd kong-dist-kubernetes
git checkout 2.0.0
```

[文档](https://docs.konghq.com/install/kubernetes/)中前两步分别是准备Kubernetes集群、部署数据库，已经有现成的环境不需要再折腾，直接进入第三步： 数据库初始化，使用的数据库是PostgreSQL。 文档中给出的方法是`kubectl create -f kong_migration_postgres.yaml`，yaml文件内容如下，创建了一个Job：

~~~yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kong-migration
spec:
  template:
    metadata:
      name: kong-migration
    spec:
      containers:
      - name: kong-migration
        image: kong:1.5.1-centos
        env:
          - name: KONG_NGINX_DAEMON
            value: 'off'
          - name: KONG_PG_PASSWORD
            value: kong
          - name: KONG_PG_HOST
            value: postgres.default.svc.cluster.local
        command: [ "/bin/sh", "-c", "kong migrations bootstrap" ]
      restartPolicy: Never
~~~

Job中的操作很简单，就是执行`kong migrations bootstrap`，完成数据库的初始化，环境变量`KONG_PG_PASSWORD`和`KONG_PG_HOST`的值要根据自己的环境更改。 这里更让人关心的是image是从哪来的？包含哪些内容？使用的kong的版本是多少？

>  这里我们的kong Image 最好和 上一章中从官网下载yaml的Kong版本一致 



查看pod创建情况

~~~bash
[root@m kongs]# kubectl get pods -n kong 
NAME                            READY   STATUS      RESTARTS   AGE
ingress-kong-8494d44445-zfjl4   2/2     Running     1          3h11m
kong-migration-6lr74            0/1     Completed   0          3h12m
~~~

>  因为他是一个Job，这里执行完一次就结束了 所以当前status为`Completed`



注意： 以上操作如果不做的话，执行Kong的pod会报错：`kong  Database needs bootstrapping; run 'kong migrations bootstrap'`



# 创建Postgres的pod

从`https://github.com/Kong/kong-dist-kubernetes/` 仓库中下载`postgres.yaml`文件，具体内容：

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: kong
spec:
  ports:
  - name: pgql
    port: 5432
    targetPort: 5432
    protocol: TCP
  selector:
    app: postgres

---
apiVersion: v1
kind: ReplicationController
metadata:
  name: postgres
  namespace: kong
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:9.6
          env:
            - name: POSTGRES_USER
              value: kong
            - name: POSTGRES_PASSWORD
              value: kong
            - name: POSTGRES_DB
              value: kong
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - containerPort: 5432
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: pg-data
      volumes:
        - name: pg-data
          emptyDir: {}
~~~

就是简单地创建一个postgres `ReplicationController`

执行`kubectl apply postgres.yaml `



使用`kubectl get ` 查看Pod

~~~bash
[root@m kongs]# kubectl get pods -n kong 
NAME                            READY   STATUS      RESTARTS   AGE
ingress-kong-8494d44445-zfjl4   2/2     Running     1          3h11m
kong-migration-6lr74            0/1     Completed   0          3h12m
postgres-zlwn7                  1/1     Running     0          3h13m
~~~



此时数据库已经构建完成，修改kong的yaml文件中的postgre连接信息



# 修改 KONG 管理 postgres

通过配置`kong-ingress-dbless.yaml `

~~~yaml
spec:
      containers:
      - env:
        - name: KONG_DATABASE
          value: "postgres"
        - name: KONG_NGINX_WORKER_PROCESSES
          value: "1"
        - name: KONG_NGINX_HTTP_INCLUDE
          value: /kong/servers.conf
        - name: KONG_ADMIN_ACCESS_LOG
          value: /dev/stdout
        - name: KONG_ADMIN_ERROR_LOG
          value: /dev/stderr
        - name: KONG_ADMIN_LISTEN
          value: 0.0.0.0:8001
        - name: KONG_PROXY_LISTEN
          value: 0.0.0.0:8000, 0.0.0.0:8443 ssl http2
        - name: KONG_PG_HOST
          value: "postgres"
        - name: KONG_PG_USER
          value: "kong"
        - name: KONG_PG_PASSWORD
          value: "kong"
        - name: KONG_PG_DATABASE
          value: "kong"
        image: kong:1.5.1-centos
        lifecycle:
          preStop:
            exec:
              command:
              - kong migrations bootstrap
              - /bin/sh
              - -c
              - kong quit
        name: proxy
        ports:
        - containerPort: 8001
          name: admin
          protocol: TCP
        - containerPort: 8000
          name: proxy
          protocol: TCP
        - containerPort: 8443
          name: proxy-ssl
          protocol: TCP
        - containerPort: 9542
          name: metrics
          protocol: TCP
        securityContext:
          runAsUser: 1000
        volumeMounts:
        - mountPath: /kong
          name: kong-server-blocks
      - args:
        - /kong-ingress-controller
~~~

`KONG_DATABASE` 设置`postgres` 

`KONG_PG_HOST` 设置`postgres` （service的name名称）

`KONG_PG_USER` 设置 postgres的用户名

`KONG_PG_PASSWORD` 设置postgres的密码

`KONG_PG_DATABASE` 设置postgres的数据库

这里主要和上一步的`创建Postgres的pod` 的配置文件一致



# 重新执行kong-ingress-dbless.yaml



