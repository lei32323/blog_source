# active mq使用

## queue的使用

### 创建ActiveMQConnectionFactory工厂

~~~java
ActiveMQConnectionFactory activeMQConnectionFactory = new ActiveMQConnectionFactory("tcp://ip:端口");
~~~

tcp://ip:端口  =  tcp:///192.168.9.11:16161 

后面可以追加参数如：`jms.useAsynSend=true`(异步请求) , `jms.producerWindowSize=90` (设置存储的大小，针对于异步发送消息有用)

### 创建一个连接

```java
Connection connection = activeMQConnectionFactory.createConnection();
connection.start();
```

再connection可以设置









# active mq 发送消息

~~~java
//调用
producer.send(textMessage);
~~~

~~~java
 //ActiveMQMessageProducerSupport
 send(Message message);
~~~

~~~java
//ActiveMQMessageProducer
public void send(Destination destination, Message message, int deliveryMode, int priority, long timeToLive)
~~~

~~~~java
//ActiveMQMessageProducer
public void send(Destination destination, Message message, int deliveryMode, int priority, long timeToLive, AsyncCallback onComplete){
    ...
    //验证Producer是否已经关闭
	checkClosed();
    ...
    //如果producerWindow不为Null
	 if (producerWindow != null) {
            try {
                //等待producerWindow的大小释放
                producerWindow.waitForSpace();
            } catch (InterruptedException e) {
                throw new JMSException("Send aborted due to thread interrupt.");
            }
        }
	...
 	this.session.send(this, dest, message, deliveryMode, priority, timeToLive, producerWindow, sendTimeout, onComplete);
}
~~~~

~~~java
//ActiveMQSession
protected void send(ActiveMQMessageProducer producer, ActiveMQDestination destination, Message message, int deliveryMode, int priority, long timeToLive,
                    MemoryUsage producerWindow, int sendTimeout, AsyncCallback onComplete) throws JMSException {
    //验证session是否已经关闭
    checkClosed();
    ...
        //加上锁执行
     	synchronized (sendMutex) {
       	 	//事务是否开始
        	doStartTransaction();
        	...
        	//消息是否持久化
        	message.setJMSDeliveryMode(deliveryMode);
        	...
            //消息设置成只读
            msg.onSend();
        	...
            //onComplete==null：onComplete 发送回调内容为空
            //sendTimeout <= 0:设置的超时时间小于等于0
            //!msg.isResponseRequired():消息不需要响应
            //!connection.isAlwaysSyncSend():非同步发送连接
                
           	//(!msg.isPersistent() || connection.isUseAsyncSend() || txid != null)
            //!msg.isPersistent():非持久化消息
            //connection.isUseAsyncSend():connection异步发送
            //txid != null:开启了事务
             if (onComplete==null && sendTimeout <= 0 && !msg.isResponseRequired() && !connection.isAlwaysSyncSend() && (!msg.isPersistent() || connection.isUseAsyncSend() || txid != null)) {
                 //异步发送
                this.connection.asyncSendPacket(msg);
                 if (producerWindow != null) {
                     ...
                    //设置producerwindow的creaseUsage的大小
                    int size = msg.getSize();
                    producerWindow.increaseUsage(size);
                 }
             }else{
                 if (sendTimeout > 0 && onComplete==null) {
                     //同步发送
                    this.connection.syncSendPacket(msg,sendTimeout);
                }else {
                    this.connection.syncSendPacket(msg, onComplete);
                }
             }
    }
    
}
~~~

## 异步发送

~~~java
//ActiveMQConnection
 public void asyncSendPacket(Command command) throws JMSException {
     //判断是否close
 	if (isClosed()) {
     	throw new ConnectionClosedException();
 	 } else {
        //执行异步
    	 doAsyncSendPacket(command);
 	 }
 }
~~~

~~~java
//ActiveMQConnection
private void doAsyncSendPacket(Command command) throws JMSException {
        try {
            //执行transport的oneway
            this.transport.oneway(command);
        } catch (IOException e) {
            throw JMSExceptionSupport.create(e);
        }
  }
~~~

### this.transport是啥?

当前位置往上翻，看到本类是ActiveMQConnection。看到属性`private final Transport transport`,猜测：应该是在初始化connection的时候赋值的？？？

找到connection创建的地方

`Connection connection = factory.createConnection();` 

~~~java
//ActiveMQConnectionFactory
public Connection createConnection() throws JMSException {
        return createActiveMQConnection();
    }
~~~

~~~java
//ActiveMQConnectionFactory
protected ActiveMQConnection createActiveMQConnection(String userName, String password) {
    ...
    //创建transport
    Transport transport = createTransport();
    ...
    //启动transport
     transport.start();
}
~~~

#### 发现创建transport的地方：

`Transport transport = createTransport();` 

~~~java
// ActiveMQConnectionFactory
protected Transport createTransport() {
    //获取broker地址
     URI connectBrokerUL = brokerURL;
     String scheme = brokerURL.getScheme();
     if (scheme == null) {
          throw new IOException("Transport not scheme specified: [" + brokerURL + "]");
      }
    //判断scheme
     if (scheme.equals("auto")) {
         //在这里我们没有设置，为默认的auto 那么就是tcp
         connectBrokerUL = new URI(brokerURL.toString().replace("auto", "tcp"));
      } else if (scheme.equals("auto+ssl")) {
         connectBrokerUL = new URI(brokerURL.toString().replace("auto+ssl", "ssl"));
      } else if (scheme.equals("auto+nio")) {
          connectBrokerUL = new URI(brokerURL.toString().replace("auto+nio", "nio"));
      } else if (scheme.equals("auto+nio+ssl")) {
           connectBrokerUL = new URI(brokerURL.toString().replace("auto+nio+ssl", "nio+ssl"));
      }
    //调用TransportFactory的connect进行构建对象
    return TransportFactory.connect(connectBrokerUL);
 }
~~~

转到transportFactory类上

~~~java
//TransportFactory
public static Transport connect(URI location) throws Exception {
    //此处为创建TcpTransportFactory
        TransportFactory tf = findTransportFactory(location);
    //组装transport
        return tf.doConnect(location);
    }
~~~

~~~java
//TransportFactory
public static TransportFactory findTransportFactory(URI location) {
    String scheme = location.getScheme();
    if (scheme == null) {
        throw new IOException("Transport not scheme specified: [" + location + "]");
     }
    //判断再集合中是否存在。
    TransportFactory tf = TRANSPORT_FACTORYS.get(scheme);
    if (tf == null) {
      // Try to load if from a META-INF property.
       try {
           //不存在就创建一个TcpTransportFactory
           tf = (TransportFactory)TRANSPORT_FACTORY_FINDER.newInstance(scheme);
           TRANSPORT_FACTORYS.put(scheme, tf);
        } catch (Throwable e) {
           throw IOExceptionSupport.create("Transport scheme NOT recognized: [" + scheme + "]", e);
        }
     }
        return tf;
 }
~~~

查看`TRANSPORT_FACTORYS` 变量为

`ConcurrentMap<String, TransportFactory> TRANSPORT_FACTORYS = new ConcurrentHashMap<String, TransportFactory>()` 

查看`TRANSPORT_FACTORY_FINDER` 变量为

`private static final FactoryFinder TRANSPORT_FACTORY_FINDER = new FactoryFinder("META-INF/services/org/apache/activemq/transport/")` 

看到这里有点类似于java的spi思想。我们去META-INF/services/org/apache/activemq/transport/这个位置看下

找到对应的tcp文件

看到文件中的内容:

```java
class=org.apache.activemq.transport.tcp.TcpTransportFactory
```

我们再回到创建的地方会发现`path+key` 

~~~java
//FactoryFinder
public Object newInstance(String key) throws IllegalAccessException, InstantiationException, IOException, ClassNotFoundException {
    //通过path地址+key进行加载TcpTransportFactory类
        return objectFactory.create(path+key);
    }
~~~

#### 这里还有一行代码，别忘记看咯

~~~java
//TransportFactory-->connect(URI location){}
return tf.doConnect(location)
~~~

我们进入看下

~~~java
//TransportFactory
public Transport doConnect(URI location, Executor ex) throws Exception {
        return doConnect(location);
    }
~~~

进入到`duConnect()`  `TransportFactory`类

~~~java
//TransportFactory
public Transport doConnect(URI location) throws Exception {
        try {
            Map<String, String> options = new HashMap<String, String>(URISupport.parseParameters(location));
            if( !options.containsKey("wireFormat.host") ) {
                options.put("wireFormat.host", location.getHost());
            }
            //创建WireFormat 发送数据解析相关的协议信息。比如说缓存
            WireFormat wf = createWireFormat(options);
            //创建Transport
            Transport transport = createTransport(location, wf);
            //配置信息 如：Socket
            Transport rc = configure(transport, wf, options);
            //remove auto 
            IntrospectionSupport.extractProperties(options, "auto.");

            if (!options.isEmpty()) {
                throw new IllegalArgumentException("Invalid connect parameters: " + options);
            }
            return rc;
        } catch (URISyntaxException e) {
            throw IOExceptionSupport.create(e);
        }
    }

~~~

先来看下创建WireFormat`createWireFormat`

~~~java
//TransportFactory
protected WireFormat createWireFormat(Map<String, String> options) throws IOException {
        WireFormatFactory factory = createWireFormatFactory(options);
        WireFormat format = factory.createWireFormat();
        return format;
    }
~~~

再来看创建Transport`createTransport(location, wf)`通过创建好的wireFormat进行创建transport

~~~java
//TransportFactory
protected Transport createTransport(URI location, WireFormat wf) {
    ...
}
~~~

最后我们来看下`configure(transport, wf, options)` 

~~~java
//TransportFactory
public Transport configure(Transport transport, WireFormat wf, Map options)  {
    //封装composite 实现心跳机制
        transport = compositeConfigure(transport, wf, options);
	//封装mutexTransport 实现锁
        transport = new MutexTransport(transport);
    //封装responseCorrelator 实现异步请求
        transport = new ResponseCorrelator(transport);

        return transport;
    }
~~~

**最终封装成一个ResponseCorrelator(MutexTransport(WireFormatNegotiator(InactivityMonitor(Tcptransport))))**

**拥有解析数据，实现心跳机制，实现锁，实现异步请求的transport**



看到这里。我们已经明白了transport从哪里来的了。



我们再回过头来看异步发送,

~~~java
//ActiveMQConnection
private void doAsyncSendPacket(Command command) throws JMSException {
        try {
            //执行transport的oneway
            this.transport.oneway(command);
        } catch (IOException e) {
            throw JMSExceptionSupport.create(e);
        }
  }
~~~

因为上面已经发现了transport从哪来的了,所以我们只要找到tcpTransport上的`oneway`方法

~~~java
//TcpTransport
@Override
    public void oneway(Object command) throws IOException {
        checkStarted();
        //执行消息发送
        wireFormat.marshal(command, dataOut);
        dataOut.flush();
    }
~~~

