# spring 模块

+ 基础模块,提供spring的基础功能
  + spring-core 
    + IoC container IOC 容器
      + org.springframework.beans
      + org.springframework.context
    + Events 事件监听
    + Resources 资源管理
    + i18n 国际化
    + Validation 校验
    + Data Binding 数据绑定
    + Type Conversion  类型转换
    + SpEL 表达式 过滤信息
    + AOP 
  + spring-beans 

+ 上下文模块，基于基础模块实现
  + spring-context  类似于一个JNDI注册表
    + 国际化
    + 事件传播
    + 资源负载
    + 透明创建上下文（servlet容器)
    + EJB
    + JMX 
    + spring-context-support 支持整合普通第三方库
      + 高速缓存（ehcache,Jcache
      + 调度（CommonJ,Quartz)
+ 表达式语言模块 spring-expression 
  + 设置和获取属性值
  + 属性分配
  + 方法调用
  + 访问数组，集合，索引器的内容
  + 逻辑和算数运算
  + 变量命名以及从Spring的IoC容器中以名称检索对象
  + 持列表投影和选择以及常见的列表聚合

+ AOP 
  + spring-aop AOP 切面
    + spring-aspects 
      + 对AspectJ的集成
  + spring-instrument 类植入
    + spring-instrument-tomcat  支持Tomcat 的植入代理
+ 消息 
  + spring-messaging  消息传递模块
    + Message
    + MessageChannel
    + MessageHandler
    + 其他用来传输消息的基础应用
    + 注解编程模型（消息映射到方法）
+ 数据访问/集成
  + JDBC
    + spring-jdbc  减少了jdbc复杂的代码，并且整合了各个数据库厂商特有的错误代码解析
  + ORM（object-relational mapping）
    + spring-orm 可以整合处理对象关系映射的插件
      + JPA
      + Hibernate
      + 简单声明性事务管理
  + OXM
    + spring-oxm 支持对象/XML映射实现的抽象层
      - JAXB
      - Castor
      - Jibx
      - xStream
  + JMS （(Java Messaging Service） 生产和消费消息的功能
    + spring-jms
      + spring-messaging （spring framework4.1开始支持）
  + 事务
    + spring-tx
+ WEB
  + spring - web 
    + 
  + spring -webmvc
    + 
  + spring-websocket
    + 



## IoC container（Inversion of Control）

IOC：控制反转  \ DI:依赖注入

### 说明：

​	传统模式： 我们需要什么依赖对象的时候，需要手动去new，然后注入到对象中

​	通过spring容器：把创建依赖对象权利交给了spring 容器，具体使用只要去找spring 要就可以了，不需要考虑对象是什么时候创建的，怎么创建的。

​	最终就是：依赖对象的方式发生了反转



### 具体实现类

 +  基础
     +  org.springframework.beans 
     +  org.springframework.context

+ `org.springframework.beans.factory.BeanFactory`  
  + `org.springframework.context.ApplicationContext`（责实例化、配置、组装beans）
    + `org.springframework.context.annotation.AnnotationConfigApplicationContext`
    + `org.springframework.context.support.ClassPathXmlApplicationContext`（xml形式加载）
    + `org.springframework.context.support.FileSystemXmlApplicationContext`(文件形式加载)

![1546846978566](https://wanglei.club/beanFactory类图.png)

+  配置元数据
  +  基于注解配置：
  +  基于java配置
        + @Configuration
        + @Bean
        + @Import
        + @DependsOn等等





1.    @Configuration/@Componen 和 @Bean使用的区别 

   ~~~java
   @Component/@Configuration
   public class AppConfig {
       @Bean
       public StudentService studentService(){
           StudentService studentService = new StudentService();
           System.out.println("studentService:"+ studentService.hashCode());
           return studentService;
       }
       @Bean
       public SchoolService schoolService(){
           StudentService studentService = studentService();
           System.out.println("schoolService:"+ studentService.hashCode());
           return new SchoolService(studentService);
       }
   }
   ~~~

   > 在@Component下，调用studentService() 会重新new一遍   称为 lite 模式
   >
   > 在@Configuration下，调用studentService() 不会重新new一遍

2. @Inject、@Autowired、@Resource区别

+ @Inject

  + `javax.inject.Inject` jdk包下
  + 注入的时候看name是否相同 可以和`@Name`搭配使用

+ @Autowired

  + `org.springframework.bean.factory.Autowired` spring包下

  + 默认是属性name来注入的，如果设置了@Qualifier注解，那么会按照@Qualifier的name属性进行注入

    ~~~java
    @Bean
    @Qualifier("schoolService2")
    public SchoolService schoolService(){
        StudentService studentService = studentService();
        System.out.println("schoolService:"+ studentService.hashCode());
        return new SchoolService(studentService);
    }
    //---------------------------依赖注入------------------------
    @Autowired
    @Qualifier("schoolService2")
    private SchoolService schoolService;
    
    ~~~

  + 可以设置参数required 如果为false那么没有注入成功就是不报错，为true就报错 ,默认是true

    ~~~java
    @Autowired(required = false)
    @Qualifier("schoolService2")
    private SchoolService schoolService;
    ~~~

+ @Resource

  + `javax.annotation.Resource`   jdk包下

  +  一般需要设置name属性 

    ~~~java
    @Resource(name = "userMapper")
    private UserMapper userMapper;
    ~~~

  + 一般是先看name是否能匹配到，如果没匹配到按照type类型来找



4. 生命周期

   + 执行Bean的构造器
     + `BeanFactoryAware`   读取Bean定义文件，生产各个实例
   + 为Bean注入属性 初始化bean
     + 方式1` InitializingBean  ` 初始化Bean的信息
     + 方式2 `@PostConstruct`
     + 方式3 `@Bean(initMethod = "init")`

   + 运行

   + 销毁bean
     + 方式1 `DiposableBean `  容器关闭后后销毁Bean
     + 方式2 `@PreDestroy`
     + 方式3 `@Bean(destroyMethod = "cleanup")`



5.  *Aware 

   + `BeanFactoryAware `
   + `BeanNameAware`
   + `MessageSourceAware`
   + `ApplicationContextAware` 等


6. 指定bean的作用域 @Scope 
   + @Scope("prototype") 
     + 单例
   + @Scope("request") 
     + 针对每一次HTTP请求都会产生一个新的bean，同时该bean仅在当前HTTP request内有效
   + @Scope("session")
     + 针对每一次HTTP请求都会产生一个新的bean，同时该bean仅在当前HTTP session内有效
   + @Scope("global session")
     + 类似于标准的HTTP Session作用域，不过它仅仅在基于portlet的web应用中才有意义

7. @Bean  属性

   + name 可以取一个名字，也可以取多个名字

     ~~~java
     @Bean(name = { "dataSource", "subsystemA-dataSource", "subsy
     stemB-dataSource" })
     ~~~

8. @Description 添加描述 

   ~~~java
   @Description("Provides a basic example of a bean")
   ~~~



9. @Profile  用来标明当前运行环境的注解

   ~~~java
   @Bean
   @Profile("prod")
   public StudnetProd studnetProd(){
       return new StudnetProd();
   }
   
   @Bean
   @Profile("test")
   public StudnetTest studnetTest(){
       return new StudnetTest();
   }
   ~~~

   > Profile 可以放在类上，也可以声明在方法级别上
   >
   > 也可以同时设置多个激活场景 @Profile({"test","prod"}) 
   >
   > @Profile({"test","!prod"})  这样的意思是：test为激活时执行 或者prod没激活的时候执行

   **激活场景**

   1) 代码激活

   ~~~java
   AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext();
   ...       annotationConfigApplicationContext.getEnvironment().setActiveProfiles("test","prod");
   ...
   ~~~

   2）JVM命令激活

   ~~~properties
   -Dspring.profiles.active=test,prod
   ~~~

   **默认profile** @Profile("default")

   当激活了任意的profile，那么这个不会执行

   当没有激活profile,则会执行 

   > 修改profile默认的名称
   >
   > 1. 上下文中使用setDefaultProfiles()
   >
   >    ~~~java
   >    annotationConfigApplicationContext.getEnvironment().setDefaultProfiles("default");
   >    ~~~
   >
   > 2. 通过spring 属性 来修改
   >
   > ~~~properties
   > spring.profiles.default=...
   > ~~~

10. @Conditiona  条件注解

     ~~~java
    public class StudnetProd  implements Condition {  // 继承Condition
    
private String name ="生产";
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
      
     ~~~
    

   ~~~java
@Override
public String toString() {
    return "StudnetTest{" +
        "name='" + name + '\'' +
        '}';
}
public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    return  false;
}
   ~~~



```java
public class StudnetTest implements Condition {  // 继承Condition

    private String name ="测试";

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
    @Override
    public String toString() {
        return "StudnetTest{" +
            "name='" + name + '\'' +
            '}';
    }
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return true;
    }
}
```


   
```java
@Bean
@Conditional(StudnetTest.class)  //当matches 返回true的时候 创建bean
public StudnetTest studnetTest(){
    return new StudnetTest();
}  

@Bean
@Conditional(StudnetProd.class) //当matches 返回true的时候 创建bean
public StudnetProd studnetProd(){
    return new StudnetProd();
}
```

    1)   继承Condition
    
    2)  在生成@Bean的添加注解@Conditional(StudnetTest.class) 

11. @ImportResource  加载xml资源文件

    **spring-context.xml**

    ~~~xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:context="http://www.springframework.org/schema/context"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context.xsd" >
    
    
            <context:property-override location="classpath:db.properties"/>
    
            <bean id="dbConfig" class="com.wl.spring.DbConfig"/>
    </beans>
    ~~~

    **db.properties**

    ~~~properties
    //dbConfig 是bean的名称  name是bean的属性
    dbConfig.name = zhangsna
    ~~~

    **DbConfig**

    ~~~java
    public class DbConfig {
    
        private String name;
        
        public String getName() {
            return name;
        }
    
        public void setName(String name) {
            this.name = name;
        }
    
        @Override
        public String toString() {
            return "DbConfig{" +
                    "name='" + name + '\'' +
                    '}';
        }
    
    }
    ~~~

**ConfigurationBean**

```java

@Configuration
@ImportResource("classpath:spring-context.xml")
public class ConfigurationBean {}

```





12. 属性源抽象

    > System.getProperties() 是JVM的系统属性
    >
    > System.getenv() 是系统环境变量

    **默认所有的属性资源在 `org.springframework.core.env.StandardEnvironment`**

    通过`Environment.getProperties("")`在查询的时候 系统属性 > 系统环境变量

13. @PropertiesSource 添加配置文件

    同 ImportResource 

14. @EnableLoadTimeWeaving  代码织入

    代码织入分为3种情况

    + 代码织入
      + 编译期织入
      + 类加载期织入
      + 运行期织入

    AspectJ 采用的是编译期织入、类加载期织入的方式

    @EnableLoadTimeWeaving 采用的就是类加载期织入

15. 标准和自定义事件

    ApplicationEvent

    ApplicationListener 

    1) 定义事件  内置事件

    + `org.springframework.context.event.ContextRefreshedEvent` 
    + `org.springframework.context.event.ContextClosedEvent`
    + `org.springframework.context.event.ContextStartedEvent`
    + `org.springframework.context.event.ContextStoppedEvent`
    + `RequestHandlerEvent`

    2.1) 创建自定义事件 继承`org.springframework.context.ApplicationEvent`

    ~~~java
    public class MyEvent extends ApplicationEvent {
    
        //消息
       private final String message;
    
        public MyEvent(Object source, String message) {
            super(source);
            this.message = message;
        }
    
    }
    ~~~

    2.2) 注册事件  实现` org.springframework.context.ApplicationEventPublisherAware`接口

    ~~~java
    public class ConfigurationBean  implements ApplicationEventPublisherAware {
        
        private ApplicationEventPublisher applicationEventPublisher;
    
        public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
            this.applicationEventPublisher = applicationEventPublisher;
        }
    
    ~~~

16. 事件监听 `org.springframework.context.ApplicationListener`

    > 事件都支持返回参数，但是当被@Aysnc注解 的时候，就不支持了。

    1) 代码实现

    ~~~java
    public class MyListener implements ApplicationListener<MyEvent> {
        public void onApplicationEvent(MyEvent event) {
            System.out.println("获取到消息:"+event.getMessage());
        }
    }
    ~~~

    2）注解实现 `@EventListener`

    ~~~java
    @EventListener
    public void shouMessage(MyEvent message){
        System.out.println("eventListener:"+message.getMessage());
    }
    ~~~

    ​	2.1) 属性condition  过滤一些条件 当系统属性中myname不存在的时候就执行

```java
@EventListener(condition = "#systemEnvironment.get('myname')==null" )
public void shouMessage(String message){
    System.out.println("bbbb");
}
```

​		2.2） 其他介绍

![image-20190108114649146](https://wanglei.club/其他介绍.png)

​	3） 顺序的监听器`@Order` 设置优先级

~~~java
@Order(1)
@EventListener(condition = "#systemEnvironment.get('myname')==null" )
public void shouMessage(String message){
    System.out.println("bbbb");
}
~~~

​	4） 泛型事件

​	

17. 应用上下文 ResourceLoader  -> `org.springframework.context.ResourceLoaderAware`

    **获取不同类型的Resource**

    >  所有上下文都实现了ResourceLoader，所以可以直接调用getResource()获取

    + ```java
      //resource = 返回applicationContext的声明类型的Resource
      Resource resource = applicationContext.getResource("application-context.xml");
      ```

    + ~~~java
      // resource = FileSystemResource
      Resource resource = applicationContext.getResource("file:application-context.xml");
      ~~~

    + ~~~java
      //resource = ClassPathResource 
      Resource resource = applicationContext.getResource("classpath:application-context.xml");
      ~~~

    + ~~~java
      // resource = UrlResource
      Resource resource = applicationContext.getResource("http:application-context.xml");
      ~~~

    ![1546938519972](https://wanglei.club/spring文件加载条件.png)

    **代码使用ResourceLoaderAware**

    ​	通过实现`org.springframework.context.ResourceLoaderAware`,可以把ResourceLoader 添加到本类进行使用

    **注解使用ResourceLoaderAware**

    ​	@Autowring

18. 资源加载

+ Resource 
  + org.springframework.core.io.UrlResource
    + 封装了java.net.URI 对象，
      + Http target
      + FTP target
      + 文件
  + org.springframework.core.io.FileSystemResource
  + org.springframework.core.io.ClassPathResource
  + ServletContextResource
  + org.springframework.core.io.InputStreamResource 
  + org.springframework.core.io.ByteArrayResource

19.  Bean 和 BeanWrapper

    + Bean ->`org.springframework.beans`

    + BeanWrapper  

      + 设置和获取属性值(单独或批量)、获取属性描述符以及查询属性以确定它们是可读还是可写的功能

    + BeanWrapperImpl 代码编程

      + ~~~java
        //管理对象
        BeanWrapper beanWrapper = new BeanWrapperImpl(new Message());
        //设置参数
        beanWrapper.setPropertyValue("message","123");
        //获取属性的详细信息
        PropertyDescriptor message = beanWrapper.getPropertyDescriptor("message");
        ~~~

20. 类型转换

+ PropertyEditor 
  - ByteArrayPropertyEditor 
    - 针对字节数组的编辑器。字符串会简单地转换成相应的字节表示。默认情况下由 BeanWrapperImpl 注册。
  - ClassEditor
    - 将类的字符串表示形式解析成实际的类形式并且也能返回实际类的字符串表示形式。如果找不到类，会抛出一个 IllegalArgumentException 。默认情况下由 BeanWrapperImpl 注册。
  - CustomBooleanEditor
    - 针对 Boolean 属性的可定制的属性编辑器。默认情况下由 BeanWrapperImpl 注册，但是可以作为一种自定义编辑器通过注册其自定义实例来进行覆盖。
  - CustomCollectionEditor
    - 针对集合的属性编辑器，可以将原始的 Collection 转换成给定的目标 Collection 类型。
  - CustomDateEditor
    - 针对java.util.Date的可定制的属性编辑器，支持自定义的时间格式。不会被默认注册，用户必须使用适当格式进行注册。
  - CustomNumberEditor
    - 针对任何Number子类(比如 Integer 、 Long 、 Float 、 Double )的可定制的属性编辑器。默认情况下由 BeanWrapperImpl 注册，但是可以作为一种自定义编辑器通过注册其自定义实例来进行覆盖。
  - FileEditor
    - 能够将字符串解析成 java.io.File 对象。默认情况下由 BeanWrapperImpl 注册。
  - InputStreamEditor
    - 一次性的属性编辑器，能够读取文本字符串并生成(通过中间的 ResourceEditor 以及 Resource )一个 InputStream 对象，因此 InputStream 类型的属性可以直接以字符串设置。请注意默认的使用方式不会为你关闭 InputStream ！默认情况下由 BeanWrapperImpl 注册。
  - LocaleEditor
    - 能够将字符串解析成 Locale 对象，反之亦然(字符串格式是[country][variant]，这与Locale提供的toString()方法是一样的)。默认情况下由 BeanWrapperImpl 注册。
  - PatternEditor
    - 能够将字符串解析成 java.util.regex.Pattern 对象，反之亦然。
  - PropertiesEditor
    - 能够将字符串(按照 java.util.Properties 类的java文档定义的格式进行格式化)解析成 Properties 对象。默认情况下由 BeanWrapperImpl 注册。
  - StringTrimmerEditor
    - 用于缩减字符串的属性编辑器。有选择性允许将一个空字符串转变成 null 值。不会进行默认注册，需要在用户有需要的时候注册。
  - URLEditor
    - 能够将一个URL的字符串表示解析成实际的 URL 对象。默认情况下由 BeanWrapperImpl 注册。

​		

21. SPEL  (基础类，自定义的不可用)

+ String 
  + 调用String 的方法

1. ~~~java
   ExpressionParser expressionParser = new SpelExpressionParser();
   Expression expression = expressionParser.parseExpression("'Hello world'.equals('!')");
   Boolean result = (Boolean) expression.getValue();
   System.out.println(result); // false
   ~~~

   > 可以获取字符串的所有方法

   +  调用String 的属性get方法  getBytes() 

   ~~~java
   ExpressionParser expressionParser = new SpelExpressionParser();
   Expression expression = expressionParser.parseExpression("'Hello world'.bytes");
   byte[] result = (byte[]) expression.getValue();
   System.out.println(result); //[B@123ef382
   ~~~

   + 可以级联调用

   ~~~java
   ExpressionParser expressionParser = new SpelExpressionParser();
   Expression expression = expressionParser.parseExpression("'Hello world'.bytes.length");
   Integer result = (Integer) expression.getValue();
   System.out.println(result); //11
   ~~~

   + 调用String的构造函数

   ~~~java
   ExpressionParser expressionParser = new SpelExpressionParser();
   Expression expression = expressionParser.parseExpression("new String('hello world')");
   String result = (String) expression.getValue();
   System.out.println(result); //hello world
   ~~~

   + **对于泛型方法的调用!!!!!!!!!!!!!!!!**

   ~~~java
   
   ~~~

+ 编译器配置

  > 默认是没开启编译器模式的，需要手动打开
  >
  > OFF:编译器关闭状态
  >
  > IMMEDIATE: 即时生效，表达式会尽快编译，如果出错，(表达式求值的调用点)就会抛出异常
  >
  > MIXED:混合模式，会自动在解释器模式和编译器模式之间来回切换，如果中间出现错误，则会切换到解释器模式，并且会吞掉错误，内部处理掉

  + 代码打开 SpelParserConfiguration

    ~~~java
    SpelParserConfiguration spelParserConfiguration  = new SpelParserConfiguration(SpelCompilerMode.IMMEDIATE,WlSpringContext.class.getClassLoader());//off/immediate/mixed
    
    ExpressionParser expressionParser = new SpelExpressionParser(spelParserConfiguration);
    Expression expression = expressionParser.parseExpression("new DbConfig('123')");
    Object result = expression.getValue();
    System.out.println(result);
    
    ~~~

  + 系统环境变量设置

    ~~~properties
    spring.expression.compiler.mode = off/immediate/mixed
    ~~~

  + 局限性

    + 不支持所有类型的表达式

    + 以下表达式不支持编译

      + 涉及到赋值 的表达式
      + 依赖于转换服务的表达式
      + 使用到自定义解析器或者存取器的表达式
      + 使用到选择器或者投影的表达式

    + 支持：

      + 支持字符串表达式类型

      + 数值类型（int,eeal,hex)  支持负数，指数，小数点  ,默认是使用Double.parseDouble()解析

        ~~~java
        ExpressionParser expressionParser = new SpelExpressionParser(spelParserConfiguration);
        Integer result = (Integer) expressionParser.parseExpression("1").getValue();
        ~~~

      + 布尔

      + 数组

      + 方法调用

      + 运算符

        ~~~java
        ExpressionParser expressionParser = new SpelExpressionParser(spelParserConfiguration);
        Boolean value = expressionParser.parseExpression("1==1").getValue(Boolean.class);
        ~~~

      + null.strings

      + 逻辑运算符 and ,or,!

        ~~~java
        ExpressionParser expressionParser = new SpelExpressionParser(spelParserConfiguration);
        //and
        Boolean value = expressionParser.parseExpression("true and false").getValue(Boolean.class);
        
        !or
        Boolean value = expressionParser.parseExpression("true or false").getValue(Boolean.class);
        
        //!
        Boolean value = expressionParser.parseExpression("!false").getValue(Boolean.class);
        ~~~

      + 算数运算符  减号，乘号，除法只能用于数字   / 取模（%）和指数（^）

      ~~~java
      Integer value = expressionParser.parseExpression("1+1").getValue(Integer.class);//2
      
      String value = expressionParser.parseExpression("'a'+'b'").getValue(String.class);//ab
      
      ...
      ~~~

      + 赋值

        1）方式1

        ~~~java
        
        DbConfig dbConfig = new DbConfig();
        ExpressionParser expressionParser = new SpelExpressionParser(spelParserConfiguration);
        StandardEvaluationContext dbConfigContext = new StandardEvaluationContext(dbConfig);
        expressionParser.parseExpression("name").setValue(dbConfigContext,"张三");
        String name = expressionParser.parseExpression("name").getValue(dbConfigContext, String.class);
        System.out.println(name);
        ~~~

        2） 方式2

        ~~~java
        DbConfig dbConfig = new DbConfig();
        ExpressionParser expressionParser = new SpelExpressionParser(spelParserConfiguration);
        StandardEvaluationContext dbConfigContext = new StandardEvaluationContext(dbConfig);
        String value1 = expressionParser.parseExpression("name = '张三2'").getValue(dbConfigContext, String.class);
        System.out.println(value1);
        ~~~

      + 类型   T操作符   

        > java.lang 下的类可以不指定全路径
        >
        > 其他包下的需要指定

        String类型

        ~~~java
        ExpressionParser expressionParser = new SpelExpressionParser(spelParserConfiguration);
        Class c = expressionParser.parseExpression("T(String)").getValue(Class.class);//class java.lang.String
        ~~~

        Date类型

        ~~~java
        ExpressionParser expressionParser = new SpelExpressionParser(spelParserConfiguration);
        Class d = expressionParser.parseExpression("T(java.util.Date)").getValue(Class.class);//class java.util.Date
        ~~~

      + 构造器 

        > 除了基础类型（int,double,float等） 和String 不需要全路径名
        >
        > 其他的都需要

        ~~~java
        ExpressionParser expressionParser = new SpelExpressionParser(spelParserConfiguration);
        DbConfig dc = expressionParser.parseExpression("new com.wl.spring.DbConfig()").getValue(DbConfig.class);//DbConfig{name='null'}
        ~~~

      + 变量

        > 表达式中的变量可以通过 #变量名 使用 在StandardEvaluationContext中通过setVariable设置

        ~~~java
        DbConfig dbConfig = new DbConfig();
        
        StandardEvaluationContext dbConfigEvaluationContext = new StandardEvaluationContext(dbConfig);
        
        dbConfigEvaluationContext.setVariable("newName","李四");
        
        ExpressionParser parser = new SpelExpressionParser();
        parser.parseExpression("name = #newName").getValue(dbConfigEvaluationContext);
        System.out.println(dbConfig.getName());
        ~~~

      + `#this` 和`#root`变量

      ~~~java
      StandardEvaluationContext dbConfigEvaluationContext = new StandardEvaluationContext();
      ExpressionParser parser = new SpelExpressionParser();
      
      List<Integer> prims = new ArrayList<Integer>();
      prims.addAll(Arrays.asList(2,3,4,5,6,7));
      
      dbConfigEvaluationContext.setVariable("prims",prims);
      //获取到大于5的数据
      List<Integer> value = (List<Integer>) parser.parseExpression("#prims.?[#this>5]").getValue(dbConfigEvaluationContext);
      
      System.out.println(value);
      ~~~

    + 函数

    + Bean应用

    + 三元运算符

    + Elvis运算符

    + 安全引用运算符

    + 集合筛选

    + 集合投影

    + 表达式模板

    + 

22. Bean定义时使用表达式

    1. 基于注解 #{}

    ~~~java
    @Value("#{systemProperties['dbConfig.name']}")  
    ~~~

    ​	1.1 可以定义在属性上

    ~~~java
    @Value("#{systemProperties['dbConfig.name']}")
    private String name;
    ~~~

    ​	1.2 也可以定义在set方法上

    ~~~java
    @Value("#{systemProperties['dbConfig.name']}")
    public void setName(String name) {
    	this.name = name;
    }
    ~~~

    ​	1.3 使用@Autowired注解的方法或者构造器也可以使用@Value注解

    ~~~java
    @Autowired
    public DbConfig(@Value("#{systemProperties['dbConfig.name']}") String name) {
    	this.name = name;
    }
    ~~~




23. 