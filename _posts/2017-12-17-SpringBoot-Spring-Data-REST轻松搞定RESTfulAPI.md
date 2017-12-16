---
layout:     post
title:      SpringBoot-Spring-Data-REST轻松搞定RESTfulAPI
subtitle:   两行代码即可实现实体类的RESTful风格的所有接口
date:       2017-12-17
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - Spring Boot
    - RESTful API
---

### 背景

昨天同事问我有没有研究过`spring-boot-starter-data-rest`，没有～但是看名字就大概知道是做什么的（命名的重要性），因为之前有了解过`spring-boot-starter-data-jpa` ，过一会发过两个截图过来。真的很强大，感觉这个在使用RESTful风格接口协议的微服务时都不用写Controller了。

### 什么是Spring Data REST 

Spring Data REST是基于Spring Data的Repository，把 Repository 自动输出为REST资源，目前支持Spring Data JPA、Spring Data MongoDB、Spring Data Neo4j、Spring Data GemFire、Spring Data Cassandra的 Repository 自动转换成REST服务。注意是***自动***。Spring Data REST把我们需要编写的大量REST模版接口做了自动化实现。

![spring-data-rest](/img/spring-data-rest-0.png)

### 两行代码即可实现

在网上大概了解一下，然后动手做个demo，果然是两行代码即可实现。

* 新建一个Spring Boot项目，添加依赖

```
<parent>
    <artifactId>spring-boot</artifactId>
    <groupId>com.gemantic</groupId>
    <version>1.0-SNAPSHOT</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-rest</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
    
	<!-- 简化代码 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.16.14</version>
    </dependency>
    
	<!-- 类似swagger -->
    <dependency>
        <groupId>org.springframework.data</groupId>
        <artifactId>spring-data-rest-hal-browser</artifactId>
    </dependency>

    <!-- datasource pool-->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
        <version>1.0.14</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

</dependencies>
```

* 表结构

```
CREATE TABLE `user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_name` varchar(100) DEFAULT NULL,
  `password` varchar(100) DEFAULT NULL,
  `phone` varchar(100) DEFAULT NULL,
  `locked` tinyint(1) DEFAULT '0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_user_user_name` (`user_name`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

```

放点数据进去，如下图：

![user](/img/spring-data-rest-1.png)

* 与表对应的实体

```
import lombok.Data;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

/**
 * Created by Yezhiwei on 17/10/31.
 */

@Data
@Entity
public class User {

    @Id
    @GeneratedValue(strategy= GenerationType.AUTO)
    private Long id;

    private String userName;

    private String password;

    private String phone;

    private Integer locked;
}
```

* 创建User表对应的Repository

```
import com.gemantic.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

/**
 * @author Yezhiwei
 * @date 17/12/16
 */
@RepositoryRestResource(path="user")
public interface UserRepository extends JpaRepository<User, Long> {

}
```

自定了一个接口UserRepository 继承了JpaRepository，其中泛型中的User是实体类，Long是主键类型，在类的头部加上了一个 @RepositoryRestResource注解，并添加了一个Path为user。就这样，两行代码即可实现User实体类的RESTFul风格的所有接口。

* 测试，访问 `http://localhost:8080/user`

![user](/img/spring-data-rest-2.png)

*接口中自动附带查询详情的链接*

分页测试，`http://localhost:8080/user?size=2&page=3`

![user](/img/spring-data-rest-3.png)

*接口中同样自动附带分页的链接，分页信息*

***这样更便于解耦前后端，后端如果链接地址变了，前端不用改，直接用Link里面的地址访问***

* 同样，也提供了一个类似swagger的接口测试UI

```
<!-- 类似swagger -->
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-rest-hal-browser</artifactId>
</dependency>
```

![The HAL Browser](/img/spring-data-rest-4.png)

***

问题：

* 现在还不知道自动转换成REST服务有哪些缺点？
* 自动转换成REST服务，是否支持自定义功能？
* 还需要进一步测试与Feign一起使用的情况。
* ......











