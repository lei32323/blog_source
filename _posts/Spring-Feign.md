---
title: Spring cloud  Feign
date: 2018-12-11 22:07:36
tags: spring cloud
---



APT

[Spring cloud官方中文文档](http://ifeve.com/%E3%80%8Aspring-cloud-netflix%E5%AE%98%E6%96%B9%E6%96%87%E6%A1%A3%E3%80%8B7-%E5%A3%B0%E6%98%8E%E5%BC%8F-rest-%E5%AE%A2%E6%88%B7%E7%AB%AF%EF%BC%9A-feign/)

# Feign

## 申明式服务客户端Feign 基本使用

申明式：接口申明，Annotation驱动

Web服务：HTTP的方式作为通讯协议

客户端：用于服务调用的存根

Feign：原生并不是Spring Web Mvc的实现，基于JAX-RS(java Rest规范) 实现。Spring Cloud封装了Feign，使其支持Spring Web MVC。`RestTemplate` ,`HttpMessageConverter`。

> `RestTemplate` 以及Spring web mvc 可以显示的自定义`HttpMessageConverter` 实现



假设：有一个java接口`PresonServer`,Feign 可以将其声明它是以HTTP方式调用的。



需要的组件（SOA)：

### Eureka Server:服务发现和注册 

1. 创建`StringBootApplication`

   ```java
   @SpringBootApplication
   @EnableEurekaServer
   public class SpringCloudFeignEurekaApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(SpringCloudFeignEurekaApplication.class, args);
   	}
   }				
   ```

2. 配置application.properties

   ```properties
   spring.application.name= studnet-eureka
   
   server.port=12345
   
   ### 不进行自我注册
   eureka.client.fetch-registry=false
   ### 不进行注册到eureka
   eureka.client.register-with-eureka=false
   ```

### Feign 声明接口（契约）：定义一种Java强类型接口

```java
@FeignClient(value = "student-service") //服务提供方应用名称
public interface StudentService {

    /**
     * 存储学生信息{@link Student}
     * @param student 学生实体
     * @return 成功<code>true</code> 失败<code>false</code>
     */
    @RequestMapping("/student/save")
    boolean save(Student student);


    /**
     * 获取学生信息 {@link Collection<Student>}
     * @return 不会返回null
     */
    @GetMapping("/student/find/all")
    Collection<Student> findAll();

}
```



### Fegin客户（服务消费）端：调用Fegin申明接口

1. 创建`SpringBootApplication`

```java
@SpringBootApplication
@EnableEurekaClient //eureka客户端
@EnableFeignClients(clients = StudentService.class) //feign api接口
public class StudentClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(StudentClientApplication.class,args);
    }
}
```

2. 配置application.properties

```properties
spring.application.name= student-client

server.port=8080

eureka.client.service-url.defaultZone = http://localhost:1234/eureka

management.security.enabled=false
```

3. 编写`StudentClientController`

```java
@RestController
public class StudentClientController implements StudentService {//可以不实现

    private final StudentService studentService;

    @Autowired
    public StudentClientController(StudentService studentService) {
        this.studentService = studentService;
    }

    @Override
    public boolean save(@RequestBody Student student) {
        return studentService.save(student);
    }

    @Override
    public Collection<Student> findAll() {
        return studentService.findAll();
    }
}
```

**不实现，看下方例子**

~~~java
@RestController
public class UserClientController {

    private final UserService userService;

    public UserClientController(UserService userService) {
        this.userService = userService;
    }
    /**
     * 保存用户 {@link User}
     * @param user
     * @return 成功<code>true</code> 失败<code>false</code>
     */
    @PostMapping("/user/save")
   public boolean save(@RequestBody User user){
        return userService.save(user);
    }


    /**
     * 查询所有的用户{@link Collection <User> }
     * @return  不会返回Null
     */
    @GetMapping("/user/find/all")
    public Collection<User> findAll(){
       return  userService.findAll();
    }
}

~~~



### Feign服务（服务提供）端：**不一定强制实现Fegin申明接口**

1.  创建`SpringBootApplication`

~~~java
@SpringBootApplication
@EnableEurekaClient
public class StudentProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(StudentProviderApplication.class,args);
    }
}

~~~

2. 创建application.properties

```properties
spring.application.name= student-service

server.port=9090

eureka.client.service-url.defaultZone = http://localhost:1234/eureka

management.security.enabled=false
```

3. 编写`StudentServiceProvider`

~~~java
/**
 * 实现student的类
 */
@RestController
public class StudentServiceProvider {

    private static  Map<Long,Student> stus = new ConcurrentHashMap<Long,Student>();

    /**
     * 存储学生信息{@link Student}
     * @param student 学生实体
     * @return 成功<code>true</code> 失败<code>false</code>
     */
    @RequestMapping("/student/save")
    boolean save(@RequestBody Student student){
        return stus.putIfAbsent(student.getId(),student)==null;
    }


    /**
     * 获取学生信息 {@link Collection <Student>}
     * @return 不会返回null
     */
    @GetMapping("/student/find/all")
    Collection<Student> findAll(){
        return stus.values();
    }


}
~~~



> Fegin客户（服务消费）端 和Feign服务（服务提供）端以及Feign 声明接口（契约） 存放再同一个工程



**调用过程：**

​	 student-service和student-client项目都注册到eureka服务端。

​	student-client 可以感知student-service应用的存在，并且Spring Cloud 帮助解析`StudentService` 中声明的应用名称："student-provider",因此"student-client" 的服务被调用的时候，路由到了"student-provider"的URL上

## Ribbon整合  

### 关闭Eureka 注册 （注意：`eureka.client.service-url.defaultZone` 还是要写的）

#### 方式一：调整 Student-client 关闭Eureka 

~~~properties
###Ribbon不适用Eureka注册
ribbon.eureka.enabled=false
~~~

#### 方式二：注释`@EnableEurekaClient`

~~~java
@SpringBootApplication
//@EnableEurekaClient 注释服务器端的注册
@EnableFeignClients(clients = StudentService.class) //feign api接口
public class StudentClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(StudentClientApplication.class,args);
    }
}

~~~

#### 定义服务ribbon的服务列表（服务名称：student-service)

~~~properties
### 定义“student-service”负载均衡的服务列表
student-service.ribbon.listOfServers=\
  http://localhost:9090,http://localhost:9090
~~~

### 自定义Ribbon规则

#### 接口和Netflix内部实现

+ IRule

  + 随机规则：RandomRule
  + 最可用规则：BestAvailableRule
  + 轮训规则：RoundRobinRule
  + 重试规则：RetryRule
  + 客户端配置：ClientConfigEnabledRoundRobinRule
  + 可用性过滤规则：AvailabilityFilteringRule
  + RT权重规则：WeightedResponseTimeRule
  + 规避区域规则：ZoneAvoidanceRule

##### 实现IRule (客户端实现)

1.  实现IRule

~~~java
package com.feign.student.ribbon.rule;

import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AbstractLoadBalancerRule;
import com.netflix.loadbalancer.ILoadBalancer;
import com.netflix.loadbalancer.IRule;
import com.netflix.loadbalancer.Server;

import java.util.List;

/**
 * 自定义一个随机规则的负载均衡
 */
public class MyRule extends AbstractLoadBalancerRule {

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {

    }

    @Override
    public Server choose(Object key) {

        ILoadBalancer iLoadBalancer = getLoadBalancer();

        List<Server> allList = iLoadBalancer.getAllServers();

        return allList.get(0);
    }
}
~~~



2. 暴露自定义的Rule为Spring 的Bean

   ~~~java
   @Bean
   public MyRule getRule(){
      return new MyRule();
   }
   
   ~~~

3. 激活配置

   ~~~java
   package com.feign.student;
   
   
   import com.feign.student.api.StudentService;
   import com.feign.student.ribbon.rule.MyRule;
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
   import org.springframework.cloud.netflix.feign.EnableFeignClients;
   import org.springframework.cloud.netflix.ribbon.RibbonClient;
   import org.springframework.cloud.netflix.ribbon.RibbonClients;
   import org.springframework.context.annotation.Bean;
   
   @SpringBootApplication
   //@EnableEurekaClient //注释服务器端的注册
   @RibbonClient(value = "student-service",configuration = StudentClientApplication.class) //设置服务提供名称和配置信息。
   @EnableFeignClients(clients = StudentService.class) //feign api接口
   public class StudentClientApplication {
       public static void main(String[] args) {
           SpringApplication.run(StudentClientApplication.class,args);
       }
   
   
       @Bean
       public MyRule getRule(){
           return new MyRule();
       }
   }
   
   ~~~

>@RibbonClient(value = "student-service",configuration = StudentClientApplication.class)
>
>configuration = StudentClientApplication.class 当自定义配置类的时候添加这个，这里已经加在Application中了，所以不需要重复引用

## 整合Netflix Hystrix (和feign api客户端添加)

1. 创建hystrix类 

   ~~~java
   public class MyHystrix implements UserService{ //最好事先api接口，安全
   
       @Override
       public boolean save(User user) {
           return false;
       }
   
       @Override
       public Collection<User> findAll() {
           return Collections.emptyList();
       }
   }
   
   ~~~

2. 引用Hystrix

   ~~~java
   @FeignClient(value = "user-provider",fallback = MyHystrix.class) //fallback 信息
   public interface UserService {
   
       /**
        * 保存用户 {@link User}
        * @param user
        * @return 成功<code>true</code> 失败<code>false</code>
        */
       @PostMapping("/user/save")
       boolean save(@RequestBody User user);
   
   
       /**
        * 查询所有的用户{@link Collection<User> }
        * @return  不会返回Null
        */
       @GetMapping("/user/find/all")
       Collection<User> findAll();
   
   }
   
   ~~~

## FeignClient 手动装配

1. 注释掉`@FeignClient`

   ~~~java
   //@FeignClient(value = "user-provider",fallback = MyHystrix.class)
   public interface UserService {
   
       /**
        * 保存用户 {@link User}
        * @param user
        * @return 成功<code>true</code> 失败<code>false</code>
        */
       @PostMapping("/user/save")
       boolean save(@RequestBody User user);
   
   
       /**
        * 查询所有的用户{@link Collection<User> }
        * @return  不会返回Null
        */
       @GetMapping("/user/find/all")
       Collection<User> findAll();
   
   }
   ~~~

2. 在`UserClientControllerManually`添加装配Feign Client信息

   ~~~java
   package com.wl.cloud.feign.controller;
   
   
   import com.wl.cloud.feign.api.UserService;
   import com.wl.cloud.feign.domain.User;
   import feign.Client;
   import feign.Contract;
   import feign.Feign;
   import feign.auth.BasicAuthRequestInterceptor;
   import feign.codec.Decoder;
   import feign.codec.Encoder;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.cloud.netflix.feign.FeignClientsConfiguration;
   import org.springframework.context.annotation.Import;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.PostMapping;
   import org.springframework.web.bind.annotation.RequestBody;
   import org.springframework.web.bind.annotation.RestController;
   
   import java.util.Collection;
   
   
   @Import(FeignClientsConfiguration.class)//Spring Cloud为Feign默认提供的配置类
   @RestController
   public class UserClientControllerManually {
   
       private final UserService userService;
   
       @Autowired
       public UserClientControllerManually(Decoder decoder, Encoder encoder, Client client, Contract contract){
           this.userService = Feign.builder().client(client).encoder(encoder).decoder(decoder).contract(contract)
                   .requestInterceptor(new BasicAuthRequestInterceptor("user", "password1"))
                   .target(UserService.class, "http://user-provider/");
       }
   
       /**
        * 保存用户 {@link User}
        * @param user
        * @return 成功<code>true</code> 失败<code>false</code>
        */
       @PostMapping("/user/save")
       public boolean save(@RequestBody User user){
           return userService.save(user);
       }
   
   
       /**
        * 查询所有的用户{@link Collection <User> }
        * @return  不会返回Null
        */
       @GetMapping("/user/find/all")
       public Collection<User> findAll(){
           return  userService.findAll();
       }
   }
   
   ~~~

**注意**   ！！！！！！！！！！！！！！！ 一定要加入这个，因为默认是false

```properties
feign.hystrix.enabled =true
```

## `@Primary` 使用

​	当使用Feign 的 Hystrix回调方法时，有许多相同类型的beans存在于应用上下文中，这会导致@Autowired注解由于找不到具体的bean或主要的bean导致不能注入，为了解决这个问题，Spring Cloud Netflix 标记所有Feign实例bean为@Priamry类型，所以spring框架可以正确注入bean，有些情况下这并不需要，可以通过在@FeignClient注解里设置primary属性为off来关闭它

```properties
@FeignClient(value = "user-service",fallback = UserServiceControllerHystrix.class,primary = false)
```

## Feign 客户的支持继承

1. 编写BaseService   通用的函数编写

~~~java
package com.wl.feign.service;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

public interface BaseService {

    @RequestMapping("/user/{id}")
    boolean findById(@PathVariable("id") Long id);

}

~~~

2. 继承BaseService   `public interface UserService extends BaseService`

~~~java

/**
 * 用户API（Feign Client）
 */
@FeignClient(value = "user-service",fallback = UserServiceControllerHystrix.class,primary = false) //激活feignclient 从eureka上关联provider
public interface UserService extends BaseService{

    /**
     * 保存用户的信息 {@link User}
     * @param user
     * @return 成功<code>true</code> <code>false</code>
     */
    @PostMapping("/user/save")
    boolean saveUser(@RequestBody  User user);


    /**
     * 获取用户信息 {@link Collection<User>}
     * @return 不会返回null
     */
    @GetMapping("/user/find/all")
    @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value ="500")
    Collection<User> findAll() throws InterruptedException;

}
~~~

## Feign 请求/响应体 压缩

1. 开启对Feign请求的请求体或响应体进行GZIP压缩功能

~~~properties
feign.compression.request.enabled=true
feign.compression.response.enabled=true
~~~

2. 请求的压缩设置

~~~properties
### 开启压缩
feign.compression.request.enabled=true
### 压缩文件类型
feign.compression.request.mime-types=text/xml,application/xml,application/json
### 最小的文件请求长度
feign.compression.request.min-request-size=2048
~~~

## Feign 日志

每创建一个Feign客户端，就会创建一个Logger ，默认的Logger的名字为接口的全类名，feign默认记录DEBUG级别的日志，可以自定义

**方式1**  配置文件的形式

~~~properties
### 设置客户端的日志级别 user 为模块 名称，UserClient为客户端
logging.level.project.user.UserClient: DEBUG
~~~

- NONE, 不记录(默认)
- BASIC, 仅记录请求的方法和地址以及响应的状态码和执行时间
- HEADERS, 单独的记录请求和响应头的信息
- FULL, 记录请求和响应的请求头，请求体和元数据

**方式2** 代码的形式

~~~java
@Configuration
public class MyConfig {

    @Bean
    public MyLoadBalancerRule getMyLoadBalancerRule(){
        return new MyLoadBalancerRule();
    }

    //设置Feign Client的日志级别
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
     }

}
~~~

**注意**： 每个客户端可以不同的配置

## 问答互动

