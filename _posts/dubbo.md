---
title: dubbo SPI扩展点
date: 2019-09-19 12:55:36
tags: dubbo,SPI
---



1. 负载均衡  客户端和服务端 优先级  方法层级> 客户端 >服务端
2. 服务降级  客户端  （mock）
3. 容错  配置在客户端
4. 强制依赖  dubbo.registry.check=false  非强制依赖
5. ip地址绑定   
   1. dubbo.protocol.host=ip  绑定IP地址
   2. 或者配置Host文件 然后配置`dubbo.protocol.host` 为hostName
6. dubbo 的配置中心的优先级 
   1. 配置中心的优先级 最高 
   2. application.poroperties 一定要有个配置 兜底作用
7. 元数据中心  简化 dubbo 的连接配置 
   1. `dubbo.registry.simplified=true`
   2. `dubbo.metadata-report.address=zookeeper://...`





## SPI 扩展点

### 静态扩展点

```java
Protocol dubbo =ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("dubbo");
```



### 默认扩展点

~~~java
Protocol dubbo = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("true");
~~~

1. 以上代码就是获取一个默认扩展点 

默认扩展点就是 被`@SPI(dubbo)` 注解后 值信息      ,所以 上面代码 或得的是一个dubbo 的 扩展点 

~~~java
@SPI("dubbo")
public interface Protocol 
~~~



+ 获取的函数

~~~java
 public T getExtension(String name) {
        if (StringUtils.isEmpty(name)) {  // 判断是否 有值
            throw new IllegalArgumentException("Extension name == null");
        }
        if ("true".equals(name)) { //如果是true的话，
            return getDefaultExtension(); // 调用 默认的扩展点
        }
       ...忽略其他代码
    }
~~~



+ 获取默认的扩展点

~~~java
public T getDefaultExtension() {
    getExtensionClasses();   // 加载所以的扩展点 信息 就是spi的配置信息
    if (StringUtils.isBlank(cachedDefaultName)  
        || "true".equals(cachedDefaultName)) { // 这里的cachedDefaultName 为SPI注解值
        return null;                           // 具体看cacheDefaultExtensionName 函数
    }
    return getExtension(cachedDefaultName);  // 调用getExtension 函数
}
~~~



+ 具体`cachedDefaultName` 从何而来的函数

~~~java
 private void cacheDefaultExtensionName() {
     final SPI defaultAnnotation = type.getAnnotation(SPI.class);
     if (defaultAnnotation != null) {
         String value = defaultAnnotation.value();
         if ((value = value.trim()).length() > 0) {
             String[] names = NAME_SEPARATOR.split(value);
             if (names.length > 1) {
                 throw new IllegalStateException("More than 1 default extension name on extension " + type.getName() + ": " + Arrays.toString(names)); 
             }
             if (names.length == 1) {
                 cachedDefaultName = names[0];   // 这里进行赋值的
             }
         }
     }
 }


~~~

+ 扩展点的创建

~~~java
 public T getExtension(String name) {
       ...忽略其他代码
        Holder<Object> holder = getOrCreateHolder(name);  // 获取一个缓存对象
        Object instance = holder.get();  // 获取实例 
        if (instance == null) {          // 
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name);        // 创建扩展点
                    holder.set(instance);
                }
            }
        }
    }
~~~

+ 具体扩展点的创建

~~~java
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name);   // 从缓存中获取 对应的class 实现类
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);    // 从缓存中获取对应的实例
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance()); // 如果没有就创建一个
            instance = (T) EXTENSION_INSTANCES.get(clazz);  
        }
        injectExtension(instance);   // 这里 是对扩展点的 属性注入
        ... 忽略代码
    }
}
~~~

+ 扩展点 属性注入

~~~java
private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {  // 遍历属性
                    if (isSetter(method)) {
                        if (method.getAnnotation(DisableInject.class) != null) {  
                            continue;
                        }
                        Class<?> pt = method.getParameterTypes()[0];
                        if (ReflectUtils.isPrimitives(pt)) {
                            continue;
                        }
                        try {
                            String property = getSetterProperty(method); // 获取扩展点的name名称
                            Object object = objectFactory.getExtension(pt, property); // 从缓存中获取实例   objectFactory这里是哪里来的？？？
                            if (object != null) {
                                method.invoke(instance, object); //执行赋值
                            }
                        ... 省略代码
        return instance;
    }
~~~

+ `objectFactory` 从哪里来的 

~~~java
// 构造函数传入的
private ExtensionLoader(Class<?> type) {
    this.type = type;
    // 根据
    objectFactory = (type == ExtensionFactory.class ? null :                      ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
~~~

> 这里的ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension() 就是下面要说的自适应扩展点



### 自适应扩展点

~~~java
ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension() 
~~~

+ 如何获取扩展点

~~~java
public T getAdaptiveExtension() {
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            instance = createAdaptiveExtension(); // 创建激活扩展点
                            cachedAdaptiveInstance.set(instance);
                    ...省略代码
        }

        return (T) instance;
    }
~~~

+ 具体`createAdaptiveExtension`

~~~java
private T createAdaptiveExtension() {
        try {
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
~~~

+ `getAdaptiveExtensionClass` 获取自适应扩展点的class 信息

~~~java
private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {  // 类上的spi 信息
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass(); // 创建一个class
    }
~~~

+ 具体如何创建的class

~~~java
private Class<?> createAdaptiveExtensionClass() {
    // 获取扩展点的类字符串	
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader(); // 获取类加载器
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension(); // 先获取到编译的激活扩展点 再进行编译
    return compiler.compile(code, classLoader); //加载到jvm中
}
~~~



## Activate 自动激活扩展点

就是 在 类上进行 使用`@Activate` 进行注解的类 

~~~java
// 通过group 进行 设置 是在客户端 还是服务端生效    如果都有的话就是 都生效
@Activate(group = {CONSUMER, PROVIDER}, value = CACHE_KEY)
public class CacheFilter implements Filter {

~~~









## 服务发布和注册

1. 解析dubbo的配置信息 
2. 创建服务，
3. 缓存到本地，交给spring ioc
4. 进行发布服务
5. 序列化  反序列化
6. netty 发布 



InitializingBean,

​	初始化bean的最后，进行属性设置

DisposableBean,

​	bean销毁的时候 需要做的事情

ApplicationContextAware,

​	注入applicationContext属性

 ApplicationListener<ContextRefreshedEvent>,

​	监听spring的事件，这里是 监听context刷新的事件 

BeanNameAware,

​	设置beanName 

ApplicationEventPublisherAware 

​	spring 上下文的 





以上最终的是监听context刷新完成后的事件 需要做的事情

`cachedWrapperClasses `  缓存 集合

`class org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper`

`class org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper`

`class org.apache.dubbo.qos.protocol.QosProtocolWrapper`





~~~java
registry://172.31.180.242:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-serice&dubbo=2.0.2&pid=18304&qos-enable=false&registry=zookeeper&release=2.7.2&simplified=false&timestamp=1564821299358
~~~

com.wl.dubbo.api.IUserService

<dubbo:protocol name="dubbo" port="21880" valid="true" id="dubbo" prefix="dubbo.protocols." />



~~~java
dubbo://10.0.75.1:21880/com.wl.dubbo.api.IUserService?anyhost=true&application=dubbo-serice&bean.name=ServiceBean:com.wl.dubbo.api.IUserService&bind.ip=10.0.75.1&bind.port=21880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.wl.dubbo.api.IUserService&methods=hello&pid=18304&qos-enable=false&register=true&release=2.7.2&side=provider&timestamp=1564821526705
~~~



createServer 

~~~java
dubbo://10.0.75.1:21880/com.wl.dubbo.api.IUserService?anyhost=true&application=dubbo-serice&bean.name=ServiceBean:com.wl.dubbo.api.IUserService&bind.ip=10.0.75.1&bind.port=21880&channel.readonly.sent=true&codec=dubbo&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&heartbeat=60000&interface=com.wl.dubbo.api.IUserService&methods=hello&pid=18304&qos-enable=false&register=true&release=2.7.2&side=provider&timestamp=1564821526705
~~~



如果让我们 来开发一个 配置中心的话，需要做什么 

1. 服务端 配置更新  





http://thesecretlivesofdata.com/raft/



RaftCode 原理



https://media.pearsoncmg.com/aw/ecs_kurose_compnetwork_7/cw/content/interactiveanimations/selective-repeat-protocol/index.html  滑动窗口