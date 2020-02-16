
## Cloud Config Server

### 构建spring cloud文件

实现步骤

1. 在Configuration Class 标记`@EnableConfigServer`

2. 配置文件目录（基于Git)

   - liangdu.properties(默认)
   - liangdu-dev.properties(profile="dev")//开发环境
   - liangdu-test.properties(profile="test")//测试环境
   - liangdu-staging.properties(profile="staging")//仿真环境
   - liangdu-prod.properties(profile="prod")//生产环境

   ```properties
   liangdu.properties
   liangdu-dev.properties
   liangdu-test.properties
   liangdu-prod.properties
   ```

3. 服务端配置版本仓库(本地)

   ```properties
   spring.cloud.config.server.git.uri = /Users/wanglei/Development/git/cloud
   ### 存放在子目录的时候，需要检索文件夹
   spring.cloud.config.server.git.search-paths=*,zuul
   ```

1. 详细配置

```properties
###服务名
spring.application.name= config-server
###服务端口
server.port=9090
###本地git地址
spring.cloud.config.server.git.uri = /Users/wanglei/Development/git/cloud

###全局关闭actuator安全
#management.security.enabled=false
### 细腻度控制actuator安全 sensitive 关注敏感，安全
endpoints.env.sensitive=false
```



### Spring cloud 客户端配置

1. 创建客户端项目

2. ```properties
   ### bootstrap 上下文配置
   ### git上master分支上的liangdu-prod.properties
   ### 配置服务器
   spring.cloud.config.uri= http://localhost:9090/
   ### 配置客户端的名称（application)
   spring.cloud.config.name=liangdu
   ### 激活的配置
   spring.cloud.config.profile= prod
   ### git里面指的是分支的名称
   spring.cloud.config.label= master
   
   
   ###全局关闭actuator安全
   #management.security.enabled=false
   ### 细腻度控制actuator安全 sensitive 关注敏感，安全
   endpoints.env.sensitive=false
   ### 调用refresh 安全权限
   endpoints.refresh.sensitive=false
   ```

3. 这个时候修改config-server的配置信息，要调用客户端的refresh/refresh.json方法,则会刷新配置信息

   注意：一定要添加bootstrap.properties 信息，在bootstrap.properties中添加信息



### RefreshEndpoint  属性映射实体

用于修改阈值的更改，和开关

#### @RefreshScope 属性注入及时刷新

1. 在属性类上添加`@RefreshScope` 
2. 调用refresh/refresh.json方法

### ContextRefresh  管理上下文刷新

#### ·客户端·自定义自动刷新

```java
    private final ContextRefresher contextRefresher;

    private final Environment environment;

    @Autowired
    public SpringCloudConfigClientDemoApplication(ContextRefresher contextRefresher, Environment environment) {
        this.contextRefresher = contextRefresher;
        this.environment = environment;
    }


    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigClientDemoApplication.class, args);
    }

    //fixedDelay：上一次执行完成后间隔多少秒 10秒
    //initialDelay 第一次延迟多久执行 3秒
    @Scheduled(fixedDelay = 10 * 1000, initialDelay = 3 * 1000)
    public void refresherProperty() {
       Set<String> contextSet = contextRefresher.refresh();
        contextSet.forEach(propertyName->
                    System.err.printf("[线程:%s] key : %s,value : %s \n",
                    Thread.currentThread().getName(),
                    propertyName,
                    environment.getProperty(propertyName)
                    )
         );

        if (!contextSet.isEmpty()) {
            System.out.printf("[线程:%s]刷新了属性:\n%s", Thread.currentThread().getName(), contextSet);
        }
    }
```



### 健康检查

### /health GET 请求

```properties
###/health 健康检查
endpoints.health.sensitive=false
```

然后访问浏览器`/health `

##### 意义

可以任意的输出应用的各个(业务健康，系统监控）指标

实现类 `HelthEndpoint`

健康指示器 `HealthIndicator`

`HealthIndicator` :`HelthEndpoint ` 一对多的关系

### 健康指示器 HealthIndicator 

1. 添加架包

```xml
<dependency>
	<groupId>org.springframework.hateoas</groupId>
	<artifactId>spring-hateoas</artifactId>
</dependency>
```

1. 添加配置信息

```properties
### 细腻度控制actuator安全
endpoints.actuator.sensitive=false
```

1. 访问/actuator即可

   ```json
   {
       links: [
           {
           rel: "self",
           href: "http://localhost:8080/actuator"
           },
           {
           rel: "configprops",
           href: "http://localhost:8080/configprops"
           },
           {
           rel: "dump",
           href: "http://localhost:8080/dump"
           },
           {
           rel: "health",
           href: "http://localhost:8080/health"
           },
           {
           rel: "mappings",
           href: "http://localhost:8080/mappings"
           },
           {
           rel: "heapdump",
           href: "http://localhost:8080/heapdump"
           },
           {
           rel: "env",
           href: "http://localhost:8080/env"
           },
           {
           rel: "refresh",
           href: "http://localhost:8080/refresh"
           },
           {
           rel: "auditevents",
           href: "http://localhost:8080/auditevents"
           },
           {
           rel: "info",
           href: "http://localhost:8080/info"
           },
           {
           rel: "metrics",
           href: "http://localhost:8080/metrics"
           },
           {
           rel: "loggers",
           href: "http://localhost:8080/loggers"
           },
           {
           rel: "features",
           href: "http://localhost:8080/features"
           },
           {
           rel: "autoconfig",
           href: "http://localhost:8080/autoconfig"
           },
           {
           rel: "trace",
           href: "http://localhost:8080/trace"
           },
           {
           rel: "beans",
           href: "http://localhost:8080/beans"
           }
   	]
   }
   ```

### 自定义HealthIndicator健康检查

1. 实现`AbstractHealthIndicator`接口,重写`doHealthCheck`方法

   ```java
   public class MyHealthIndicator extends AbstractHealthIndicator {
       @Override
       protected void doHealthCheck(Health.Builder builder) throws Exception {
           builder.up().withDetail("result"," up up ");
       }
   }
   ```

2. 暴露自定义的健康检查为`Bean`

   ```java
   @Bean
   public MyHealthIndicator myHealthIndicator(){
       return new MyHealthIndicator();
   }
   ```

3. 关闭安全控制

   `management.security.enabled=false`

4. 访问 `/health`

   ```json
   {
       status: "UP",
       my: {
           status: "UP",
           result: " up up "
       },
       diskSpace: {
           status: "UP",
           total: 250685575168,
           free: 212232921088,
           threshold: 10485760
       },
       configServer: {
       	status: "UP",
     	  	propertySources: [
               "configClient",
               "/Users/wanglei/Development/git/cloud/liangdu-prod.properties",
               "/Users/wanglei/Development/git/cloud/liangdu.properties"
           ]
       }
   }
   ```

5. 注意：这是聚合的关系,有一个是down 那么status就是down，全是Up，那么status就是down

## 其他

jconsole 可以手动调用方法

Netty 基于NIO

NIO Reactor 同步非阻塞-多工处理

NIO Reactive = Reactor + Asyn = 异步非阻塞-多工处理

Netty 类似 Reactive， 观察者模式的实现

