## Spring MVC

### Thymeleaf

#### 渲染上下文（模型) Model

- Spring Web MVC
  - 接口模式
    - ModelMap
    - ModelAndView
      - Model -> ModelMap
      - View
  - 注解模式
    - `@ModelAttrtibute`

#### EL表达式

- 字符值
  - boolean
  - int
- 逻辑表达式
  - if else
- 迭代表达式
  - for each /while
- 反射
  - java Reflection
  - CGLib



### 模板寻址

prefix + view-name +suffix



模板缓存

默认cache = true 

cache = false ->设置缓存时间



### Spring MVC 模板渲染逻辑

Spring mvc 核心控制器 `DispatcherServlet`

- C : `DispatcherServlet`
- M :
  - Spring mvc（部分）
    - Model/ModelMap/ModelAndView(Model部分)
    - `@ModelAttribute`
  - 模板引擎（通用）
    - 内置JSP九大变量
      - Scope 上下文
        - PageContext （page变量）
        - ServletRequest(请求上下文)
        - ServletSession(会话上下文)
        - ApplicationContext(应用上下文)
      - JSP内置变量(JSP-Servlet)
        - out(Writer=ServletResponse#getWriter())
        - error(Throwable)
        - Config(ServletConfig)
        - Page(JSP所对应的Servet对象)
        - response(ServletResponse)
    - Thymeleaf内置变量
      - 上下文（模型）
- V : 



### 国际化

#### spring boot

- `MessageSource`
  - `MessageSourceAutoConfiguration`
    - `messages.properties`
    - `messages_en.properties`
    - `messages_zh.properties`
    - `messages_zh_CN.properties`
    - `messages_zh_TW.properties`
- thymeleaf 
  - #key => messagesSource.get



### 学习技巧

假设需要了解的技术是thymeleaf ->thymeleaf Properties->`ThymeleafProperties`

第一步：找到`@ConfiguartionProperties`,确认前缀

```java
@ConfigurationProperties(prefix = "spring.thymeleaf")
public class ThymeleafProperties {
    
}
```

比如:"spring.thymeleaf"



第二步：进一步确认，是否字段和属性名一一对应

```properties
spring.thymeleaf.mode = HTML
spring.thymeleaf.cache = true
```

->

```java
@ConfigurationProperties(prefix = "spring.thymeleaf")
public class ThymeleafProperties {
    ...
    private String mode = "HTML";
    ...
    private boolean cache = true;
    ...
}
```

## Rest

https://en.wikipedia.org/wiki/Representational_state_transfer

RPC

- 语言相关的
  - Java-RMI
  - .NET-COM+
- 语言无关的
  - SAO
    - Web Services
      - SOAP(传输介质协议)
      - HTTP、SMTP（通讯协议）
  - 微服务(MSA)
    - REST
      - HTML、JSON、XML等等
      - HTTP(通讯协议)
        - HTTP1.1
          - 短连接
          - Keep-Alive
          - 连接池
          - Long Polling (长时间轮询)
        - HTTP/2
          - 长连接
      - 技术
        - Spring 客户端：RestTemplate
        - SpringWebMVC:@RestController=`@Controller`+`@ResponseBody`+`@RequestBody`
        - Spring Cloud：`RestTemplate`扩展+`@LoadBalanced`



### Cacheability(可缓存性)

`@ResponseBody` ->响应体(Response Body)

- 响应
  - 响应头（Headers)
    - 元信息（Meta-Data)
      - Accept-Language ->`Locale`
      - Connection ->Keep-Alive
  - 响应体
    - 业务信息(Business Data)
    - Body:HTTP实体、REST
      - `@ResponseBody`
    - Payload:消息JMS、事件、SOAP



### HTTP状态码(`org.springframework.http.HttpStatus`)

- 200
  - `org.springframework.http.HttpStatus#OK`
- 304
  - `org.springframework.http.HttpStatus#NOT_MODIFIED`
  - 第一次完整请求，获取响应头（200），直接获取
  - 第二次请求，只读取请求头，响应头（200），客户端（流量器）取上次Body的结果
- 400
  - `org.springframework.http.HttpStatus#BAD_REQUEST`
- 404
  - `org.springframework.http.HttpStatus#NOT_FOUND`
- 500
  - `org.springframework.http.HttpStatus#INTERNAL_SERVER_ERROR`

### Uniform interface(统一接口)



#### 资源定位

​	

#### 资源操作-HTTP动词

GET

- `@getMapping`

  - 注解属性别名和覆盖

    - spring framework 4.2才引用

      - Spring Framework 1.3才可以使用

    - Spring boot加以扩展

      ```java
      @RequestMapping(method = RequestMethod.POST)
      public @interface PostMapping {
          @AliasFor(annotation = RequestMapping.class)
      	String name() default "";
      }
      ```

      - `@PostMapping`是注解，`@RequestMapping`是`@PostMapping`的注解

        - `@RequestMapping`是`@PostMapping`的元注解
        - `@RequestMapping`元标注了`@PostMapping`

        `@AliasFor`只能针对注解属性，所指向的必须为元注解

POST

- `@PostMapping`

PUT

- `@PutMapping`

PATCH

- `@PatchMapping`

DELETE

- `@DeleteMapping`



### 自描述消息

#### 注解驱动

- `@RequestBody`
  - JSON -> `MappingJackson2HttpMessageConverter`
  - TEXT -> `StringHttpMessageConverter`
- `@ResponseBody`
  - JSON -> `MappingJackson2HttpMessageConverter`
  - TEXT -> `StringHttpMessageConverter`
- 返回值处理类 -> RequestResponseBodyMethodProcessor

### 接口编程

`ResponseEntity ` extends `HttpEntity`

`RequestEntity ` extends `HttpEntity`

返回值处理类 -> `HttpEntityMethodProcessor`



### 媒体类型（MediaType）

- `org.springframework.http.MediaType#APPLICATION_JSON_UTF8_VALUE`
  - "application/json;charset=UTF-8"

消息转换器类（`HttpMessageConverter`）

- application/json
  - `MappingJackson2HttpMessageConverter`
- text/html
  - `StringHttpMessageConverter`

### 代码导读

`@EnableWebMvc`

- 导入`DelegatingWebMvcConfiguration` (配置class)
  - 注册`WebMvcConfigurer` 
  - 装配各种Spring MVC需要的bean
  - 注解驱动扩展点
    - `HandlerMethodArgumentResolver`
    - `HandlerMethodReturnValueHandler`
    - `@RequestBody`和`@ResponseBody`实现类
    - `RequestResponseBodyMethodProcessor`
    - `HttpEntityMethodProcessor`





URL和URI

U：Uniform

R：Resource

I：鉴别

L：定位



URI

```:central_african_republic:
URI = scheme:[//authority]path[?query][#fragment]
```

scheme：HTTP、wechat

URL

protocol协议：





### 注解别名，委派属性

```java
@AliasFor(annotation = EnableAutoConfiguration.class)
String[] excludeName() default {};
```

![1541578008778](C:\Users\wanglei\AppData\Roaming\Typora\typora-user-images\1541578008778.png)











## 类说明

| 类                                    | 说明                                                         |
| :------------------------------------ | ------------------------------------------------------------ |
| HandlerMapping                        | 将请求映射到处理程序以及用于预处理和后处理的[拦截器](https://docs.spring.io/spring/docs/5.1.2.RELEASE/spring-framework-reference/web.html#mvc-handlermapping-interceptor)列表 。映射基于某些标准，其细节因`HandlerMapping` 实施而异。 |
| HandlerAdapter                        | `DispatcherServlet`无论实际调用处理程序如何，都可以帮助调用映射到请求的处理程序。例如，调用带注释的控制器需要解析注释。一个主要目的`HandlerAdapter`是保护`DispatcherServlet`这些细节。 |
| HandlerExceptionResolver              | 解决异常的策略，可能将它们映射到处理程序，HTML错误视图或其他目标。请参阅[例外](https://docs.spring.io/spring/docs/5.1.2.RELEASE/spring-framework-reference/web.html#mvc-exceptionhandlers)。 |
| ViewResolver                          | `String`将从处理程序返回的基于逻辑的视图名称解析为`View` 用于呈现给响应的实际视图。请参阅[查看分辨率](https://docs.spring.io/spring/docs/5.1.2.RELEASE/spring-framework-reference/web.html#mvc-viewresolver)和[查看技术](https://docs.spring.io/spring/docs/5.1.2.RELEASE/spring-framework-reference/web.html#mvc-view)。 |
| LocaleResolver，LocaleContextResolver | 解决`Locale`客户正在使用的问题以及可能的时区问题，以便能够提供国际化的观点。请参阅[区域设置](https://docs.spring.io/spring/docs/5.1.2.RELEASE/spring-framework-reference/web.html#mvc-localeresolver)。 |
| ThemeResolver                         | 解决Web应用程序可以使用的主题 - 例如，提供个性化布局。见[主题](https://docs.spring.io/spring/docs/5.1.2.RELEASE/spring-framework-reference/web.html#mvc-themeresolver)。 |
| MultipartResolver                     | 在一些多部分解析库的帮助下，解析多部分请求（例如，浏览器表单文件上传）的抽象。请参阅[Multipart Resolver](https://docs.spring.io/spring/docs/5.1.2.RELEASE/spring-framework-reference/web.html#mvc-multipart)。 |
| FlashMapManager                       | 存储和检索“输入”和“输出” `FlashMap`，可用于将属性从一个请求传递到另一个请求，通常是通过重定向。请参阅[Flash属性](https://docs.spring.io/spring/docs/5.1.2.RELEASE/spring-framework-reference/web.html#mvc-flash-attributes)。 |

## Spring boot MVC

### Spring mvc 自定义配置

**接口 ：** `WebMvcConfigurer` 

**操作：** 

 	1. 实现 `WebMvcConfigurer`  ,
 	2. 类上添加`@Configuration`,
 	3. 重写方法 ，
 	4. 注释掉application启动类的`@EnableWebMvc` 注解



### HttpMessageConverters 转换HTTP请求和响应

**接口：**`HttpMessageConverters`

**实现类：**  

  +  `AbstractHttpMessageConverter`    
  +  `AbstractJackson2HttpMessageConverter`  处理Json的信息
  +  `BufferedImageHttpMessageConverter` 流的信息
  +  `FormHttpMessageConverter` 从请求和响应读取/编写表单数据。默认情况下，它读取媒体类型 application/x-www-form-urlencoded 并将数据写入 MultiValueMap<String,String>。
  +  `Jaxb2CollectionHttpMessageConverter`
  +  `MarshallingHttpMessageConverter`
  +  `ResourceRegionHttpMessageConverter`
  +  `ObjectToStringHttpMessageConverter`

**操作：**

 1. 添加方法,设置注解

    ~~~java
    @Bean
    public HttpMessageConverters customConverters() {
    		
    }
    ~~~



###  JSON序列化程序和反序列化程序

当`@ResponseBody` 注解返回json数据的时候，返回的是一个Object,spring 会自动将Object转换为json字符串。但是在某些时候，我们希望可以对Json做进一步的处理，比如再次封装下，那么`@JsonComponent`就可以使用了。

~~~java
@JsonComponent
public class Example {

	public static class Serializer extends JsonSerializer<SomeObject> {
		// ...
	}

	public static class Deserializer extends JsonDeserializer<SomeObject> {
		// ...
	}

}
~~~

**说明：** 设置`@JsonComponent` 标签后，会自动配扫描到，因为`@JsonComponent` 的元注解为`@Conponent`

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface JsonComponent {
~~~

**案例：**

~~~java
@JsonComponent
public class Example {
    public static class Serializer extends JsonSerializer<UserEx> {

        @Override
        //返回信息知错处理
        public void serialize(UserEx user, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
            Map<String,Object> map = new HashMap<String,Object>();
            //添加自定义的值
            map.put("a","处理一个参数");
            //原数据返回
            map.put("base",user);

            //输出
            jsonGenerator.writeString(new ObjectMapper().writeValueAsString(map));

            // 使用以上方法默认会转义引号等符号，可以使用 writeRaw 方法进行输出
            // raw 即未加工的 不会改变格式
            // 1.使用自带解析器
            // writeRaw(new ObjectMapper().writeValueAsString(map));
            // 2.使用FastJson
            // arg1.writeRaw (JSON.toJSONString(map));

        }
    }

    public static class Deserializer extends JsonDeserializer<UserEx> {
        @Override
        public UserEx deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
            TreeNode treeNode = jsonParser.getCodec().readTree(jsonParser);
            TextNode base = (TextNode) treeNode.get("base");//转换错误。需要详细

            System.out.println("======================"+base.asText());
            return new UserEx();
        }
    }
}
~~~

[继续](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#boot-features-spring-mvc)





​		