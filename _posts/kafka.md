---
title:  kafka
date: 2020-02-16 18:28:36
tags: kafka,分布式
---



## kafka



分布式消息和订阅系统，高性能，高吞吐量；sala语言.



内置分区、实现集群、



#### 企业中的应用：

行为跟踪：

​		通过用户的操作，可以分析用户需要什么

日志收集：

![1540176222750](https://www.lei32323.com/kafka_日志收集.png)



#### 架构图：

![1540176323471](https://www.lei32323.com//kafa_架构图.png)

##### Topic ：类似于大表

##### partition：类似于大表中的小表

##### group:消费端所属的组

 

#### kafka和mq的不同点：

1：producer，把消息发送到borker。（mq是把消息发送到Consumer）

2：Consumer，消息不是producer发送过来的，是直接从broker中pull下来的。（Mq是直接从Producer中获取的消息）

3：可以有多个producer去broker中发送消息，也可以有多个Consumer去broker中获取消息



#### 插件使用

##### 启动：

方式一：

当没有zookeeper的时候，使用自带的zookeeper启动（设置zookeeper.properties中的zk地址）

~~~bash
bin/zookeeper-server-start.sh config/zookeeper.properties
~~~



方式二：

当有zookeeper的时候，设置server.properties中的zk地址（`zookeeper.connect`）

~~~bash
bin/kafka-server-start.sh config/server.properties
~~~





后台运行的时候，再启动时加上`-daemon`

~~~bash
bin/kafka-server-start.sh -daemon config/server.properties
~~~





##### 创建topic

~~~bash
sh bin/kafka-topics.sh --create --zookeeper 192.168.83.128:2181 --replication-factor 1 --partitions 1 --topic test
~~~



zookeeper 如果是集群的话，中间用,号隔开



##### 查看topic

~~~bash
sh bin/kafka-topics.sh --list --zookeeper 192.168.83.128:2181
~~~





##### producer发送消息

~~~bash
sh bin/kafka-console-producer.sh --broker-list 192.168.83.129:9092 --topic test
~~~





##### Consumer接收消息

~~~bash
sh bin/kafka-console-consumer.sh --bootstrap-server 192.168.83.129:9092 --topic test --from-beginning
~~~





##### 设置多代理集群

修改server.properties

~~~properties
	broker.id=1  //属性是群集中每个节点的唯一且永久的名称
    listeners=PLAINTEXT://当前kafka的ip:9093
    log.dirs=/tmp/kafka-logs-1
~~~

##### 创建一个集群topic

```bash
sh bin/kafka-topics.sh --create --zookeeper 192.168.83.128:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```

--replication-factor 有几个副本写几个

--partitions 想分几个区写几个



##### 查看topic在集群的状态

~~~bash
sh bin/kafka-topics.sh --describe --zookeeper 192.168.83.128:2181 --topic my-replicated-topic
~~~



##### 使用Kafka Connect导入/导出数据

1.准备一个文件

2.执行脚本

```bash
sh bin/connect-standalone.sh config/connect-standalone.properties config/connect-file-source.properties config/connect-file-sink.properties
```







# API使用

## Producer

### ProducerConfig.ACK_CONFIG（-1）  

​	0  消息发送给broker以后，不需要确认，缺点：性能较高，但是数据容易丢失  

​	1 只需要获得kafka集群中leader节点的确认即可返回(leader/follower) 缺点：性能较高，但是数据容易丢失

​	all(-1)  需要ISR中所有的Replica进行确认(需要集群中的所有节点确认)，最安全，但是性能是最低的     缺点：当isr中只有一个副本的时候，容易出现数据丢失

### Batch.size（默认16KB）配合linger.ms使用，满足一个即可

​	Producer 对于同一个分区来说，会按照batch.size的大小进行统一收集进行批量发送

### Linger.ms(默认是0毫秒)  配合batch.size使用，满足一个即可

​	5message/s  延迟几秒进行收集，然后聚合，再次批量发送到broker 

### MAX_REQUEST_SIZE_CONFIG (默认是1M)

​	发送的请求大小。

#### RETRIES_CONFIG 重试测试



### RETRY_BACKOFF_MS_CONFIG 重发时间

失败之后，等待多久进行重发



## 消息的同步发送和异步发送

kafka在1.1.0之后。默认是异步发送



## Consumer

### GROUP_ID_CONFIG(group.id)  消费组

每个消费者必须要存在于消费组中

第一种情况：同一个组中只有一个消费组才能消费（类似于P2P)

第二种情况：  



### AUTO_OFFSET_RESET_CONFIG

对于新的groupid来说，如果设置了earliest,那么他会从最早的消息开始消费

对于已经存在的groupid来说，如果设置了earliest，那么他会从offset最大的开始消费

latest:对于新的groupid,直接取已经消费并且提交的	最大的Offset

earliest:对于新的groupid来说，重置Offset

none：如果新的groupid，没有offset的时候，会抛出异常

### ENABLE_AUTO_COMMIT_CONFIG 自动提交

true:自动提交，配合AUTO_COMMIT_INTERVAL_MS_CONFIG使用

false:手动提交，通过consumer.commitAsync() 提交

### AUTO_COMMIT_INTERVAL_MS_CONFIG  毫秒

多久后自动提交，配合ENABLE_AUTO_COMMIT_CONFIG使用

### MAX_POLL_RECORDS_CONFIG 

每一次返回的消息数量，减少poll的间隔次数 





## spring 整合kafka实现注册成功以后去设置抽奖名额（赠送一次抽奖机会）

项目1：用户进行注册，注册后返回注册成功，并且对kafka进行发送一条消息，说明用户已经注册成功了

项目2：监听消息，当获取到消息后，读取消息信息，进行对该用户进行设置一次抽奖机会。







## Topic&Partition

Topic是存储消息的逻辑概念

partition:

1.每个topic可以划分多个分区

2.相同topic下的不同分区的消息 是不同的





### 消息：

[key]->value

根据key来决定消息发放到哪个分区

#### 自定义分区策略

默认算法是hash取模算法



#### 消费机制 

一个消费者对一个分区对应消费 

一个消费者对2个分区的话，则会消费2个分区

2个消费者对1个分区的话，则会消费一个分区

跨分区不保证顺序

注意：consumer的数量>partition的数量  consumer最好是partition的整数倍 



增减consumer,borker,partition会导致Rebalance（重新负载）



#### 分区分配策略

##### Range（范围）-》默认

假设10个分区 对应3个消费者

topic1 partition 0,1,2,3,4,5,6,7,8,9

topic2 partition 0,1,2,3,4,5,6,7,8,9

C1/C2/C3

10/3 用分区总和/消费者

C1: topic1 0,1,2,3                   topic2 0,1,2,3

C2: 4,5,6

C3: 7,8,9

##### RoundBobin(轮询) 

按照Hashcode进行排序，进行轮询





##### 什么时候触发rebalance策略？

1.对于consumer group 新增消费者的时候 会触发rebalance（重新负载，使用范围分区策略）

2.消费者离开consumer group

3.topic中新增分区

4.消费者主动取消订阅topic



##### partition.assignment.strategy

consumer_offsets分区的总数



##### 谁来执行rebalance 重新负载以及管理consumer group？

1. 选举coordinator

   kafka提供了一个角色->Coordinator

   怎么去确定Coordinator -> GroupGoordinatorRequest 返回负载最小得borker节点的id

2. 选举leader

   消费者只要启动就会发送一个joingroup请求(包含group.id,member_id,protocol_metadata....信息)，coordinator会从consumer group中选举一个consumer担任leader角色，然后选举结果(generation_id,leader_id,members-leader(只能是leader用户才会有值，其他的都是空),member_metadata....)返回给每一个消费者

###### joinGroupRequest

发送joingroup告诉coordinator说要加入到group

###### syncGroupRequest 

发送同步请求，维持心跳。如果没有响应，说明挂掉了。就会执行重新负载的机制（默认是范围）

有响应会返回一个member_assignment（分发消息策略）给consumer。consumer执行分发信息



#### Offset在哪维护？

consumer_offsets(topic)  默认有50个分区

1.先定位当前consumer_group 存储在哪个分区上，

2.使用计算公式`group.id.hashCode()%groupMetadataTopicPatitionCount(就是consumer_offsets分区数)`

3.等于几就相当于消息存储在哪个consumer_offsets里面

![1540370326620](https://www.lei32323.com/kafka_offset查看.png)

~~~bash
sh kafka-simple-consumer-shell.sh --topic __consumer_offsets --partition 所在的那个分区 --broker-list 192.168.83.129:9092,192.168.83.130:9092,192.168.83.131:9092 --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter"
~~~



#### 消息的存储

##### 消息的保存路径

​	-> topic(逻辑)

​	->partition[topic_0/topic_1/topic_2]



##### 消息的写入性能

按照顺序写入消息

(io瓶颈)零拷贝 

消息的存储机制



##### LogSegment 分段保存

index(索引)->log(消息内容)

index->log



##### 消息的清理（压缩）

server.properties

根据时间来保存

根据大小来保存





##### patition的副本的概念

###### 副本中也存在leader副本和follower副本的概念

~~~bash
sh bin/kafka-topics.sh --create --zookeeper 192.168.83.128:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
~~~

replication-factor 3     为副本的备份为3份

get /brokers/topics/topic_name1/partitions/partition_num/state

{"controller_epoch":"1","leader":"1","version":"1","leader_epoch":"0","isr":[1,0,2]}



leader:leader是谁。



###### leader副本：负责接收客户端的消息写入和消息的读取请求

###### follower副本：负责从Leader副本去读取数据(不接受客户端的请求，主要是做备份使用)



###### isr：维护当前分区的所有的副本集。（in sync repilcas)

follower副本集必须要和leader副本的数据再阈值范围内保持一致

就是说，当一个副本从Leader同步消息延迟到了配置的时间。就会从isr中剔除去，一直等到同步的时间达到配置的时间内就会重新加入到isr中

isr(replica.lag.time.max.ms)  follwer副本同步leader副本的关键时间（1.X的版本）



###### 如果isr为空怎么办???

1.等待isr中的任意一个replica活过来，重新选举Leader

2.选择一个活过来的replica作为leader（可能不包含在isr集合中，默认策略)



###### LEO

​	log end offset  消息写到最后一个文件



###### HW

​	如果消息被标记为HW的话，那么代表这个消息可以被消费。如果没有被标记的话，则代表不能被消费



## 消息的同步

初始化 ，没有消息发送过来的时候。

![1540804103612](https://www.lei32323.com/kafka_没有消息发送过来的时候.png)

有消息进来的时候

![1540803926214](https://www.lei32323.com/有消息进来的时候.png)

1：把消息写入到对应分区的Log文件中，同时更新Leader副本的LEO

2： 尝试去更新Leader的HW的值，比较自己本身的LEO和remote LEO的值，取最小的值作为HW



数据开始同步的时候

![1540804233152](https://www.lei32323.com/kafka_数据开始同步的时候.png)

1：Leader副本读取Log消息，并且去更新remote LEO (根据followe的fetch传递过来的offset)

2：去更新 HW，HW=0

3：把读取到的消息和当前分区的HW值返回给fowllower副本



第一次交互之后 （推送消息）

![1540804594570](https://www.lei32323.com/kafka_推送消息.png)

HW的取值：拿着HW和LEO进行比较，取最小值所以为0

这次交互之后是没有数据能消费的



第二次交互 （确认消息）

![1540804801092](https://www.lei32323.com/kafka_确认消息.png)

这个时候HW为1的时候 那么就可以消费第一条消息了





### 存在数据丢失的情况

​	再这个前提下：min.insunc.replicas=1的时候，并且acks=-1的情况下。







## 监控

kafka monitor

kafka offset monitor

kafka -manager



## 分区应该设置多少合适？？？

根据项目吞吐量来决定

1.采用操作系统层面的页缓存来缓存数据

2.日志采用顺序写入以及零拷贝的方式来提升IO性能

3.partition的水平分区概念，能够把一个topic拆分多个分区

4.发送端和消费端都可以采用并行的方式来消费分区中的消息



设置50000个分区。->批量发送batch.size,linger.size.ms(针对用一个partition而言)->内存占用过高

50000个消费者->多线程（怎么去分配consumer的消费能力)



文件句柄。logsegment->index/log文件



副本->1个分区1个副本，  5000个分区落到10个broker上。每个Broker有5000个分区，如果1个broker挂了。那么5000个分区不能用了。存在5000个分区需要选举leader

1.硬件资源

2.消息大小

3.目标吞吐量

（峰值的消息大小）一个topic一个分区的情况下，针对目标硬件资源压测->tps   发送端的tps和消费端的tps

如果每秒10M的吞吐量的话，那么业务需要100M/S的吞吐量，则需要10个broker即可







# 使用KafkaOffsetMonitor

```
java -cp KafkaOffsetMonitor-assembly-0.2.1.jar \
     com.quantifind.kafka.offsetapp.OffsetGetterWeb \
     --offsetStorage kafka
     --zk zk-server1,zk-server2 \
     --port 8080 \
     --refresh 10.seconds \
     --retain 2.days
```