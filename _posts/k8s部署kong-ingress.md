---
title: k8s使用kong-ingress
date: 2020-04-18 19:55:36
tags: K8s,分布式
---



# 什么是Kong

ong 基于一个基于[nginx](https://nginx.org/en/)和[Lua语言（](https://www.lua.org/)称为[OpenResty）](https://openresty.org/en/)的Web平台，并具有nginx的高性能，Lua实现的灵活和高级功能以及插件机制的可扩展性。有。诸如路由和访问控制之类的大多数设置都保存在[PostgreSQL](https://www.postgresql.org/)或[Cassandra的](https://cassandra.apache.org/)数据库中，并且可以通过Kong的REST API（管理API）进行管理。（从Kong 1.1.0开始，还支持带有声明性配置文件的DBless模式。）

此Admin API可以管理的主要资源是服务，路由和插件。服务代表在Kong后面运行并实际实现API的后端服务，主要用于设置请求路由的目的地。路由是与一个服务关联的资源，主要用于设置路由到该服务的请求的条件。插件用于链接各种服务和路由，并为请求及其响应路由设置各种控件。

Kong和社区提供了[各种](https://docs.konghq.com/hub/)插件，[您可以](https://docs.konghq.com/hub/)执行[各种操作，例如](https://docs.konghq.com/hub/)[基本身份验证](https://docs.konghq.com/hub/kong-inc/basic-auth/)，[LDAP链接](https://docs.konghq.com/hub/kong-inc/ldap-auth/)，[访问控制列表](https://docs.konghq.com/hub/kong-inc/acl/)，[Prometheus链接](https://docs.konghq.com/hub/kong-inc/prometheus/)，[响应处理](https://docs.konghq.com/hub/kong-inc/response-transformer/)等。由于该插件也是由Lua制作的，因此可以对现有插件进行一些修改，并且创建新插件相对容易。

# Kong配置管理中的挑战

基本上，Kong设置是，如果您使用Admin API处理它，它将被保存在PostgreSQL等数据库中。Admin API是一种典型的REST API，并且由于它是一个说明性和顺序性的设置过程，因此如今在吹捧声明式操作时，似乎有些过时了。当我将Kong视为微服务之一时，似乎有状态的PostgreSQL坚持使用了它，这使其操作起来更加费力并且令人不快。

![imparative](https://wanglei.club/imparative.png)

如果在无DB模式下运行Kong，则不需要DBMS，但是过程是在启动时从设置文件中读取初始设置，并在更改设置时重写并重新加载设置文件，因此很难使用。

# Ingress和Ingress Controller

可以帮助解决该问题的Kubernetes功能称为[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)和[Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)。

入口通常由诸如L7的负载均衡器之类的软表达式来解释，这是一种用于暴露HTTP（S）访问点的机制，但是总而言之，它是一种Kubernetes资源，用于表达反向代理的设置。  `kind: Ingress`Kubernetes内置有一个API，您可以在其清单中编写通用的反向代理设置，并在Kubernetes中注册它。但是，Kubernetes本身没有反向代理功能，因此注册Ingress资源没有任何作用。需要使用Ingress Controller来利用Ingress资源。

Ingress Controller是监视Ingress资源创建和修改的控制器服务与根据资源定义运行的反向代理的组合。该反向代理的访问端口通过NodePort类型或LoadBalancer类型的服务暴露在Kubernetes集群外部，以便用户可以访问。通过部署Ingress Controller和注册Ingress资源，将可以使用反向代理接收和处理来自用户的请求。

顺便说一句，负载均衡器服务控制着Kubernetes集群外部的负载均衡器和NodePort服务，并且通过NodePort服务以很好的方式将用户对负载均衡器的请求路由到Pod。换句话说，它是一种服务，如果没有外部负载平衡器和一些控制它的控制器，就不能使用它，并且通常仅在托管的Kubernetes环境（例如GKE）中使用。（[如果](https://metallb.universe.tf/)使用MetalLB，则可以在内部使用它。）

## Ingress Controller

对于每个反向代理实现，Ingress Controller都有[各种](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/#additional-controllers)实现。其中之一是Kong Ingress Controller。Kong Ingress Controller使用Kong（当然）将其用作反向代理，其Controller服务读取Ingress资源的定义并调用Kong的Admin API来使设置正确。

入口资源是反向代理的一般定义，因此不足以表示高级Kong设置。因此，Kong Ingress Controller使用[Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)弥补了这一不足。简而言之，自定义资源是Kubernetes API的扩展。Kong Ingress Controller提供了Kong Ingress和Kong Plugin等自定义资源（详细信息将在后面描述），以便可以表达Kong的详细设置。



![declarative](https://wanglei.club/declarative.png) 

如果使用Kong Ingress Controller，则可以使用Kubernetes API声明性地设置Kong，这样就可以解决上述问题。



# Kong Ingress Controller处理的Kubernetes资源

Kong Ingress Controller处理的Ingress资源和自定义资源的摘要。

- Ingress

  Kubernetes内置资源。定义要路由的请求条件和路由目标的服务（Kubernetes）。根据此定义，Kong Ingress Controller会设置Kong Route和Service。

- Kong Ingress

  Kong Ingress Controller自定义资源。将其与Ingress结合使用，并描述仅由Ingress无法表达的路由设置。`configuration.konghq.com`通过使用称为Ingress注释的键指定Kong Ingress 的名称来进行链接。

- Kong Plugin

  Kong Ingress Controller自定义资源。用于定义Kong插件设置。可以`plugins.konghq.com`通过使用诸如Ingress或Service注解之类的键指定KongPlugin的名称来进行链接。 `global: "true"`如果您创建带有标签的KongPlugin，则该插件将应用于所有请求。



还有一个自定义资源，用于处理身份验证信息，称为KongConsumer和KongCredential。但是，我这次不会使用，因此将省略细节。

# 阅读Kong Ingress Controller的官方清单

[Kong Ingress Controller 0.8.0的正式清单](https://github.com/Kong/kubernetes-ingress-controller/blob/0.6.2/deploy/single/all-in-one-dbless.yaml)中定义了以下资源。

- 命名空间

  `kong`命名空间。以下所有资源都属于。

- CustomResourceDefinition（4）

  自定义资源的定义，例如KongIngress和KongPlugin。

- 集群角色

  一种角色，提供诸如节点，容器，入口，服务等资源，以及上述自定义资源的引用权限。

- 服务帐号

  用于将上述ClusterRole与控制器服务相关联的帐户名。

- ClusterRoleBinding

  将Controller服务与上述ClusterRole关联的定义。

- ConfigMap

  定义文件，将用于度量获取和运行状况检查的API添加到Kong（nginx）。

- 服务（kong-proxy）

  一种LoadBalancer类型的服务，用于在外部公开Kong访问端口。

- 服务（kong-validation-webhook）

  一个ClusterIP类型的服务，该服务将控制器服务端口公开到集群内部（kube-apiserver），以便在注册或更改上述自定义资源时启用验证。

- 部署方式

  启动Kong和控制器服务的定义。Kong设计为在无DB模式下运行。

有很多资源，但是查看每个资源并不是那么困难。

# 部署Kong Ingress Controller

现在部署。

我们要部署到的Kubernetes集群是[最近使用CentOS 8创建的，](https://www.kaitoy.xyz/2019/12/05/k8s-on-centos8-with-containerd/)而Kubernetes版本是1.14.0。Kong Ingress Controller的最新版本是0.8.0。

先从[官网](https://docs.konghq.com/2.0.x/kong-for-kubernetes/install/)文档中找到yaml文件并且 [下载](kubectl apply -f https://bit.ly/kong-ingress-dbless) kong-ingress-dbless

 `kubectl apply -f https://bit.ly/kong-ingress-dbless`

> 注意：如果你的kubernetes是1.16版本的及以上的，Deployment的apiVersion版本`extensions/v1beta1`太旧，无法将其部署到Kubernetes 1.16，因此`apps/v1`请将其修复。校正差异是这样的。
>
> ~~~yaml
> -apiVersion: extensions/v1beta1
> +apiVersion: apps/v1
>  kind: Deployment
>  metadata:
>    labels:
>      app: ingress-kong
>    name: ingress-kong
> ~~~



Kong Ingress Controller的正式清单`kong-proxy`是LoadBalancer类型的服务，该服务(需要通过第三方运营商)难以使用，因此这次将其更改为NodePort。

~~~yaml
@@ -448,17 +448,18 @@
   ports:
   - name: proxy
     port: 80
     protocol: TCP
     targetPort: 8000
+    nodePort: 30080
   - name: proxy-ssl
     port: 443
     protocol: TCP
     targetPort: 8443
   selector:
     app: ingress-kong
-  type: LoadBalancer
+  type: NodePort
 ---
 apiVersion: v1
 kind: Service
 metadata:
   name: kong-validation-webhook
@@ -470,11 +471,11 @@
     protocol: TCP
     targetPort: 8080
   selector:
     app: ingress-kong
 ---
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   labels:
     app: ingress-kong
   name: ingress-kong
~~~

NodePort的端口号是，它将Kong的访问端口暴露在外面`30080`。

当我制作修改后的清单时，使用`kubectl apply`它可以正常启动。

~~~bash
[root@m kongs]# kubectl get pods -n kong 
NAME                            READY   STATUS      RESTARTS   AGE
ingress-kong-8494d44445-zfjl4   2/2     Running     1          163m
~~~

如果Kong容器没有启动，则控制器服务将在启动期间删除，因此在开始时几次执行CrashLoopBackOff会有些呆板。

将GET请求发送到Kong访问端口（即NodePort）时，会收到`no Route matched with those values`一条消息，说Kong还没有路由设置。（Kubernetes节点的IP地址为`192.168.1.101`）

~~~bash
$ NODE_IP=192.168.1.101
$ curl http://${NODE_IP}:30080/
{"message":"no Route matched with those values"}
~~~

现在我可以确定Kong服务已经启动成功了。

# 尝试Ingress

注册Ingress并进行Kong设置。

首先，为了部署仅返回请求内容的回显服务，作为Kong目的地的后端服务，请执行以下清单`kubectl apply`。

~~~yaml
piVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
  ports:
  - port: 8080
    protocol: TCP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - image: gcr.io/kubernetes-e2e-test-images/echoserver:2.2
        name: echo
        ports:
        - containerPort: 8080
~~~

确认一下

~~~bash
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
echo-588b79f67f-xxn56   1/1     Running   0          14h
~~~

说明启动了。该容器与`echo`服务相关联，并且服务的端口设置为8080。尝试将GET请求发送到该服务。

~~~bash
$ curl http://$(kubectl get svc echo -o jsonpath='{.spec.clusterIP}'):8080


Hostname: echo-588b79f67f-xxn56

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.12.2 - lua: 10010

Request Information:
        client_address=10.32.0.1
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://10.0.185.110:8080/

Request Headers:
        accept=*/*
        host=10.0.185.110:8080
        user-agent=curl/7.61.1

Request Body:
        -no body in request-
~~~

响应正确返回。



要创建路由到此echo服务的Ingress，请执行以下清单`kubectl apply`。 `/echo`当您访问时`8080`，发送到回显服务端口。

~~~yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo
spec:
  rules:
  - http:
      paths:
      - path: /echo
        backend:
          serviceName: echo
          servicePort: 8080
~~~

现在，您应该已经在Kong中设置了代表``echo服务`的服务及其路由。`8444`Admin API在Kong容器的端口处打开，您可以通过访问它来查看Kong设置。

~~~bash
$ kubectl exec -it -n kong ingress-kong-8494d44445-zfjl4 -c proxy -- curl -k https://localhost:8444/routes | jq
{
  "next": null,
  "data": [
    {
      "strip_path": true,
      "tags": null,
      "updated_at": 1575845092,
      "destinations": null,
      "headers": null,
      "protocols": [
        "http",
        "https"
      ],
      "created_at": 1575845092,
      "snis": null,
      "service": {
        "id": "dac20c80-66d8-59e2-8f57-66e046e18d45"
      },
      "name": "default.echo.00",
      "preserve_host": true,
      "regex_priority": 0,
      "id": "e97bc643-59d4-53c1-83e2-87169eaa28d5",
      "sources": null,
      "paths": [
        "/echo"
      ],
      "https_redirect_status_code": 426,
      "methods": null,
      "hosts": null
    }
  ]
}
$ kubectl exec -it -n kong ingress-kong-8494d44445-zfjl4 -c proxy -- curl -k https://localhost:8444/services | jq
{
  "next": null,
  "data": [
    {
      "host": "echo.default.svc",
      "created_at": 1575845092,
      "connect_timeout": 60000,
      "id": "dac20c80-66d8-59e2-8f57-66e046e18d45",
      "protocol": "http",
      "name": "default.echo.8080",
      "read_timeout": 60000,
      "port": 80,
      "path": "/",
      "updated_at": 1575845092,
      "client_certificate": null,
      "tags": null,
      "write_timeout": 60000,
      "retries": 5
    }
  ]
}
~~~

做完了

`/echo`尝试访问Kong中的节点端口。

~~~bash
$ NODE_IP=192.168.1.101
$ curl http://${NODE_IP}:30080/echo


Hostname: echo-588b79f67f-xxn56

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.12.2 - lua: 10010

Request Information:
        client_address=10.32.0.5
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://192.168.1.101:8080/

Request Headers:
        accept=*/*
        connection=keep-alive
        host=192.168.1.101:30080
        user-agent=curl/7.61.1
        x-forwarded-for=10.32.0.1
        x-forwarded-host=192.168.1.101
        x-forwarded-port=8000
        x-forwarded-proto=http
        x-real-ip=10.32.0.1

Request Body:
        -no body in request-
~~~

看来他能够通过Kong访问`echo服务`。 可以看到Kong已将`Request Headers``x-forwarded-for`其添加到中。

`Request Information`中 发现 `real path`却成了`/`。这是因为`strip_path`在默认情况下，`"Route of Kong"`会设置为true，因此Kong会截断`curl`在中发送的URL路径`/echo`。无法使用Ingress更改此设置，因此，如果要更改此设置，则需要KongIngress。

# 尝试Kong Ingress

`strip_path`让我们创建一个Kong Ingress，以便将上一节中创建的Route设置为false。首先，对Kong Ingress做以下清单`kubectl apply`。

~~~yaml
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: keep-path
route:
  strip_path: false
~~~

然后，为了将此KongIngress与上一节中创建的Ingress关联，请把Ingress `configuration.konghq.com`添加使用以下命令调用的注释。

~~~bash
kubectl patch ingress echo -p '{"metadata":{"annotations":{"configuration.konghq.com":"keep-path"}}}'
~~~

这`strip_path`应该更改`“ Kong Route”`设置。看一下

~~~bash
$ kubectl exec -it -n kong ingress-kong-8494d44445-zfjl4 -c proxy -- curl -k https://localhost:8444/routes | jq '.data[].strip_path'
false
~~~

`/echo`尝试访问Kong中的节点端口。

~~~bash
$ NODE_IP=192.168.1.101
$ curl -s http://${NODE_IP}:30080/echo | grep 'real path'
        real path=/echo
~~~

发现发送的URL路径中`real path`的值已经开始变成`/echo`。

# 试试KongPlugin

最后，使用KongPlugin启用[Correlation ID插件](https://docs.konghq.com/hub/kong-inc/correlation-id/)。这是一个将每个请求的唯一UUID附加到已应用请求的HTTP标头的插件。您可以将Service，Route等指定为启用插件的目标，但这一次它是全局的，适用于所有请求。

请执行以下清单`kubectl apply`。

~~~yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: correlation-id
  labels:
    global: "true"
config:
  header_name: Global-Request-ID
plugin: correlation-id
~~~

这将为您的Kong创建插件设置。使用Kong的Admin API获取插件并进行检查。

~~~bash
$ kubectl exec -it -n kong ingress-kong-8494d44445-zfjl4 -c proxy -- curl -k https://localhost:8444/plugins | jq
{
  "next": null,
  "data": [
    {
      "created_at": 1575852780,
      "config": {
        "echo_downstream": false,
        "header_name": "Global-Request-ID",
        "generator": "uuid#counter"
      },
      "id": "65d138a5-b9b3-5559-84d6-c459f3cc456a",
      "service": null,
      "enabled": true,
      "tags": null,
      "consumer": null,
      "run_on": "first",
      "name": "correlation-id",
      "route": null,
      "protocols": [
        "grpc",
        "grpcs",
        "http",
        "https"
      ]
    }
  ]
}
~~~



做完了 实际发送一个请求。

~~~bash
$ NODE_IP=192.168.1.101
$ curl -s http://${NODE_IP}:30080/echo


Hostname: echo-588b79f67f-xxn56

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.12.2 - lua: 10010

Request Information:
        client_address=10.32.0.5
        method=GET
        real path=/echo
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://192.168.1.101:8080/echo

Request Headers:
        accept=*/*
        connection=keep-alive
        global-request-id=caad53f7-2e95-483d-8515-79a45b6a52d3#2
        host=192.168.1.101:30080
        user-agent=curl/7.61.1
        x-forwarded-for=10.32.0.1
        x-forwarded-host=192.168.1.101
        x-forwarded-port=8000
        x-forwarded-proto=http
        x-real-ip=10.32.0.1

Request Body:
        -no body in request-
~~~

发现header中已经出现`global-request-id`的UUID值

# 总结

在Kubernetes集群中部署了Kong Ingress Controller，并确认可以由Ingress，KongIngress和KongPlugin设置Kong。

如果使用Kong Ingress Controller，则可以使用Kubernetes API声明性地管理Kong设置，这将会取得很大的进步。





[关于kong-ingress的配置详细在github上  https://github.com/lei32323/kubenetes-demo/tree/master/kong](https://github.com/lei32323/kubenetes-demo/tree/master/kong)


