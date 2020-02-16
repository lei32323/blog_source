---
title: Spring boot +spring cloud
date: 2018-12-16 12:53:36
tags: Spring boot,spring cloud
---



# Spring boot +spring cloud 整理回顾

2个属性相同的不同路径的类需要转换的方法

~~~java
BeanUtils.copyProperties(输入对象,输出对象);
~~~



## JAP

Java持久化的标准

>Hibernate Session#save

entity包：类中有id

domain包：类中没有Id

### Spring Data JPA

2. server.class

~~~java
@PersistenceContext
private EntityManager entityManager;

//存储数据
public void save(Person person){
    entityManager.persist(person);
}
~~~

3. application启动类添加注解

   ~~~java
   @SpringBootApplication
   @EnableTransactionManagement(proxyTargetClass = true) //使用类作为代理对象
   public class JpaDemoApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(JpaDemoApplication.class, args);
   	}
   
   ~~~

4. 配置数据源

   ~~~properties
   spring.application.name=jpa-demo
   server.port=8080
   spring.datasource.url=jdbc:mysql://localhost:3306/test?characterEncoding=utf-8
   spring.datasource.driverClassName=com.mysql.jdbc.Driver
   spring.datasource.username=root
   spring.datasource.password=root
   ~~~

5. 添加驱动依赖

   ~~~xml
   <dependency>
       <groupId>mysql</groupId>
       <artifactId>mysql-connector-java</artifactId>
   </dependency>
   ~~~

6. 分页

   接口`PresonRepository` 继承	`PagingAndSortingRepository<Person,Long>` 注解使用@Repository

   使用`Pageable` 进行分页   接收用`Page<Preson>` 接收 是spring包下的

   ~~~java
   @Repository
   public interface PersonRepositoey extends PagingAndSortingRepository<Person,Long> {
   }
   ~~~


7. 为什么分页参数能识别？

   `PageableArgumentResolver.java`

### Spring Boot JPA