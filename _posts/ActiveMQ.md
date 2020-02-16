---
title: Active mq 使用 
date: 2020-02-16 20:14:36
tags: Active mq,分布式
---

## 介绍

1. 从JMS规范了解Active MQ

   JMS的定义：消息中间件的API； jBoss MQ ,active MQ 

   MOM模式：面向消息中间件

2. JMS基本功能

   a) 消息传递域：point-point ;发布订阅（publish/subscribe）topic

   b) p2p:每个消息只能有一个消费者   ------点对点

   c) pub/sub: 每个消息有多个消费者   ----广播     只能订阅 “发布者发布后的消息”

   d) 消息头 消息体（Text,Map,Bytes,Stream,Object） 消息属性

   e) 发布，订阅的时间相关性 (发布者必须在线，订阅者也在线 才能收到  )

   f) 发布，订阅有持久化订阅，这样可以订阅者不在线。（生产者指定setClientID，消费者获取createDurableSubscriber）类似于QQ号

3. 消息的确认方式

   ACK机制：

   + AUTO_ACKNOWLEDGE = 1  自动确认消息，不需要手动调用message.acknowledge();;
   + CLIENT_ACKNOWLEDGE = 2    消费客户端手动确认 ，和发送方无关   （有10条消息进行消费，再第6条的时候进行确认消费，那么1-6都被消费了）
   + DUPS_OK_ACKNOWLEDGE = 3    自动批量确认，延迟消费
   + SESSION_TRANSACTED = 0    事务提交并确认
   + 消息的处理阶段：客户端接收消息，客户的处理消息，消息被确认

4. 会话存在2种机制

+  事务性会话
  + 事务型事务：
    + 对于发送端来说 是针对于消息的，commit()消息提交到队列 rollback()消息回滚到没发布到队列
    + 对于客户端来说 是针对于消息的，commit()消息已经确认消费了。Rollback()消息没有被处理，可以被其他人处理
  + 非事务型会话

+ 没有事务一说，所以没有commit和rollback，都为自动确认

  

5. P2P模型和PUB/SUB模型总结

+ P2P

  + 如果session 关闭时，有一些消息已经被收到，但是没有被签收，消费者再下一次连接到相同队列的时候，这些消息仍会被接收。

  + 如果用户再receive方法中设定了消息的选择条件（消息过滤）

  + 如果是持久化消息，消息会被持久化保存，直到消息被签收

+ PUB/SUB 
  + 持久化订阅和非持久化订阅
  + 在非持久化订阅的前提下，不能恢复或者重新指派一个未签收的消息
  + 如果所有消息必须要签收，则使用持久订阅
  + Session创建的时候第二个参数的含义
    + Session.AUTO_ACKNOWLEDGE为自动确认，客户端发送和接收消息不需要做额外的工作。哪怕是接收端发生异常，也会被当作正常发送成功。
    + Session.CLIENT_ACKNOWLEDGE为客户端确认。客户端接收到消息后，必须调用javax.jms.Message的acknowledge方法。jms服务器才会当作发送成功，并删除消息。
    + DUPS_OK_ACKNOWLEDGE允许副本的确认模式。一旦接收方应用程序的方法调用从处理消息处返回，会话对象就会确认消息的接收；而且允许重复确认。



### **P2P**

创建者quque

生产者

![生产者](https://wangllei.club/activemq 生产者.png)

如果需要持久化消息需要设置：

![](https://wanglei.club/activemq生产者2.png)



消费者

![](https://wanglei.club/activemq 消费者.png)



### **Topic**

生产者(持久化)

![](https://wanglei.club/activemq 生产者topic.png)

消费者：

![](https://wanglei.club/activemq 消费者topic.png)

![](https://wanglei.club/activemq 消费者topic2.png)



## **问题**

### 发布订阅 和P2P的区别？

​		P2P 是单对单的 类似于QQ号对QQ好 

​		发布订阅 是 广播的形式：生产者在线发布消息，消费者如果不在线就收不到消息，当消费者在线的时候才能收到生产者后续发的消息

### 事务和非事务的区别？

​	事务：

​		对于订阅者来说，提交事务，消息才会推送到队列中

​		对于消费者来说，提交事务，消息才能再队列中被消费

### 订阅消息持久化？

+  持久化和非持久化消息的发送策略

+  消息的持久化方案及实践

+  消费端消费消息的原理

+  关于PrefetchSize的优化	

### 持久化消息和非持久化消息的发送策略

​	setDeliveryMode

### 消息的同步发送和异步发送

消息同步发送和异步发送是针对Broker而言

默认情况，非持久化消息是异步发送的

非持久化消息并且在非事务模式下是同步发送的

​	`setDeliveryMode（DeliveryMode.NON_PERSISTENT）&& connection.createSession(FALSE,…)`

在开启事务的情况下，消息都是异步发送



手动设置异步发送：

1. brokerRL:后面追加：?jms.useAsynSend=true

   ![](https://wanglei.club/activemq 异步发送.png)

2. 代码形式设置：

   ![](https://wanglei.club/activemq 异步发送2.png)

![](https://wanglei.club/activemq 异步发送3.png)



### ProducerWindowSize（针对于异步发送消息有用）

设置方式 设置的为百分比  90=90%

1.  brokerRL:后面追加：?jms.producerWindowSize=90

2. 再配置文件中设置 `activemq.xml`

   ~~~xml
   <systemUsage>                                                           
               <systemUsage>                                             
                   <memoryUsage>                                                    
                       <memoryUsage percentOfJvmHeap="70" />  设置允许的内存大小（设置的为百分比）                                          
                   </memoryUsage>                                                                             
                   <storeUsage>                                                                               
                       <storeUsage limit="100 gb"/>                                                              
                   </storeUsage>                                                                              
                   <tempUsage>                                                                          
                       <tempUsage limit="50 gb"/>                                                          
                   </tempUsage>                                                                       
               </systemUsage>                                                                          
           </systemUsage>  
   ~~~

   

> 备注：1.设置的越大，消耗的内存也就越大
>
> ​		 2.当消息达到memoryUsage 内存限制的时候，对于非持久化消息来说，会把消息持久化到临时存储中tempUsage



## 源码解析

~~~java
if (onComplete==null && sendTimeout <= 0 && !msg.isResponseRequired() && !connection.isAlwaysSyncSend() && (!msg.isPersistent() || connection.isUseAsyncSend() || txid != null)) {
onComplete==null：onComplete 发送回调内容为空
sendTimeout <= 0:设置的超时时间小于等于0
!msg.isResponseRequired():消息不需要响应
!connection.isAlwaysSyncSend():非同步发送连接
(!msg.isPersistent() || connection.isUseAsyncSend() || txid != null)
!msg.isPersistent():非持久化消息
connection.isUseAsyncSend():connection异步发送
txid != null:开启了事务
If(以上都成立){步发送，否则为异步发送
		同步发送：this.connection.syncSendPacket(msg, sendTimeout);
}else{
		异步发送：this.connection.asyncSendPacket(msg)
		添加windows大小：producerWindow.increaseUsage((long)size);
}

	//通过transport执行消息发送
this.transport.oneway(command);

//创建transport
Transport transport = this.createTransport();  

//根据url创建对应的transport
URI connectBrokerUL = this.brokerURL; //获取brokerUrl
String scheme = this.brokerURL.getScheme();//获取scheme
if (scheme == null) {//如果为空的就报错
   throw new IOException("Transport not scheme specified: [" + this.brokerURL + "]");
} else {//默认为auto,根据不同的scheme创建不同的scheme
   if (scheme.equals("auto")) {
        connectBrokerUL = new URI(this.brokerURL.toString().replace("auto", "tcp"));
    } else if (scheme.equals("auto+ssl")) {
        connectBrokerUL = new URI(this.brokerURL.toString().replace("auto+ssl", "ssl"));
    } else if (scheme.equals("auto+nio")) {
        connectBrokerUL = new URI(this.brokerURL.toString().replace("auto+nio", "nio"));
    } else if (scheme.equals("auto+nio+ssl")) {
        connectBrokerUL = new URI(this.brokerURL.toString().replace("auto+nio+ssl", "nio+ssl"));
    }
	//通过工厂创建不同的transport
   return TransportFactory.connect(connectBrokerUL);
}
~~~



工厂形式获取transport `TransportFactory`

~~~java
public static TransportFactory findTransportFactory(URI location) throws IOException {
   	String scheme = location.getScheme();
   	if (scheme == null) {
    	   throw new IOException("Transport not scheme specified: [" + location + "]");
   	} else {
	//通过类似于java spi的思想 建获取对象   路径：META-INF/services/org/apache/activemq/transport/
	   TransportFactory tf = (TransportFactory)TRANSPORT_FACTORYS.get(scheme);
     	  if (tf == null) {
     	      try {
		//如果没有就创建一个
     	          tf = (TransportFactory)TRANSPORT_FACTORY_FINDER.newInstance(scheme);
     	          TRANSPORT_FACTORYS.put(scheme, tf);
     	      } catch (Throwable var4) {
       	        throw IOExceptionSupport.create("Transport scheme NOT recognized: [" + scheme + "]", var4);
       	    }
     	  }

    	   return tf;
 }
~~~

