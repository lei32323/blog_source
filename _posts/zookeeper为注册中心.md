> f版本：
>
> spring cloud :Edgware.SR5
>
> zookeeper:3.4.6
>
> jdk :1.8

**AP 高一致性**



**多线程异常处理，可以不用try catch ，线程不报错**

~~~java
Thread.currentThread().setUncanughtExceptionHandler((t,e)->{
	System.out.println(e.getMessage());
})

throws new RunntimeException("异常信息")；
~~~



# zookeeper 为注册中心

## zookeeper 作为注册中心的同时，也可以为配置中心

1. xml依赖配置

   ~~~xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
   </dependency>
   ~~~

2. properties配置

   ~~~properties
   # 设置zookeeper连接地址
   spring.cloud.zookeeper.connect-string=localhost:2181
   ~~~

3. zookeeper查看

   ~~~bash
    get /services/spirng-cloud-zk/f015d36b-a082-4755-aa7b-847bab54d439
   ~~~

   ~~~json
   {
     "name": "spirng-cloud-zk",
     "id": "f015d36b-a082-4755-aa7b-847bab54d439",
     "address": "DESKTOP-APGDO5C",
     "port": 55442,
     "sslPort": null,
     "payload": {
       "@class": "org.springframework.cloud.zookeeper.discovery.ZookeeperInstance",
       "id": "spirng-cloud-zk:0",
       "name": "spirng-cloud-zk",
       "metadata": {
         "instance_status": "UP"
       }
     },
     "registrationTimeUTC": 1545204393606,
     "serviceType": "DYNAMIC",
     "uriSpec": {
       "parts": [
         {
           "value": "scheme",
           "variable": true
         },
         {
           "value": "://",
           "variable": false
         },
         {
           "value": "address",
           "variable": true
         },
         {
           "value": ":",
           "variable": false
         },
         {
           "value": "port",
           "variable": true
         }
       ]
     }
   }
   ~~~

## 服务调用 初始

### 编写客户端应用

1. 服务注册中心客户端的关键类`DiscoveryClient`

   ~~~java
   @Autowire
   private DiscoveryClient discoveryClient;
   ~~~

2. 需要一个存放服务的集合

   ~~~java
    private  List<String> serviceNames = new CopyOnWriteArrayList<>();
   ~~~

3. 编写定时任务信息，没过60秒获取服务信息

   ~~~java
   // 每60秒获取服务器列表
   @Scheduled(fixedDelay = 1000*60)
   public void init(){
        Set<String> oldServiceNames = this.serviceNames;
       //1. 获取当前服务的机器
       List<ServiceInstance> instances = discoveryClient.getInstances(serviceName);
       //2. 获取服务列表 http://192.168.19.12:8080/
       Set<String>  newServiceNames = instances.stream().map(serviceInstance -> {
           return serviceInstance.isSecure()?
               "https://"+serviceInstance.getHost()+":"+serviceInstance.getPort():
           "http://"+serviceInstance.getHost()+":"+serviceInstance.getPort();
       }).collect(Collectors.toSet());
       //赋值
       this.serviceNames = newServiceNames;
       //清空老的
       oldServiceNames.clear();
   }
   ~~~

   **代码集合 **

   ~~~java
   @RestController
   public class HelloController {
   
       @Autowired
       private DiscoveryClient discoveryClient;
   
       private volatile   Set<String> serviceNames = new HashSet<>();
   
       @Value("${spring.application.name}")
       private String serviceName;
   
       // 每60秒获取服务器列表
       @Scheduled(fixedDelay = 1000*60)
       public void init(){
           Set<String> oldServiceNames = this.serviceNames;
           //1. 获取当前服务的机器
           List<ServiceInstance> instances = discoveryClient.getInstances(serviceName);
           //2. 获取服务列表 http://192.168.19.12:8080/
           Set<String>  newServiceNames = instances.stream().map(serviceInstance -> {
               return serviceInstance.isSecure()?
                       "https://"+serviceInstance.getHost()+":"+serviceInstance.getPort():
                       "http://"+serviceInstance.getHost()+":"+serviceInstance.getPort();
           }).collect(Collectors.toSet());
           //赋值
           this.serviceNames = newServiceNames;
           //清空老的
           oldServiceNames.clear();
       }
   
       @GetMapping("/say")
       private String say(@RequestParam String message){
   
           List<String> services = new ArrayList<>(this.serviceNames);
           //3. 进行负载均衡实现
           Random random = new Random();
           int i = random.nextInt(serviceNames.size());
   
           //4. 进行发送请求
          return  restTemplate.getForObject(services.get(i)+"/sayhello?message="+message,String.class);
       }
   
   }
   ~~~

4. 启动类配置

   ~~~java
   @SpringBootApplication
   @EnableDiscoveryClient//激活注册中心客户端
   @EnableScheduling  //激活定时任务
   public class ZkDemoApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(ZkDemoApplication.class, args);
   	}
   
   }
   ~~~

### 编写服务端

~~~java
@RestController
public class SayController {

    @Autowired
    private RestTemplate restTemplate;
    
    @GetMapping("/sayhello")
    public String sayhello(@RequestParam String message){
        System.out.println("say获取信息:"+message);
        return "hello";
    }

    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}

~~~

## 服务调用 增进

### 服务端应用

~~~java
@RestController
public class SayClientController {


    @Autowired
    private DiscoveryClient discoveryClient;


    @Autowired
    private RestTemplate restTemplate;

    @Value("${spring.application.name}")
    private String serviceName ;


    private volatile Map<String,Set<String>> serviceNames = new HashMap<>();


    @Scheduled(fixedDelay = 1000*30) //1分钟刷新一次
    private void flushServiceNames(){

        List<String> services = discoveryClient.getServices();

        services.forEach(s -> {

            List<ServiceInstance> instances = discoveryClient.getInstances(s);

            Set<String> collect = instances.stream().map(serviceInstance -> {
                return serviceInstance.isSecure() ?
                        "https://" + serviceInstance.getHost() + ":" + serviceInstance.getPort()
                        : "http://" + serviceInstance.getHost() + ":" + serviceInstance.getPort();
            }).collect(Collectors.toSet());

            serviceNames.put(s,collect);

        });



    }

    @GetMapping("/say/{servicename}")
    private String sayhello(@PathVariable String servicename ,
                            @RequestParam String message){
        List<String> servicenames = new ArrayList<>(this.serviceNames.get(servicename));
        Random random = new Random();
        int i = random.nextInt(serviceNames.size());
        return restTemplate.getForObject(servicenames.get(i)+"/service/say?message="+message,String.class);
    }





    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

~~~

### 客户端

~~~java
@RestController
public class SayServiceController {


    @GetMapping("/service/say")
    public String sayHello(String message){
        System.out.println("获取到信息:"+message);
        return "hello:"+message;
    }

}

~~~

## 服务调用 拦截器实现

### 拦截器

~~~java
public class LdConverter implements ClientHttpRequestInterceptor {

    @Autowired
    private DiscoveryClient discoveryClient;

    //存储服务名称
    private volatile Map<String,Set<String>> servicenames = new HashMap<String,Set<String>>();

    @Scheduled(fixedDelay = 10*1000)
    public void flushServiceName(){

        //这样写的好处是 在多线程的情况下，防止数据出现错误，所以先备份老的，然后在最后进行清除
        Map<String,Set<String>> olds = this.servicenames;
        Map<String,Set<String>> news =  new HashMap<String,Set<String>>();

        List<String> services = discoveryClient.getServices();
        services.forEach(servicename->{

            Set<String> newServiceNames = discoveryClient.getInstances(servicename).stream().map(serviceInstance -> {
                return serviceInstance.isSecure()?
                        "https://"+serviceInstance.getHost()+":"+serviceInstance.getPort():
                        "http://"+serviceInstance.getHost()+":"+serviceInstance.getPort();
            }).collect(Collectors.toSet());

            news.put(servicename,newServiceNames);
        });
        //替换
        servicenames = news;
        //清除
        olds.clear();

    }

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {

        //获取url
        URI uri = request.getURI();
        String path = uri.getPath();
        String[] split = StringUtils.split(path.substring(1),"/");
        String servicename = split[0];//服务名称
        String aa = split[1];
        //根据服务名字获取Ip和端口
        List<String> snames = new LinkedList<>(servicenames.get(servicename));
        Random random = new Random();
        int i = random.nextInt(snames.size());
        String iphost = snames.get(i);

        URL url = new URL(iphost+"/"+aa+"?"+uri.getQuery());

        URLConnection urlConnection = url.openConnection();
        InputStream inputStream = urlConnection.getInputStream();
        HttpHeaders httpHeaders = new HttpHeaders();

        return new SimpleClientHttpResponse(inputStream,httpHeaders);
    }

    public class SimpleClientHttpResponse extends AbstractClientHttpResponse{

        private InputStream inputStream;

        private HttpHeaders httpHeaders;

        public SimpleClientHttpResponse(InputStream inputStream, HttpHeaders httpHeaders) {
            this.inputStream = inputStream;
            this.httpHeaders = httpHeaders;
        }

        @Override
        public int getRawStatusCode() throws IOException {
            return 200;
        }

        @Override
        public String getStatusText() throws IOException {
            return "Ok";
        }

        @Override
        public void close() {

        }

        @Override
        public InputStream getBody() throws IOException {
            return this.inputStream;
        }

        @Override
        public HttpHeaders getHeaders() {
            return this.httpHeaders;
        }
    }
}
~~~

### 客户端

~~~java
 	@Autowired
    private RestTemplate restTemplate;

    @Autowired
    private RestTemplate ldRestTemplate;

    @GetMapping("/{servicename}/say")
    public String say(@PathVariable String servicename,
                       @RequestParam String message){
       return  restTemplate.getForObject("/"+servicename+"/sayhello?message="+message,String.class);
    }


    @GetMapping("/{servicename}/ld/say")
    public String ldsay(@PathVariable String servicename,
                       @RequestParam String message){
        return  ldRestTemplate.getForObject("/"+servicename+"/sayhello?message="+message,String.class);
    }


~~~

### 服务端

~~~java
@RestController
public class SayController {

    @GetMapping("/sayhello")
    public String sayhello(@RequestParam String message){
        System.out.println("say获取信息:"+message);
        return "hello";
    }

}
~~~

### applicationContext.java

~~~java
@SpringBootApplication
@EnableDiscoveryClient
@EnableScheduling
public class ZkDemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(ZkDemoApplication.class, args);
	}

	@Bean
	@Autowired
	@Qualifier //这个地方的restTemplate被Qualifier标志，下方的注入则可以获取到
	public RestTemplate restTemplate(LdConverter ldConverter){
		RestTemplate restTemplate = new RestTemplate();
		//添加拦截器
		restTemplate.setInterceptors(Arrays.asList(ldConverter));
		return restTemplate;
	}

	@Bean
	public RestTemplate ldRestTemplate(){
		return new RestTemplate();
	}

	@Bean
	public LdConverter ldConverter(){
		return new LdConverter();
	}

    //获取所有的resttemplate进行追加自定义的拦截器
	@Bean
	@Autowired
    //@Qualifier 注解过的restTemplates参数，在注入的时候，会进行过滤只有被@Qualifier的类
	public Object myaaa(@Qualifier Collection<RestTemplate>  restTemplates,LdConverter ldConverter){
		restTemplates.forEach(r->{
			r.setInterceptors(Arrays.asList(ldConverter));
		});
		return new Object();
	}

}
~~~



> **`@Qualifier`**
>
>  `@LoadBalanced`  里面有@Qualifier,利用注解来过滤，注入法和声明适用方必须同时使用
>
> ~~~java
> /**
>  * Annotation to mark a RestTemplate bean to be configured to use a LoadBalancerClient
>  * @author Spencer Gibb
>  */
> @Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
> @Retention(RetentionPolicy.RUNTIME)
> @Documented
> @Inherited
> @Qualifier
> public @interface LoadBalanced {
> }
> 
> ~~~



## 自定义限流

### aop实现

~~~java
@Aspect
@Component
public class TimeRoundAspect {

    private ExecutorService executor = Executors.newFixedThreadPool(10);
    //拦截指定的方法
    @Around("execution(* com.cloud.zk.zkdemo.HelloController.say(..)) && args(servicename,message)")
    public Object timeOutAspect(ProceedingJoinPoint proceedingJoinPoint,String servicename,String message) throws Throwable {
        //1.使用future 进行超时拦截
        Future<Object> future = executor.submit(()->{
            Object proceed = null;
            try {
                //参数透传
                proceed = proceedingJoinPoint.proceed(new String[]{servicename,message});
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
            return proceed;
        });
        Object proceed = null;
        try {
            proceed = future.get(100, TimeUnit.MILLISECONDS);//100m等待
        }catch (TimeoutException exception){
            future.cancel(true);//取消执行
            proceed = "Fault";
        }

        return proceed;

    }

    //生命周期最后执行的
    @PreDestroy
    public void destory(){
        executor.shutdown();
    }

}
~~~

### aop+注解实现（ @annotation）

注解实现

~~~java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CustomCurrentLimiter {

    /**
     * 设置超时时间
     * @return 默认10ms
     */
    int timeout() default 100;


}
~~~

aop实现

~~~java

@Aspect
@Component
public class TimeRoundAspect {

    private ExecutorService executor = Executors.newFixedThreadPool(200);
    //拦截指定的方法
    @Around("execution(* com.cloud.zk.zkdemo.HelloController.say(..)) && args(servicename,message) && @annotation(CustomCurrentLimiter)")
    public Object timeOutAspect(ProceedingJoinPoint proceedingJoinPoint,String servicename,String message,
                                CustomCurrentLimiter customCurrentLimiter) throws Throwable {
        //获取customCurrentLimiter时间
        int timeout = customCurrentLimiter.timeout();
        //1.使用future 进行超时拦截
        Future<Object> future = executor.submit(()->{
            Object proceed = null;
            try {
                proceed = proceedingJoinPoint.proceed(new String[]{servicename,message});
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
            return proceed;
        });
        Object proceed = null;
        try {
            proceed = future.get(timeout, TimeUnit.MILLISECONDS);//200m等待
        }catch (TimeoutException exception){
            future.cancel(true);//取消执行
            proceed = "Fault";
        }

        return proceed;

    }

    //生命周期最后执行的
    @PreDestroy
    public void destory(){
        executor.shutdown();
    }

}
~~~

### aop+注解实现（ 反射）

**注解实现**

```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CustomCurrentLimiter {

    /**
     * 设置超时时间
     * @return 默认10ms
     */
    int timeout() default 100;


}
```

**aop实现**

~~~java

@Aspect
@Component
public class TimeRoundAspect {

    private ExecutorService executor = Executors.newFixedThreadPool(200);
    //拦截指定的方法
    @Around("execution(* com.cloud.zk.zkdemo.HelloController.say(..)) && args(servicename,message) ")
    public Object timeOutAspect(ProceedingJoinPoint proceedingJoinPoint,String servicename,String message) throws Throwable {
        long timeout = -1;
        if (proceedingJoinPoint instanceof MethodInvocationProceedingJoinPoint) {
            MethodInvocationProceedingJoinPoint methodPoint = (MethodInvocationProceedingJoinPoint) proceedingJoinPoint;
            MethodSignature signature = (MethodSignature) methodPoint.getSignature();
            Method method = signature.getMethod();
            CustomCurrentLimiter circuitBreaker = method.getAnnotation(CustomCurrentLimiter.class);
            timeout = circuitBreaker.timeout();
        }

        //1.使用future 进行超时拦截
        Future<Object> future = executor.submit(()->{
            Object proceed = null;
            try {
                proceed = proceedingJoinPoint.proceed(new String[]{servicename,message});
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
            return proceed;
        });
        Object proceed = null;
        try {
            proceed = future.get(timeout, TimeUnit.MILLISECONDS);//200m等待
        }catch (TimeoutException exception){
            future.cancel(true);//取消执行
            proceed = "Fault";
        }

        return proceed;

    }

    //生命周期最后执行的
    @PreDestroy
    public void destory(){
        executor.shutdown();
    }

}
~~~

### aop +注解实现（@annotation)+ 信号灯

```java
//拦截指定的方法
@Around("execution(* com.cloud.zk.zkdemo.HelloController.say(..)) && args(servicename,message) &&@annotation(CustomCurrentLimiter)")
public Object timeOutAspectSemaphore(ProceedingJoinPoint proceedingJoinPoint,String servicename,String message,CustomCurrentLimiter customCurrentLimiter) throws Throwable {

    //单机版 信号灯的实现
    Semaphore semaphore = null;
    if(semaphore==null){
        semaphore = new Semaphore(customCurrentLimiter.timeout());
    }
    Object returnValue = null;
   try{
       if(semaphore.tryAcquire()) {
           Object proceed = proceedingJoinPoint.proceed(new String[]{servicename, message});
       }else{
           returnValue = "Fault";
       }
   }catch (TimeoutException exception){
       returnValue = "Fault";
   }finally {
       semaphore.release();
   }

    return returnValue;

}
```

## 服务调用

### 自实现feign

+ `FeignClient`注解 
  + `@GetMapping()` 接口请求

+ `EnableFeignClients` 注解

## Stream

**Source 来源**

`@PostConstruct`//接口編程

`@StreamListener("channel_name")` //注解驱动編程

`ServiceActivator("channel_name")`//Spring Interation 注解驱动编程

> 如果都写轮流执行



## Spring cloud bus

### JAVA 事件/监听者模式

#### 事件

+ 所以的事件类型扩展`java.util.Event`
+ 

#### 事件监听器

+ 事件监听器接口要扩展`java.util.EventListener`

+ 方法参数类型
  + 参数是`java.util.Event`对象（子类）
  + 事件源
    + Jpa `EntityListener`
      + `@Entity`
      + `EntityManager` persist(Object)

+ 监听方法没有限定符（public)

+ 监听方法是没有返回值(void)

  + 例外：Spring `@EnventListener`

+ 监听方法不会`throws` `Throwable`

   

##### 事件说明：GUI桌面

### Spring 事件

+ Spring事件基类`ApplicationEvent`
  + 相对于`java.util.Event`来说 多了个`timestamp`事件发生时间戳的属性



### Spring Boot事件

+ 



### Spring cloud 事件



### Event Bus(事件总线)