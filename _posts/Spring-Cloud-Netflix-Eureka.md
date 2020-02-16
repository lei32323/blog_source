---
title: Spring Cloud Netflix Eureka
date: 2018-12-16 20:51:36
tags: spring cloud
---





## Spring Cloud Netflix Eureka

**高可用，并不是高一致性**

### 传统的服务治理

XML -PRC -> XML 方法描述、方法参数->WSDL(WebService定义语言)  

WebService -> Soap (HTTP、SMTP) ->文本协议（头部分，体部分）

REST-> JSON/XML(Schema:类型、结构)->文本协议（HTTP header 、Body)

W3C Schema :xsd:string 原子类型，自定义自由组合原子类型

JAVA POJO ：Int、String 

Response Header -> Content-type :application/json;charset=utf-8

Dubbo :Hession、java Serialization(二进制)

> 二进制缺点：跨语言不变，一般通过Client(java、c++) ，人员识别性差
>
> 二进制优点：性能非常好（字节流，免去字符流（字符编码），免去了字符解释，对机器友好，对人不友好）

> 序列化：把编程语言的数据结构转换成字节流
>
> 反序列化：字节流转换成编程语言的数据结构（原生类型的组合）



### 高可用架构

-消除单独点失败

-可靠性交迭

-故障的探测

#### Proxy 和Broker

- Proxy :一般性代理，路由
  - Nginx:反向代理
- Broker:包括路由，并且算法，老的称谓（MOM)
  - Message Broker:消息的路由，消息管理（消息是否可达）

#### 可用性比率计算

可用性比率：通过时间来计算的（一年或者一月）

比如：一年99.99% 

可用时间：365 * 24 * 3600 * 99.99%

不可用时间：365 * 24 * 3600 * 0.01% = 3153.6秒 < 一个小时

不可用时间：1个小时推算一年 1/24/365=0.01%



单台机器可用性比率：99%   ，不可用1%

用两台机器不可用率比率 ：1% * 1%

用N机器不可用率比率 ：1% ^ N



### 可靠性

微服务里面的问题：

A -> B -> C

99% -> 99% ->99% = 99%的三次方：97%     

A -> B -> C -> D

99% -> 99% ->99% ->99% = 99%的4次方：96%  

 

结论：增加机器可以提高可用性，增加服务调用会提高可靠性，同时也降低了可用性。





### 服务发现

#### 传统技术

- WebService 
  - UDDI
- Rest
  - HATEOAS
- JAVA
  - JMS
  - JNDI

### Eureke 服务器

application启动类`@EnableEurekaServer`

通常经验，Eureka服务器不需要开启自我注册，也不需要检索服务

```properties
###取消服务器的自我注册
eureka.client.register-with-eureka=false
###注册中心的服务器，没有必要再去检索服务
eureka.client.fetch-registry = false

###eureka Server服务URL，用于客户端注册和发现
eureka.client.service-url.defaultZone = http://localhost:${server.port}/eureka
```

但是这2个设置并不影响作为服务器的使用，不过建议关闭，为了减少不必要的异常堆栈，将减少错误的干扰（比如：系统的异常和业务异常）

> 设计方式：
>
> Fast Fail : 快速失败
>
> Fault-Tolerance：容错



**Replicas** 复制品，主从，防止eureke挂掉



### Eureke 客户端

application启动类：`@EnableDiscoveryClient`

**consumer**

```properties
Spring.application.name = .....
### Eureka 注册中心服务器端口
erueka.server.port = 9090
### 服务暴露端口
server.port = 8080
###eureka Server服务URL，用于客户端注册和发现
eureka.client.service-url.defaultZone = http://localhost:${erueka.server.port}/eureka
```

**Provider **

```properties
Spring.application.name = .....
### Eureka 注册中心服务器端口
erueka.server.port = 9090
### 服务暴露端口
server.port = 7070 
###eureka Server服务URL，用于客户端注册和发现
eureka.client.service-url.defaultZone = http://localhost:${erueka.server.port}/eureka
```

**RestTemplate**

`@LoadBalanced` 负载均衡

```java
@LoadBalanced//负载均衡
@Bean
public RestTemplate restTemplate(){
	return new RestTemplate();
}
```

**UserServiceProxy**

```java
private static final String EUREKA_REST_LOCAL = "http://userProvider/";

    @Autowired
    private RestTemplate restTemplate;

    @Override
    public boolean saveUser(User user) {
        return restTemplate.postForObject(EUREKA_REST_LOCAL+"/user/save",user,Boolean.class)!=null;
    }

    @Override
    public Collection<User> findAllUser() {

        return restTemplate.getForObject(EUREKA_REST_LOCAL+"/user/getAll",Collection.class);
    }
```





#### 问答部分

1. consul和Eureka是一样的吗？

   提供功能类似，consul功能更强大，广播式服务发现/注册

2. 重启eureka服务器，客户端的应用需要重启吗？

   不用重启，客户端在不停的上报信息，不过在eureka服务器启动过程中，客户端会大量报错

3. 客户上报的信息存储在哪里的，内存还是缓存？

   都是存储在内存中的，eureka并不是缓存所有的服务，缓存需要的服务。比如说：Eureka Server管理了200个应用，每个应用存在100个实例，总体管理20000个实例。客户端更知道自己需要的应用实例。

4. 一个业务中调用多个service时如何保证事务

   需要分布式事务实现（JTA分布式框架)，可是一般互联网项目，没有这种昂贵的操作



### Eureka客户端高可用

```
#### 高可用注册中心集群
```

​	只需要增加eureka服务注册URL：

​	如果Eureka 客户端应用配置多个Eureka注册服务器，那么默认情况只有一台可用的服务器(随机选择一台)存在注册信息

​	如果第一台**可用**的服务器宕掉了，那么Eureka客户端应用会选择下一台**可用**的Eureka服务器

​	范围端口: 

```properties
server.port=${random.int[7070,7079]}
```

源码：`EurekaClientConfigBean`

#### 获取注册时间间隔（Eureal客户端）

Eureka客户端需要获取eureka服务器上的注册信息，方便服务调用。

Eureka 客户端：`EurekaClient `                                  关联的应用集合：`Applications`

单个应用信息：`Application`,关联多个应用实例      单个应用实例：`InstanceInfo`

当Eureka客户端需要调用具体的某个服务时，比如`user-service-consumer`调用`user-service-provider`，`user-service-provider`实际对应对象是`Application`，关联了许多应用实例(`InstanceInfo`)

如果应用`user-service-provider` 的应用实例发生变化时，那么`user-service-consumer`是需要感知的，比如:`user-service-provider` 从10台降到了5台，那么作为调用方的`user-service-consumer`需要知道这个变化。可是这个变化过程，可能存在一定的延迟，可以通过调整注册时间间隔来调整。

配置项：

```properties
### 调整注册信息的获取周期，默认是30秒（当Eureka服务器出现调整后，5秒钟得到相应）
eureka.client.registryFetchIntervalSeconds = 5 //调整为5秒一次
```



#### 实例信息复制时间间隔（Eureal客户端）

具体的就是客户端信息的上报到Eureka服务器的时间。当 eureka客户端应用上报的频率越频繁，eureka的应用状态管理一致性就越高

**案例：** 当Eureka服务端停止，Eureka客户端启动的时候，会报错，然后吧Eureka服务端启动，Eureka客户端5秒就脸上了Eureka服务端

**配置项：**

```properties
### 调整客户端应用状态信息上报的周期
eureka.client.instanceInfoReplicationIntervalSeconds = 5 //调整为5秒一次
```

> Eureka 的应用信息同步方式：拉模式
>
> Eureka的应用信息上报的方式:推模式



#### 实例ID（Eureal客户端）

从Eureka Server Dashboard 里面可以看到具体某个应用中的实例信息，比如：

```
UP (1) - DESKTOP-APGDO5C:user-consumer:8080
```

其中他的命名模式：`${hostname}:${spring.application.name}:${server.port}`



实现类：`EurekaInstanceConfigBean`

** 配置项**

```properties
### Eureka 应用实例对应的ID
eureka.instance.instanceId = ${spring.application.name}:${server.port}
```

注意:instanceId不能随便改，改了对eureka 会多次存在，但是对应的会是同一个应用



#### 实例端口映射（Eureal客户端）

**源码位置：**`EurekaInstanceConfigBean` 

```java
private String statusPageUrlPath = /path
```

**配置项**

```properties
### Eureka 客户端引用实例状态 默认是/info
eureka.instance.statusPageUrlPath = /health   
```





### Eureka服务端高可用

#### eureka相互同步

1. Eureka服务端1配置项

2. ```properties
   ### 对外暴露的端口
   server.port = 9091
   ###取消服务器的自我注册
   eureka.client.register-with-eureka=true
   ###注册中心的服务器，没有必要再去检索服务
   eureka.client.fetch-registry = true
   ###eureka Server服务URL，用于客户端注册和发现
   eureka.clinet.serverUrl.defaultZone = http://localhost:9090/eureka
   ```

3. Eureka服务端2配置项

   ```properties
   ### 对外暴露的端口
   server.port = 9090
   ###取消服务器的自我注册
   eureka.client.register-with-eureka=true
   ###注册中心的服务器，没有必要再去检索服务
   eureka.client.fetch-registry = true
   ###eureka Server服务URL，用于客户端注册和发现
   eureka.clinet.serverUrl.defaultZone = http://localhost:9091/eureka
   ```

4. 启动命令行 通过`--spring.profiles.active=peer1`和`--spring.profiles.active=peer2`分别激活eureka server 1和eureka server 2

5. 启动eureka服务端

### RestTemplate



#### HTTP 消息装换器：HttpMessageConverter

自定义实现

编码问题

**切换序列化，反序列的协议**

#### HTTP Client 适配工厂 ：ClientHttpRequestFactory

实现的偏好

- Spring 实现
  - SimpleClientHttpRequestFactory
- HttpClient  高并发稳定
  - HttpComponentsClientHttpRequestFactory  
- OkHttpClient  性能好
  - OkHttp3ClientHttpRequestFactory
  - OkHttpClientHttpRequestFactory

举例说明：

```java
RestTemplate restTemplate = new RestTemplate(new HttpComponentsClientHttpRequestFactory())
```

**切换HTTP通讯实现，提升性能**

#### HTTP请求拦截器：ClientHttpRequestInterceptor

**加深RestTemplate拦截过程**

1. 重写函数

```java
@Component
public class MyInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        System.out.println(new String(body)+":"+request);
        return execution.execute(request,body);
    }
}
```

1. 添加到`RestTemplate`

```java
@LoadBalanced
@Bean
public RestTemplate restTemplate(){
     RestTemplate restTemplate = new RestTemplate(new HttpComponentsClientHttpRequestFactory());
     ArrayList<ClientHttpRequestInterceptor> clientHttpRequestInterceptors = new ArrayList<>();
     clientHttpRequestInterceptors.add(new MyInterceptor());
     restTemplate.setInterceptors(clientHttpRequestInterceptors);
     return restTemplate;
}
```





#### 问答部分

1. 为什么要用eureka？

   目前业界比较稳定云计算的开发员中间件，虽然有一些小毛病，基本上可用

2. Erueka主要功能为什么不能用浮动Ip代替呢？

   可以使用浮动IP，但是需要扩展

3. 使用的话，每个服务api都需要eurkae插件

   需要使用的eureka客户端

4. Spring cloud 日志收集 有什么解决方案

   一般用HBase、或者TSDB、ELK

5. Opentsdb 用作监控（写入速度贼快）

