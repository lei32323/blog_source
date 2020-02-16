# **Spring** cloud



### 服务短路

QPS : 经过全链路压测，计算机单机极限QPS ，集群QPS =单机QPS * 机器数 * 可靠性比率

全链路压测，除了压极限QPS ，还有错误数量



### Spring cloud Hystrix Client

针对于方法，任何Bean都可以使用

**官网：**

> Reactive Java框架
>
> - Java 9 Flow API
> - Reactor 
> - Rxjava(Reactive X)

#### 操作1  注解的方式

1. 激活Hystrix 

   在Application启动类上添加`@EnableHystrix`  

   注意：`@EnableHystrix` 激活，没有一些spring cloud功能

2. 激活熔断保护

   `@EnableCircuitBreaker`  包含`@EnableHystrix` +spring cloud 信息 如：`/hystrix.stream`

3. 在对应的方法上添加注解`@HystrixCommand(fallbackMethod="errorContent")`

```java
@RequestMapping("/hello")
@HystrixCommand(fallbackMethod="errorContent"，
commandProperties={@HystrixProperties(name="execution.isolation.thread.timeoutInMilliseconds",value="100")})
public String helloWorld(){
    return "hello world";
}
```



4. 添加errorContent方法

```java
public String errorContent(){
    return "Fault"
}
```

**注意：** 新增的errorContent方法 入参、出参、方法名必须一致

> Hystrix配置信息的wiki: https://github.com/Netflix/Hystrix/wiki/Configuration
>
>

~~~properties
Execution:
execution.isolation.strategy   执行隔离战略
execution.isolation.thread.timeoutInMilliseconds  执行、隔离、线程、超时、毫秒
execution.timeout.enabled   执行超时时间启用
execution.isolation.thread.interruptOnTimeout  执行中断隔离线程超时超时
execution.isolation.thread.interruptOnCancel  执行中断
execution.isolation.semaphore.maxConcurrentRequests 

Fallback:
fallback.isolation.semaphore.maxConcurrentRequests 
fallback.enabled   回调函数启用状态

Circuit Breaker:
circuitBreaker.enabled   断路器启用状态
circuitBreaker.requestVolumeThreshold  断路器请求容量存储
circuitBreaker.sleepWindowInMilliseconds 断路器睡眠窗口的时间
circuitBreaker.errorThresholdPercentage 断路器误差阈值百分率
circuitBreaker.forceOpen   断路器强制打开
circuitBreaker.forceClosed  断路器强制关闭

Metrics:
metrics.rollingStats.timeInMilliseconds
metrics.rollingStats.numBuckets
metrics.rollingPercentile.enabled
metrics.rollingPercentile.timeInMilliseconds
metrics.rollingPercentile.numBuckets
metrics.rollingPercentile.bucketSize
metrics.healthSnapshot.intervalInMilliseconds

Request Context:
requestCache.enabled  请求缓存开启状态
requestLog.enabled 请求日志开启状态

Collapser Properties:
maxRequestsInBatch    最大请求设置
timerDelayInMilliseconds
requestCache.enabled   请求缓存开启状态

Thread Pool Properties:
coreSize    线程池核心大小
maximumSize  线程池最大线程数
maxQueueSize  最大对猎术
queueSizeRejectionThreshold  队列退出阈值
keepAliveTimeMinutes  固定时限
allowMaximumSizeToDivergeFromCoreSize
metrics.rollingStats.timeInMilliseconds  滚动统计时间毫秒
metrics.rollingStats.numBuckets  
~~~





#### 操作2  编程方式

```java
public class HelloWorldCommand extends com.netflix.hystrix.HystrixCommand<String>{
    protected HelloWorldCommand (){
        //HelloWorld 为组名字
        super(HystrixCommandGroupKey.Factory.asKey("HelloWorld"),100);
    }
    
    
    @Override
    protected String run() throws Exception {
        return "Hello,World"
    }
    
    protected String getFallback(){
        return "fault"
    }
}
```

```java
//调用方式
new HelloWorldCommand().execute();
```



**通过aop实现**

~~~java
import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import com.netflix.hystrix.HystrixCommandProperties;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Component
@Aspect
public class HystrixCommandAdvice {

    @Pointcut("execution(* com.wl.cloud.springcloudhystrixdemo.api.service.CommonService.*(..))")
    public void hystrixPointcut(){}

    @Around("hystrixPointcut()")
    public  Object runCommand(final ProceedingJoinPoint proceedingJoinPoint){
        return wrapWithHystrixCommand(proceedingJoinPoint).execute();
    }

    private HystrixCommand<Object> wrapWithHystrixCommand(final ProceedingJoinPoint proceedingJoinPoint){
        String method = proceedingJoinPoint.getSignature().getName();
        return new HystrixCommand<Object>(setter(method)){

            @Override
            protected Object run() throws Exception {
                try {
                    return proceedingJoinPoint.proceed();
                } catch (Throwable throwable) {
                  throw   (Exception)throwable;
                }
            }

            @Override
            protected String getFallback() {
                return "execute fault";
            }
        };
    }

   private HystrixCommand.Setter setter(String method) {
        return HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey(method==null?"groupName":method))
                .andCommandKey(HystrixCommandKey.Factory.asKey(method==null?"commandName":method))
                //设置时间为100毫秒
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(100));
    }
}
~~~





#### /health 

可以查看hystrix 的安全信息



#### /hystrix.stream

可以查看hystrix信息

~~~json
data: {
  "type": "HystrixThreadPool",
  "name": "helloworld",
  "currentTime": 1542779065966,
  "currentActiveCount": 0,
  "currentCompletedTaskCount": 5,
  "currentCorePoolSize": 10,
  "currentLargestPoolSize": 5,
  "currentMaximumPoolSize": 10,
  "currentPoolSize": 5,
  "currentQueueSize": 0,
  "currentTaskCount": 5,
  "rollingCountThreadsExecuted": 4,
  "rollingMaxActiveThreads": 1,
  "rollingCountCommandRejections": 0,
  "propertyValue_queueSizeRejectionThreshold": 5,
  "propertyValue_metricsRollingStatisticalWindowInMilliseconds": 10000,
  "reportingHosts": 1
}
~~~



### Spring Cloud Hystrix Dashboard

1. `@EnableHystrixDashboard`  application启动类上添加注解
2. 浏览器输入`/hystrix.stream`

### 整合Netflix Turbine

1. `@EnableTurbine`application启动类上添加注解
2. 配置信息

~~~properties
server.port=7070

turbine.aggregator.cluster-config=default
## Eureka服务器中的serviceId列表
turbine.app-config=eureka-server,user-consumer,user-provider
## 指定了集群名称为default，
turbine.cluster-name-expression="default"
## 可以让同一主机上的服务通过主机名与端口号的组合来进行区分，默认情况下会以host来区分不同的服务
turbine.combine-host-port=true

### eureka服务端的端口
eureka.client.eureka-server-port= 9090
### eureka客户端连接的服务端地址  多个服务端地址中间用,隔开
eureka.client.service-url.defaultZone = http://localhost:${eureka.client.eureka-server-port}/eureka
~~~

3. 监控信息

![1542789144207](https://wanglei.club/Turbine.png)

**注意：** 就是连接Eureka服务器端，拦截Eureka客户端被熔断保护的方法上。





