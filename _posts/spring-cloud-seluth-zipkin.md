---
title: Spring Cloud Seluth -Zipkin
date: 2018-12-13 22:33:36
tags: spring cloud
---



# Spring Cloud Seluth -Zipkin

Spring Cloud Sleuth是为Spring Cloud实现了分布式链路追踪解决方案。

**前提：**

​	SpringCloud解决方案下，存在两个子项目,并在一个项目中使用RestTemplate或者Feign等方法调用另外一个项目中的接口

## 具体流程

### 日志发生的变化

​	当应用ClassPath 下存在org.springframework.cloud:spring-cloud-starter-sleuth时候，日志会发生调整。

> 当系统自动打印的日志会出现：`INFO [user-client-provider,,,] `
>
> 手动打印的时候出现：`INFO [user-client-provider,bc3ce5f774d7cc5e,645296b5dc1edf21,false]`
>
> [服务名称，调用链，被调用链，是否应将日志导出到Zipkin]

### 整体流程

​	spring-cloud-starter-sleuth 会自动装配一个名为TraceFilter 组件（在 Spring WebMVC DispatcherServlet之前），它会增加一些 slf4j MDC

> MDC : Mapped Diagnostic Context

`org.springframework.cloud:spring-cloud-starter-sleuth` 会调整当前日志系统（slf4j）的 MDC `Slf4jSpanLogger#logContinuedSpan(Span)`:

~~~java
@Override
public void logContinuedSpan(Span span) {
    MDC.put(Span.SPAN_ID_NAME, Span.idToHex(span.getSpanId()));
    MDC.put(Span.TRACE_ID_NAME, span.traceIdString());
    MDC.put(Span.SPAN_EXPORT_NAME, String.valueOf(span.isExportable()));
    setParentIdIfPresent(span);
    log("Continued span: {}", span);
}

~~~

## 整合 zipkin  HTTP上报

### 服务端

1. 导入依赖包

   ```xml
   <!-- zipkin服务端依赖 begin-->
   <dependency>
       <groupId>io.zipkin.java</groupId>
       <artifactId>zipkin-server</artifactId>
   </dependency>
   
   <dependency>
       <groupId>io.zipkin.java</groupId>
       <artifactId>zipkin-autoconfigure-ui</artifactId>
   </dependency>
   <!-- zipkin服务端依赖  end-->
   ```

2. 激活zipkin服务端

```java
   @SpringBootApplication
   @EnableZipkinServer// 激活zipkin
   @EnableDiscoveryClient
   public class LiangduSleuthApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(LiangduSleuthApplication.class, args);
   	}
   }
```

1. 注册到注册中心

   ```properties
   ## 服务名称
   spring.application.name=liangdu-seluth
   ## 端口
   server.port=23456
   ## 注册中心地址
   eureka.client.service-url.defaultZone = http://localhost:12345/eureka
   ## 取消权限
   management.security.enabled=false
   ## 收取的维度 默认是10%，即0.1
   spring.sleuth.sampler.percentage = 1
   ## log4f的日志样式
   logging.pattern.level=%5p [${spring.zipkin.service.name:${spring.application.name:-}},%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-Span-Export:-}]
   ```

2. 访问url 

   `http://localhost:23456/zipkin/` 

### logger

想要被zipkin 关注到的服务，需要使用Log日志输出

```java
public Logger logger = LoggerFactory.getLogger(getClass());
```



### 客户端

1. 添加依赖包

   ```xml
   <!-- zipkin 推送消息 begin-->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-zipkin</artifactId>
   </dependency>
   
   
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-sleuth</artifactId>
   </dependency>
   
   <!-- zipkin 推送消息 end-->
   ```

2. 添加配置文件

   ```properties
   ## 添加zipkin配置
   zipkin.port =23456
   zipkin.service.name=liangdu-seluth
   spring.zipkin.base-url=http://${zipkin.service.name}:${zipkin.port}/
   ## 收取的维度 默认是10%，即0.1
   spring.sleuth.sampler.percentage = 1
   ## log4f的日志样式
   logging.pattern.level=%5p [${spring.zipkin.service.name:${spring.application.name:-}},%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-Span-Export:-}]
   ```

3. 需要被监控的服务要添加日志输出

   ```java
   private Logger logger = LoggerFactory.getLogger(getClass());
   
   @Override
   @PostMapping("/user/save")
   public boolean saveUser(@RequestBody User user) {
       logger.info("user-client UserClientController#saveUser");
       return users.putIfAbsent(user.getId(),user)==null;
   }
   ```

### HTTP负载均衡

[HTTP负载均衡](https://cloud.spring.io/spring-cloud-static/Edgware.SR5/single/spring-cloud.html#_only_sleuth_log_correlation)

### 主机定位器

[主机定位](https://cloud.spring.io/spring-cloud-static/Edgware.SR5/single/spring-cloud.html#_only_sleuth_log_correlation)

##  整合 zipkin   Spring Cloud Stream 收集（消息）上报

### 服务端

1. 添加依赖

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>

<!-- 使用zipkin关联stream收集消息-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin-stream</artifactId>
</dependency>

<!--使用kafka作为stream消息通道-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-kafka</artifactId>
</dependency>


<!--推送消息到kafka，服务端可以不需要添加-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-stream</artifactId>
</dependency>
~~~

2. 改变Application.class启动类

   ~~~java
   @SpringBootApplication
   @EnableZipkinStreamServer// 激活zipkin
   @EnableDiscoveryClient
   public class LiangduSleuthApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(LiangduSleuthApplication.class, args);
   	}
   }
   ~~~

3. application.properties配置

   ~~~properties
   ## 设置zk的连接
   spring.cloud.stream.kafka.binder.zk-nodes =192.168.83.130:2181
   ## 设置kafka的连接
   spring.kafka.bootstrap-servers=192.168.83.129:9092
   ## 收取的维度 默认是10%，即0.1
   spring.sleuth.sampler.percentage = 1
   ## log4f的日志样式
   logging.pattern.level=%5p [${spring.zipkin.service.name:${spring.application.name:-}},%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-Span-Export:-}]
   ~~~

### 客户端

1. 依赖配置

   ~~~xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-sleuth</artifactId>
   </dependency>
   <!--推送消息到kafka-->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-sleuth-stream</artifactId>
   </dependency>
   <!--使用kafka作为stream消息通道-->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-stream-binder-kafka</artifactId>
   </dependency>
   ~~~

2. 添加Log日志

   ~~~java
   private Logger logger = LoggerFactory.getLogger(getClass());
   
   @Override
   @PostMapping("/user/save")
   public boolean saveUser(@RequestBody User user) {
       logger.info("user-client UserClientController#saveUser");
       return users.putIfAbsent(user.getId(),user)==null;
   }
   ~~~

3. 配置Properties

   ~~~properties
   ## 设置zk的连接
   spring.cloud.stream.kafka.binder.zk-nodes =192.168.83.130:2181
   ## 设置kafka的连接
   spring.kafka.bootstrap-servers=192.168.83.129:9092
   ## 收取的维度 默认是10%，即0.1
   spring.sleuth.sampler.percentage = 1
   ## log4f的日志样式
   logging.pattern.level=%5p [${spring.zipkin.service.name:${spring.application.name:-}},%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-Span-Export:-}]
   ~~~

# ELK

![_](https://yqfile.alicdn.com/2a7721fc3dbe3fdf08ebb202f9f9faf058e396f6.jpeg)

## 整合Elasticsearch 做数据存储和搜索

### 安装

[下载地址](https://www.elastic.co/cn/downloads)

### 启动

`nohup bin/elasticsearch > logs/elasticsearch.out & `

#### 启动报错

1. max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]


   max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

   答： 在`sudo vi /etc/security/limits.conf `添加信息

```bash
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
```

2.  max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

  答：在`sudo vi /etc/sysctl.conf` 添加信息

```properties
vm.max_map_count = 655360
```

​	执行`sudo sysctl -p`命令

[具体错误查看](https://www.jianshu.com/p/365db8b181cc)

### 验证启动

因为ElasticSearch默认是本地访问，所以需要开通其他服务器访问的权限

#### 允许外部9200访问

修改配置文件 config/elasticsearch.yml

~~~properties
network.host: 0.0.0.0
~~~

#### URL验证

http://192.168.83.129:9200/

~~~json
{
    name: "IJSh7YS",
    cluster_name: "elasticsearch",
    cluster_uuid: "TczS3U7gS22qhjUzmSlxjA",
    version: {
        number: "6.5.3",
        build_flavor: "default",
        build_type: "tar",
        build_hash: "159a78a",
        build_date: "2018-12-06T20:11:28.826501Z",
        build_snapshot: false,
        lucene_version: "7.5.0",
        minimum_wire_compatibility_version: "5.6.0",
        minimum_index_compatibility_version: "5.0.0"
    },
    tagline: "You Know, for Search"
}
~~~

#### 查看索引

`http://localhost:9200/_cat/indices?v`

##### 查看索引的内容

`http://localhost:9200/kibana_sample_data_ecommerce/_search?pretty`

## 整合logstash 收集数据 绑定到ElasticSearch

### 安装

[下载地址](https://www.elastic.co/cn/downloads)

### 启动

#### 修改配置文件

 1. 把config 文件中的`logstash-sample.conf` 复制到根目录为`core.conf`

    ~~~bash
    cp logstash-sample.conf ../core.conf
    ~~~

	2. 修改core.conf

    ~~~bash
    input {
       kafka {
            id => "my_plugin_id"
            bootstrap_servers => "localhost:9092"
            ## kafka topic名称
            topics => ["sleuth"] 
            auto_offset_reset => "latest"
          }
    }
    
    filter {
    	grok {
    		  ## patterns_dir 用于指定自定义的分析规则，在该文件下建立文件配置验证的正则规则
              patterns_dir => ["/home/mq/logstash-6.5.3/patterns"]
              match => { "message" => "%{WORD:module} \| %{LOGBACKTIME:timestamp} \| %{LOGLEVEL:level} \| %{JAVACLASS:class} - %{JAVALOGMESSAGE:logmessage}" }
              }
        }
        
    output {
      stdout { codec => rubydebug }
      elasticsearch {
        hosts => ["http://localhost:9200"]
        index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        #user => "elastic"
        #password => "changeme"
      }
    }
    ~~~

[kafka input具体配置参数](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html?spm=a2c4e.11153940.blogcont645316.23.56404122KbJDjs)

[kafka output具体配置参数](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-kafka.html)

[grok具体配置参数](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html?spm=a2c4e.11153940.blogcont645316.24.56404122KbJDjs)

基本语法：

~~~bash
input {
  ...#读取数据，logstash已提供非常多的插件，可以从file、redis、syslog等读取数据
}
 
filter{
  ...#想要从不规则的日志中提取关注的数据，就需要在这里处理。常用的有grok、mutate等
}
 
output{
  ...#输出数据，将上面处理后的数据输出到file、elasticsearch等
}
~~~



#### 启动

~~~bash
nohup bin/logstash -f core.conf --config.reload.automatic  &
~~~

+ --config.reload.automatic  自动刷新配置文件
+ -f  配置文件指定

## 整合 kibana  展示ElasticSearch数据的图形界面

### 安装

[下载地址](https://www.elastic.co/cn/downloads)

### 启动

#### 修改配置文件

​	1. 默认是localhsot，需要修改为其他服务器可以访问

​		打开`config/kibana.yml`

​		`server.host: "0.0.0.0"`

#### 启动

​	`nohup bin/kibana > bin/kibana.out &`

#### 验证

​	访问URL`http://192.168.83.129:5601`