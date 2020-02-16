# Spring Cloud Stream

## kafka

### kafka 和activeMQ区别

规范更严格代表性能越不好，但是都是支持高并发的

+ ActiveMQ：JMS (java message service) 规范
+ RabbitMQ：AMQP(Advanced Message Queue Protocol) 实现 （规范更严格）
+ Kafka：并非是某种规范实现，它灵活和性能相对是优势

CAP C 一致性 A 可用性  P 分区容错性







## Spring kafka

### 使用Kafka标准的API

~~~java
public static void producer()  {

    Properties properties = new Properties();
    properties.put("bootstrap.servers","192.168.83.129:9092");
    properties.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");
    properties.put("value.serializer","org.apache.kafka.common.serialization.StringSerializer");
    //0消息发送给borker ，不需要确认，性能高，但是容易丢失数据
    //1.只需要获取kafka集群中leader节点的确认即可返回，缺点:性能高，但是数据容易丢失
    //all(-1) 需要ISR中的所有节点进行确认（集群中的节点进行确认），最安全，但是性能比较低。当isr中只有一个节点的时候，也会出现丢失数据的情况
    //		properties.put(ProducerConfig.ACKS_CONFIG,"-1");

    //设置了大小后，会当达到一定的大小后才会去提推送消息到borker
    //		properties.put(ProducerConfig.BATCH_SIZE_CONFIG,16);
    //两者配合使用，延迟几S进行推送消息到borker上
    //		properties.put(ProducerConfig.LINGER_MS_CONFIG,30);

    //发送请求的大小（默认是1M）
    //		properties.put(ProducerConfig.MAX_REQUEST_SIZE_CONFIG,1);

    //推送消息 错误的时候，重试次数
    //		properties.put(ProducerConfig.RETRIES_CONFIG,1);
    //失败后，间隔多久进行重试
    //		properties.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG,1);

    KafkaProducer<String,Object> producer = new KafkaProducer<String, Object>(properties);

    //发送消息
    Future future = producer.send(new ProducerRecord("liangdu", "张三"));

    //强制执行
    try {
        future.get();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
}
~~~





## Spring Boot Kafka

### 设计模式

Spring 社区对data(spring-data)操作，有一个基本模式，Template模式

+ JDBC ：`JdbcTemplate`
+ Readis：`ReadisTemplate`
+ Kafka：`KafkaTemplate`
+ JMS：`JmsKafkaTemplate`
+ Rest：`RestTemplate`

>XXXTemplate一定实现了XXXOpeations
>
>KafkaTemplate一定实现了KafkaOpeations(接口)

### Maven 依赖

~~~xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
~~~



### 自动装配器`KafkaAutoConfiguration`

自动装配`KafkaTemplate`

~~~java
@Bean
@ConditionalOnMissingBean(KafkaTemplate.class)
public KafkaTemplate<?, ?> kafkaTemplate(
    ProducerFactory<Object, Object> kafkaProducerFactory,
    ProducerListener<Object, Object> kafkaProducerListener) {
    KafkaTemplate<Object, Object> kafkaTemplate = new KafkaTemplate<Object, Object>(
        kafkaProducerFactory);
    kafkaTemplate.setProducerListener(kafkaProducerListener);
    kafkaTemplate.setDefaultTopic(this.properties.getTemplate().getDefaultTopic());
    return kafkaTemplate;
}
~~~





### 属性配置类

`KafkaProperties`







### 消息发送者发送器

> 全局配置
>
> ~~~properties
> ## kafka主题
> kafka.topic = liangdu 
> ## Spring Kafka配置信息
> spring.kafka.bootstrapServers = localhost:9092
> ~~~

1. `application.properties`

   ~~~properties
   ## kafka 生产者配置
   spring.kafka.producer.key-serializer= org.apache.kafka.common.serialization.StringSerializer
   spring.kafka.producer.value-serializer= org.apache.kafka.common.serialization.StringSerializer
   ~~~

2. Java 

   ~~~java
   package com.wl.kafka.backdemo.com.kafka.producer;
   
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.kafka.core.KafkaTemplate;
   import org.springframework.stereotype.Component;
   
   @Component
   public class MyProducer {
   
       public final KafkaTemplate kafkaTemplate;
      
       private String topic;
       @Autowired
       public MyProducer(KafkaTemplate kafkaTemplate, @Value("${kafka.topic}") String topic) {
           this.kafkaTemplate = kafkaTemplate;
           this.topic = topic;
       }
   
       public void send(String message){
           kafkaTemplate.send(topic,message);
       }
   
   }
   ~~~


### 消息消费者消费器

1. `application.properties`

   ~~~properties
   ### kafka消费者配置
   spring.kafka.consumer.groupId = liangdu-1
   
   ~~~

2. java

   ~~~java
   package com.wl.kafka.backdemo.com.kafka.consumer;
   
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.kafka.annotation.KafkaListener;
   import org.springframework.kafka.core.KafkaTemplate;
   import org.springframework.stereotype.Component;
   
   @Component
   public class MyConsumer {
   
       //监听器的方式
       @KafkaListener(topics = "${kafka.topic}")
       public void getMessage(String message){
           System.out.println("消息："+message);
       }
   }
   ~~~



## Spring Cloud Stream

**相关项目：** spring Cloud Data Flow

**优点：** 不需要在意使用的是什么消息中间件，只需要关注业务

### 基本概念

**Source**：来源 ,近义词：**Producer**,**Publisher**

**Sink**： 接收器 ,近义词：**Consumer**,**Subscriber**

**Processor**：对于上流而言是Sink,对于下流而言是Source



> Reactive Streams:
>
> + Publisher
> + Subscribel
> + Processor

### Spring Cloud Stream Binder: Kafka



> **如果使用了外部的zookeeper 的话，需要配置对应的zookeeper地址（默认是本机的localhost:2181)**
>
>~~~properties
>spring.cloud.stream.kafka.binder.zk-nodes= 192.168.83.128:2181
>~~~



#### Source（OUTPUT） 使用

**源码**

~~~java
/*
 * Copyright 2015 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.cloud.stream.messaging;

import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;

/**
 * Bindable interface with one output channel.
 *
 * @see org.springframework.cloud.stream.annotation.EnableBinding
 * @author Dave Syer
 * @author Marius Bogoevici
 */
public interface Source {

	String OUTPUT = "output";

	@Output(Source.OUTPUT)
	MessageChannel output();

}

~~~



##### 原生channel用法

1. application.properoties 配置

~~~properties
## topic
kafka.topic = liangdu
### 绑定自定义通道
spring.cloud.stream.bindings.output.destination = ${kafka.topic}
~~~

2. channel调用

~~~java
@Component
@EnableBinding({Source.class})//绑定类
public class StreamProducer {

    @Autowired
    @Qualifier(Source.OUTPUT) // Bean 名称
    private MessageChannel messageChannel;
	
    @Autowired
    private Source source;

    public void send(String message){
        //方式1：通过消息管道发送消息 Payload
        messageChannel.send(MessageBuilder.withPayload(message).build());
        //方式2 ：payLoad + Headers
       messageChannel.send(MessageBuilder.createMessage(MessageBuilder.withPayload(message).build(),new MessageHeaders(new HashMap<>())));
    }

}
~~~



##### 自定义 channel

1. 编写自定义的channel

   ~~~java
   package com.wl.kafka.backdemo.com.kafka.stream.producer;
   
   import org.springframework.cloud.stream.annotation.Output;
   import org.springframework.cloud.stream.messaging.Source;
   import org.springframework.messaging.MessageChannel;
   
   public interface MySource {
       String NAME = "wanglei";
   
       @Output(NAME)//重点在这里，MessageChannel取了别名
       MessageChannel wanglei();
   
   }
   ~~~

2. 添加配置信息 

   ~~~properties
   ## topic
   kafka.topic = liangdu
   ### 绑定自定义通道
   spring.cloud.stream.bindings.wanglei.destination = ${kafka.topic}
   ~~~

3. **绑定自定义的channel接口!!!!!!!!!!!!!!!!!!!!!**

   ~~~java
   @EnableBinding({Source.class,MySource.class})//绑定类
   ~~~

4. 注入channel接口，两种方式

   ~~~java
   //注入的使用方式1
   @Autowired
   private MySource mySource;
   
   //注入的使用方式2
   @Autowired
   @Qualifier(MySource.NAME)  //NAME
   private MessageChannel myMessageChannel;
   ~~~

3. 调用接口

   ~~~java
   public void mySend(String message){
       //通过消息管道发送消息
       myMessageChannel.send(MessageBuilder.withPayload(message).build());
   }
   ~~~

   ~~~java
   package com.wl.kafka.backdemo.com.kafka.stream.producer;
   
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.beans.factory.annotation.Qualifier;
   import org.springframework.cloud.stream.annotation.EnableBinding;
   import org.springframework.cloud.stream.messaging.Source;
   import org.springframework.messaging.MessageChannel;
   import org.springframework.messaging.MessageHeaders;
   import org.springframework.messaging.support.MessageBuilder;
   import org.springframework.stereotype.Component;
   
   import java.util.HashMap;
   
   @Component
   @EnableBinding({Source.class,MySource.class})//绑定类
   public class StreamProducer {
   
       @Autowired
       @Qualifier(Source.OUTPUT) // Bean 名称
       private MessageChannel messageChannel;
   
       //注入的使用方式1
       @Autowired
       private MySource mySource;
   
       //注入的使用方式2
       @Autowired
       @Qualifier(MySource.NAME)  //NAME
       private MessageChannel myMessageChannel;
   
       public void send(String message){
           //通过消息管道发送消息
           messageChannel.send(MessageBuilder.withPayload(message).build());
           messageChannel.send(MessageBuilder.createMessage(MessageBuilder.withPayload(message).build(),new MessageHeaders(new HashMap<>())));
       }
   
       public void mySend(String message){
           //通过消息管道发送消息
           myMessageChannel.send(MessageBuilder.withPayload(message).build());
       }
   
   }
   
   
   ~~~

####  SINK(OUTPUT) 使用

**源码**

~~~java
/*
 * Copyright 2015 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.cloud.stream.messaging;

import org.springframework.cloud.stream.annotation.Input;
import org.springframework.messaging.SubscribableChannel;

/**
 * Bindable interface with one input channel.
 *
 * @see org.springframework.cloud.stream.annotation.EnableBinding
 * @author Dave Syer
 * @author Marius Bogoevici
 */
public interface Sink {

	String INPUT = "input";

	@Input(Sink.INPUT)
	SubscribableChannel input();

}
~~~



##### 原生channel用法

1. 使用`@EnableBinding` 绑定Sink

   ~~~java
   @EnableBinding({Sink.class})
   ~~~

2. 依赖注入

   1. 使用方式1进行注入`SubscribableChannel`

   ~~~java
   @Autowired
   @Qualifier(Sink.INPUT)
   private SubscribableChannel subscribableChannel;
   ~~~

   2. 使用方式2进行注入`Sink`

   ~~~java
   @Autowired
   private Sink sink;
   ~~~

3. 接收消息

   1. 使用subscribableChannel方式

      ~~~java
      @PostConstruct//构造函数构建后，执行的方法
      public void init(){
          subscribableChannel.subscribe(new MessageHandler() {
              @Override
              public void handleMessage(Message<?> message) throws MessagingException {
                  System.out.println("PostConstruct:"+message.getPayload());
              }
          });
      }
      ~~~

   2. 使用`@StreamListener` 方式

      ~~~java
      @StreamListener(Sink.INPUT)
      public void getMessageStream(String message){
          System.out.println("getMessageStream:"+message);
      }
      ~~~

   3. 使用`@ServiceActivator` 方式

      ~~~java
      @ServiceActivator(inputChannel = Sink.INPUT)
      public void getMessage(String message){
          System.out.println("getMessage:"+message);
      }
      ~~~

   ~~~java
   package com.wl.kafka.backdemo.com.kafka.stream.input;
   
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.beans.factory.annotation.Qualifier;
   import org.springframework.cloud.stream.annotation.EnableBinding;
   import org.springframework.cloud.stream.annotation.StreamListener;
   import org.springframework.cloud.stream.messaging.Sink;
   import org.springframework.integration.annotation.ServiceActivator;
   import org.springframework.messaging.Message;
   import org.springframework.messaging.MessageHandler;
   import org.springframework.messaging.MessagingException;
   import org.springframework.messaging.SubscribableChannel;
   import org.springframework.stereotype.Component;
   
   import javax.annotation.PostConstruct;
   
   @Component
   @EnableBinding({Sink.class})
   public class StreamConsumer {
   
       @Autowired
       @Qualifier(Sink.INPUT)
      private SubscribableChannel subscribableChannel;
   
       @Autowired
       private Sink sink;
   
       //使用方式1
       @PostConstruct
       public void init(){
           subscribableChannel.subscribe(new MessageHandler() {
               @Override
               public void handleMessage(Message<?> message) throws MessagingException {
                   System.out.println("PostConstruct:"+message.getPayload());
               }
           });
       }
   
       //方式2
       @StreamListener(Sink.INPUT)
       public void getMessageStream(String message){
           System.out.println("getMessageStream:"+message);
       }
   
       //方式3
       @ServiceActivator(inputChannel = Sink.INPUT)
       public void getMessage(String message){
           System.out.println("getMessage:"+message);
       }
   
   }
   ~~~

4. 配置application.properties

   ~~~properties
   kafka.topic = liangdu
   ###基本模式如下
   ### spring.cloud.stream.bindings.${channel-name}.destination = ${kafka.topic}
   spring.cloud.stream.bindings.input.destination = ${kafka.topic}
   ~~~

##### 自定义 channel

​	同 Source（OUTPUT） 使用 





###  Spring Cloud Stream Binder: RabbitMQ

​	同Spring Cloud Stream kafka一样,使用Stream方式（消息管道）

​	不同点：依赖的包不同

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-ribbit</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ribbit</groupId>
    <artifactId>spring-ribbit</artifactId>
</dependency>
~~~





### Binding 实现



### Binder 实现





## 问答部分

+ `@EnableBinding` 有什么用？

  答：`@EnableBinding` 将` Source`,`Sink`,`Processor` 提升成响应的代理

+ `@Autowird` Source source 写法是默认的官方写法

  答：是的

+ 这么多的消息框架，有什么优劣点？

  ActiveMQ : AMQP规范、JMS规范

+ **`@EnableBinder`  `@EnableZuulProxy` `@EnableDiscoverClient` 这些注解都是通过指定`@BeanPostProcessor` 实现的吗？**

  答：不完全对，主要处理接口在`@Import` :

  + `ImportSelector ` 实现类
  + `ImportBeanDefinitionRegistrar` 实现类
  + `Configuration` 标注类

+ 流式处理的场景？

  答：Stream 处理简单的说：异步处理，消息是一种处理方式

   提交申请，机器生成，对于高密度的提交任务， 多数场景采用异步处理，Stream、Event-Driven  举例说明：审核流程，鉴别黄图。

+ 如果是大量消息，怎么快速消费， 用多线程嘛 ？

  答：确实是使用多线程，不过不一定奏效。依赖于处理具体的内容，比如：一个线程使用了25%的CPU，4个线程就把CPU耗尽了。因此，同时处理100个，实际上，还是4个在处理。

  主要看：I/O密集型、CPU密集型

+ 七牛云？？？？？？？？？？？？？？？？？？、、













## 错误

1. 出现错误的时候

Caused by: java.lang.NoSuchMethodError: org.apache.kafka.clients.consumer.ConsumerRecord.headers()

添加依赖注解

```xml
<dependency>
   <groupId>org.apache.kafka</groupId>
   <artifactId>kafka-clients</artifactId>
   <version>1.1.0</version>
</dependency>
```