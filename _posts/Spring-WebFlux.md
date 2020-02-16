---
title: Spring MVC
date: 2018-12-11 22:07:36
tags: spring
---



## Spring WebFlux

### Reactive原理

- reactive 是异步非阻塞线程 （错误的）
  - reactive是同步/异步非阻塞编程
- reactive能够提升程序性能
  - 大多数情况是没有的，少数可能会有
  - https://blog.ippon.tech/spring-5-webflux-performance-tests/
- reactive解决传统编程模型遇到的困境
  - 错误的。传统困境不需要，也不能被reactive解决





## 订阅者模式：

```java
Observable
```



## 事件/监听模式

### Spring

- `java.util.EventObject` ：事件对象
  - `java.util.EventListener`  : 事件监听接口（标记）

### Spring boot

- `ApplicationEvent` 监听事件
  - `ApplicationListener` 事件监听接口
- `ApplicationEnvironmentPreparedEvent`
- `ConfigFileApplicationListener`



#### `ConfigFileApplicationListener` 

管理配置文件，比如：`application.properties`以及`application.yml`

`application-{profile}.properties`

profile = dev 、test

优先于application.properties/application.yml



**spring-boot : spring-boot-2.1.0.RELEASE.jar!\META-INF\spring.factories**

```properties
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
```



**spring-cloud : spring-cloud-context-1.3.0.RELEASE.jar!\META-INF\spring.factories**

```properties
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.cloud.bootstrap.BootstrapApplicationListener,\
org.springframework.cloud.bootstrap.LoggingSystemShutdownListener,\
org.springframework.cloud.context.restart.RestartListener
```



> 加载优先级的时候，配置的信息一定要优先于`ConfigFileApplicationListener`,否则在`application.properties`中配置了也没有效果

> 原因：
>
> ​	因为bootstarp是最先加载的，而applicationContext是后加载，并且是bootstarp的子上下文，父的上下文是访问不到子的上下文信息的
>
> ​	`BootstrapApplicationListener`  第6优先
>
> ​	`ConfigFileApplicationListener` 第11优先

###### 扩展Linstener

1. 再配置文件spring.factories中添加
2. 设置优先级
3. 可以通过实现`Order`以及标记`@Order` 来控制谁先加载

#### BootstrapApplicationListener

1. 负责加载`bootstrap.properties` 或者`bootstrap.yaml`
2. 负责初始化Bootrap ApplicationContent ID = "bootstrap"

```java
ConfigurableApplicationContext context = builder.run();
```

Bootstrap是一个根Spring上下文，parent=null

> 联想ClassLoader:
>
> ExtClassLoader<-AppClassLoader<-System ClassLoader->Bootstrap Classloader(null)



spring 重要方法 

org.springframework.context.support.AbstractApplicationContext#refresh()

### /env

### 配置顺序

[Spring 官方文档](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#boot-features-external-config)

1. [Devtools global settings properties](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#using-boot-devtools-globalsettings) on your home directory (`~/.spring-boot-devtools.properties` when devtools is active).

   Devtools 热部署的配置信息，当devtools处于活动状态时

2. [`@TestPropertySource`](https://docs.spring.io/spring/docs/5.1.2.RELEASE/javadoc-api/org/springframework/test/context/TestPropertySource.html) annotations on your tests.

   测试上的注释

3. `properties` attribute on your tests. Available on [`@SpringBootTest`](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/api/org/springframework/boot/test/context/SpringBootTest.html) and the [test annotations for testing a particular slice of your application](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#boot-features-testing-spring-boot-applications-testing-autoconfigured-tests).

   属性测试。可 [用于测试特定应用程序片段](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#boot-features-testing-spring-boot-applications-testing-autoconfigured-tests)[`@SpringBootTest`](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/api/org/springframework/boot/test/context/SpringBootTest.html)的 [测试注释](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#boot-features-testing-spring-boot-applications-testing-autoconfigured-tests)。

4. Command line arguments.

   命令行参数

5. Properties from `SPRING_APPLICATION_JSON` (inline JSON embedded in an environment variable or system property).

   来自`SPRING_APPLICATION_JSON`（嵌入在环境变量或系统属性中的内联JSON）的属性。

6. `ServletConfig` init parameters.

   ServletConfig的init参数

7. `ServletContext` init parameters.

   ServletContext的init参数。

8. JNDI attributes from `java:comp/env`.

   JNDI属性来自`java:comp/env`

9. Java System properties (`System.getProperties()`).

   Java系统属性（`System.getProperties()`）。

10. OS environment variables.

    OS环境变量

11. A `RandomValuePropertySource` that has properties only in `random.*`.

    一`RandomValuePropertySource`，只有在拥有性能`random.*`

12. [Profile-specific application properties](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#boot-features-external-config-profile-specific-properties) outside of your packaged jar (`application-{profile}.properties` and YAML variants).

    特定于配置文件的应用程序属性](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#boot-features-external-config-profile-specific-properties)在打包的jar（`application-{profile}.properties`和YAML变体）之外。

13. [Profile-specific application properties](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#boot-features-external-config-profile-specific-properties) packaged inside your jar (`application-{profile}.properties` and YAML variants).

    打包在jar中[的特定于配置文件的应用程序属性](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#boot-features-external-config-profile-specific-properties)（`application-{profile}.properties` 以及YAML变体）。

14. Application properties outside of your packaged jar (`application.properties` and YAML variants).

    应用程序属性在打包的jar之外（`application.properties`和YAML变体）

15. Application properties packaged inside your jar (`application.properties` and YAML variants).

    打包在jar中的应用程序属性（`application.properties`和YAML变体）

16. [`@PropertySource`](https://docs.spring.io/spring/docs/5.1.2.RELEASE/javadoc-api/org/springframework/context/annotation/PropertySource.html) annotations on your `@Configuration` classes.

    [`@PropertySource`](https://docs.spring.io/spring/docs/5.1.2.RELEASE/javadoc-api/org/springframework/context/annotation/PropertySource.html) 你的`@Configuration`课上的注释。

17. Default properties (specified by setting `SpringApplication.setDefaultProperties`).

    默认属性（由设置指定`SpringApplication.setDefaultProperties`）。



#### 控制顺序

java SPI :`java.util.ServiceLoader`

实现`Order`以及标记`@Order`

再spring中，数值越小，优先级越高



### spring cloud配置

更改 Bootstrap属性的位置：`spring.cloud.bootstrap.location` 默认值是bootstrap

覆盖远程属性的值 ：`spring.cloud.config.allowOverride=true` 默认为true 允许覆盖



##### Env端点 :`EnvironmentEndoint`

`Environment`关联多个带名称的`PropertySource`

参考SpringFramework上的例子

`org.springframework.web.context.support.AbstractRefreshableWebApplicationContext`

```java
protected void initPropertySources() {
		ConfigurableEnvironment env = getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, this.servletConfig);
		}
	}
```



Enbironment 有2种实现方式



### 自定义Bootstrap配置

https://cloud.spring.io/spring-cloud-static/Finchley.SR2/single/spring-cloud.html#customizing-bootstrap-properties

1. 编写代码（不要忘记@Order() 顺序）

```java
@Configuration
public class CustomPropertySourceLocator implements PropertySourceLocator {

    @Override
    public PropertySource<?> locate(Environment environment) {
        return new MapPropertySource("customProperty",
                Collections.<String, Object>singletonMap("property.from.sample.custom.source", "worked as intended"));
    }

}
```

1. 创建`META-INF/spring.factories` 
2. 添加

```properties
org.springframework.cloud.bootstrap.BootstrapConfiguration=sample.custom.CustomPropertySourceLocator
```

注意事项：

`Enbironment` 允许出现同名的配置，不过优先级高的胜出

内部的实现：`MutablePropertySources` 关联代码：

```java
private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<PropertySource<?>>();
```

propertySourceList FIFO,他有顺序

可以通过`MutablePropertySources#addFirst()`提高到最高优先级  相当于调用`List#add(0,PropertySource);`





