---
title: spring boot 的自动装配
date: 2020-02-15 12:55:36
tags: spring boot
---



# spring boot 的自动装配

## 注解使用

### @SpringBootApplication

 ### @AutoConfigurationImportSelector 

​	   方法 `selectImports` 

  		1. 加载元数据 `META-INF/spring-autoconfigure-metadata.properties`（一些判断条件)
         
         **配置文件**
         
         `ConditionalOnClass` 当类存在的时候
         

  `ConditionalOnBean` 当bean必须存在时候
         
         `AutoConfigureBefore`加载类之前做的时候
         
    `AutoConfigureAfter` 加载类之后做的时候
    
    `AutoConfigureOrder` 加载的顺序
    
    **注解**
    
    | Conditions                   | 描述                                  |      |
    | ---------------------------- | ------------------------------------- | ---- |
    | @ConditionalOnBean           | 在存在某个bean的时候                  |      |
    | @ConditionalOnMissingBean    | 不存在某个bean的时候                  |      |
    | @ConditionalOnClass          | 当前classpath可以找到某个类型的类时候 |      |
    | @ConditionalOnMissingClass   | 当前classpath不可以找到某个类型的类时 |      |
    | @ConditionalOnResource       | 当前classpath是否存在某个资源文件     |      |
    | @ConditionalOnProperty       | 当前jvm是否包含某个系统属性为某个值   |      |
    | @ConditionalOnWebApplication | 当前spring context是否是web应用程序   |      |


​	
​	
​	
​	
   2. 对一些需要加载的类做处理 
      	1. 从`META-INF/spring.factories `加载配置的类
            	2. 去掉重复的
               	3. 过滤掉配置过exclustion的类

      3. 加载合适的配置类

 

### 自定义实现自动装配

方式1 :

1. 编写配置注解

   ~~~java
   @Target({ElementType.TYPE})
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Inherited
   @AutoConfigurationPackage
   @Import({AutoImportSelector.class}) // 自定义加载类
   public @interface DefaultAutoConfig {
   ~~~

2. 实现`DeferredImportSelector` 接口并且创建 类`AutoImportSelector`

   ~~~java
   import org.springframework.context.annotation.DeferredImportSelector;
   import org.springframework.core.type.AnnotationMetadata;
   
   public class AutoImportSelector implements DeferredImportSelector {
   ~~~

3. ·`AutoImportSelector` 类中编写方法

   ~~~java
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        	// ...对注解的元信息 进行处理
        	// 过滤需要移除的类
        	// 对需要加载的类进行个性化操作
           return new String[]{UserService.class.getName()};
       }
   ~~~

方式2：通过配置文件

1. 创建`META-INF/spring.factories` ,加上自定义的配置类

   ~~~properties
   org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.wl.core.CoreConfig
   ~~~

2. 在`CoreConfig` 配置需要的配置信息

   ~~~java
   @Configurable
   public class CoreConfig {
   
       @Bean
       public Core core(){
           return new Core();
       }
   }
   ~~~

方式3 ：通过`ImportBeanDefinitionRegistrar` 接口

 1. 只要把`方式1` 的import`AutoImportSelector` 换成 `ImportBeanDefinitionRegistrar` 实现类 就可以了 

    ~~~java
    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @AutoConfigurationPackage
    //@Import({AutoImportSelector.class})
    @Import({ImportConfig.class})
    public @interface DefaultAutoConfig {
    ~~~

	2. `ImportBeanDefinitionRegistrar`  处理

    ~~~java
    public class ImportConfig implements ImportBeanDefinitionRegistrar {
        @Override
        public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
            UserService userService = new UserService();
    
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(UserService.class);
    
            String className = StringUtils.uncapitalize(UserService.class.getSimpleName());
            beanDefinitionRegistry.registerBeanDefinition(className, rootBeanDefinition);
        }
    }
    ~~~

    

​		







# spring boot 

2.X 开启监控

~~~xml
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
~~~



`management.endpoints.web.exposure.include=*`









































