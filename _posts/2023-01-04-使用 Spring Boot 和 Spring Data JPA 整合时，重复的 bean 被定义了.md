---
layout:     post
title:      使用 Spring Boot 和 Spring Data JPA 整合时，重复的 bean 被定义了
subtitle:   规范
date:       2023-01-04
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
---

真正解决方案: A bean with that name has already been defined in null and overriding is disabled.

### 问题描述

> 在使用 Spring Boot 和Spring Data JPA 整合的时候，新手很可能会看到这样的错误

```
***************************
APPLICATION FAILED TO START
***************************

Description:

The bean 'userInfoRepository', defined in null, could not be registered. A bean with that name has already been defined in null and overriding is disabled.

Action:

Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true

Process finished with exit code 1
```

### 故障分析

> Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true

> 根据错误提示，很明显，意思是重复的bean被定义了，然后给了一个提示，建议打开一个开关，bean 重写覆盖功能。

### 解决方案

> 于是乎网上大多数的解决方案都是在 application-dev.properties 里面添加这么一个配置

```
spring.main.allow-bean-definition-overriding=true
```

> 这样能解决问题么？答案是能。
> 
> 但是这种方式并不是最正确的解决方案，因为两个地方同时定义了一个 bean，使用 bean 的覆盖重写其实在某种情况下是很可能出事的。
> 
> 接下来我模拟了下出现这种情况的原因。

### 故障重现

> 演示下错误做法如下：

```
@Repository
public interface UserInfoRepository extends JpaRepository<UserInfo,Long>{
}
```

> 然后配置类这样做的配置

```
@EnableJpaRepositories(basePackages = "com.xingyun.dao.repository")
@Configuration
public class JpaConfig {
}
```

> 启动类中也配置如下

```
@EnableJpaRepositories
@SpringBootApplication
public class SampleApplication {
public static void main(String[] args) {
        //SpringApplication.run方法会构建一个Spring容器,并且返回一个ApplicationContext对象,也就是项目的上下文
        SpringApplication.run(SampleApplication.class, args);
    }
}
```

> 根据原因原来是启动类中已经配置了

```
@EnableJpaRepositories
```

，然后我们又创建了一个`Java Config` 类中再次配置了一次，于是乎就出现了重复bean。

### 第一种正确写法

> 接口类中这样定义

```
@Repository
public interface UserInfoRepository extends JpaRepository<UserInfo,Long>{
}
```

> 然后在 Java Config 类中这样配置

```
@EnableJpaRepositories
@Configuration
public class JpaConfig {
}
```

> 这种方式 Spring Data JPA 会自动扫描包含了`@Repository`注解的接口，并为其运行时生成接口实现类。

### 第二种最佳写法

> 接口类中这样定义

```
public interface UserInfoRepository extends JpaRepository<UserInfo,Long>{
}
```

> 配置类中这样定义

```
@EnableJpaRepositories(basePackages = "com.xingyun.dao.repository")
@Configuration
public class JpaConfig {
}
```

> 这种方式比第二种更好，如果只是这样说你可能不会信我，也不会明白为什么。
> 
> 其实是因为使用这种方式可以避免一种错误，比如你的类中有个继承了`JpaRepository` 有的继承`MongoDBRepository` ,那么 Spring DataJPA 在找到`@Repository`注解后为其在运行时生成接口实现类的时候，就可能会出错。  
> 而我们这种写法可以指定包名去找就可以分割开来。

```
@EnableJpaRepositories(basePackages = "com.acme.repositories.jpa")
@EnableMongoRepositories(basePackages = "com.acme.repositories.mongo")
@Configuration
public class JpaConfig {
}
```

*原文地址：https://blog.csdn.net/hadues/article/details/101840045*


