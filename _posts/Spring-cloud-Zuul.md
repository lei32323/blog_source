# Spring cloud Zuul

## Zuul基本用法

网关：Nginx+Lua

​	Lua:控制规则（A/B Test)

1. 在application启动类上上增加`@EnableZuulProxy` 

   ~~~java
   @SpringBootApplication
   @EnableDiscoveryClient  //启动服务注册，发现客户端
   @EnableZuulProxy //激活zuulProxy
   public class Application{
       public static void main(Stirng args[]){
           SpringApplication.run(Application.class,args);
       }
   }
   ~~~

2. 添加访问app路由规则

   ~~~properties
   ###zuul.routes.${application-name}=/${app-url-prefix}/**    *是当前目录，**是所有目录
   zuul.routes.student-service = /student-service/**
   server.port = 7070 
   ~~~



## 整合Ribbon

1. 启动rueka 服务
2. student-provider 服务启动

调用链路：zuul->student-provider

~~~properties
# 设置ribbon 的激活信息属性 
# ribbon 的 服务负载均衡列表
# 关闭Ribbon 对Eureka 的关联
~~~



> 访问：Http://localhost/student-service/user/find/all 
>
> student-service = zuul 配置的app-url-prefix  用来定位应用的
>
> user/find/all 是student-service的api地址

## 整合Eureka

1. 引用eureka依赖

2. 激活服务注册，发现客户端

   ~~~java
   @SpringBootApplication
   @EnableDiscoveryClient  //启动服务注册，发现 
   public class Application{
       public static void main(Stirng args[]){
           SpringApplication.run(Application.class,args);
       }
   }
   ~~~

3. 配置eureka发现的属性

   ~~~properties
   eureka.client.serviceUrl.defaultZone = \
   	http://localhost:12345/eureka
   ~~~


## 整合Hystrix

1. 添加依赖
2. 激活Hystrix
3. 配置熔断方法

## 整合Feign

spring zuul -> student-client->student-service-provider

1. 
2. Zuul 配置 
   1. 配置网关地址到student-client

## 整合Config Server

1. 添加eureka 依赖

2. 把config server 注册到注册中心上

   ~~~properties
   eureka.client.serviceUrl.defaultZone = \
   	http://localhost:12345/eureka
   ~~~

3. 激活注册中心的客户端

   ```java
   @SpringBootApplication
   @EnableConfigServer
   @EnableDiscoveryClient  //启动服务注册，发现 
   public class Application{
       public static void main(Stirng args[]){
           SpringApplication.run(Application.class,args);
       }
   }
   ```

4. 修改config Server工程中的配置Git目录的地址

    **user.dir** : 当前项目的路径  ${user.dir}/src/main/resources/configs

5. 为zull工程中添加配置文件

   在resources/configs中添加三个配置文件

   + zuul.properties
   + zuul-test.properties
   + zuul-prod.properties

6. config目录下操作(**为了让spring cloud 识别git仓库，否则添加以上三个文件也没有效果**)
   1. git init 初始化git 本地仓库 `git init`
   2. git add添加配置文件到git本地仓库 `git add *.properties`
   3. git commit 提交配置文件到git本地仓库 `git commit -m "properties init"`

 

## 调整zuul网关服务

**通过eureka注册中心连接config server**

1. 添加spring-cloud-config的依赖

2. 创建bootstrap.properties 

3. 添加配置项 config-client 客户端信息

4. 这里连接config-server的方式要改变（**变成Discovery client的方式**）

   ~~~properties
   ###服务名
   spring.application.name = spring-cloud-zuul
   ###服务端口
   server.port=8080
   ### 注册到注册中心
   eureka.client.serviceUrl.defaultZone = \
   	http://localhost:12345/eureka
   
   ### 配置客户端的名称（application)
   spring.cloud.config.name=zuul
   ### 激活的配置
   spring.cloud.config.profile= prod
   ### git里面指的是分支的名称
   spring.cloud.config.label= master
   ### 激活discovery 连接配置项
   spring.cloud.config.discovery.enable = true 
   ### 获取eureka注册中心上的application的名称
   spring.cloud.config.discovery.serverId = config-server
   
   ~~~

5. **注意！！！！**注册 到注册中心的时候 一定要放在`bootstrap.properties` 里面

   ~~~properties
   ## bootstrap.properties文件中           
   eureka.client.serviceUrl.defaultZone = \
   	http://localhost:12345/eureka
   ~~~

   原因： 因为bootstrap.application是在application.application之前执行的。如果放在application.properties里面的话，则会迟于bootstrap.application加载，那么zuul服务去eureka注册中心寻找服务的时候，是找不到的。





问答部分：

 1. Zuul中`RequestContext`已经存在了`ThreadLocal`中了，那么为什么还要继承`ConcurrentHashMap`?

    答： `ThreadLocal` 只能管理当前线程，不能管理子线程 ，子线程需要使用`InheritableThreadLocal`。

    ​	再java 中，main就是一个主线程，main也在一个main组中(group：main) 

    ​	当main中创建的线程就为子线程,main就是所有线程的父线程。



    ​	

