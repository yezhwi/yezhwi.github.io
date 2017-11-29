---
layout:     post
title:      SpringBoot访问MongoDB数据库
subtitle:   
date:       2017-11-28
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - SpringBoot
    - MongoDB
---


### MongoDB简介

MongoDB是一个基于分布式文件存储的数据库，它是一个介于关系数据库和非关系数据库之间的产品，其主要目标是在键/值存储方式（提供了高性能和高度伸缩性）和传统的RDBMS系统（具有丰富的功能）之间架起一座桥梁，它集两者的优势于一身。

传统的关系数据库一般由数据库（database）、表（table）、记录（record）三个层次概念组成，MongoDB是由数据库（database）、集合（collection）、文档对象（document）三个层次组成。

MongoDB中的一条记录就是一个文档，是一个数据结构，由字段和值对组成。MongoDB文档与JSON对象类似。字段的值有可能包括其它文档、数组以及文档数组。

MongoDB的适合对大量或者无固定格式的数据进行存储，比如：日志、缓存等。对事物支持较弱，不适用复杂的多文档（多表）的级联查询。

### MongoDB的CRUD

* 添加pom.xml依赖

Spring Boot中可以通过在pom.xml中加入spring-boot-starter-data-mongodb引入对mongodb的访问支持依赖。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

* 创建Entity对象

```
package com.gemantic.entity;

import java.io.Serializable;


public class UserEntity implements Serializable {

    private static final long serialVersionUID = -3258839839160856613L;
    private Long id;
    private String userName;
    private String passWord;

    
	// 省略getter和setter
}
```

* CRUD

```
package com.gemantic.dao.impl;

import com.mongodb.WriteResult;
import com.gemantic.entity.UserEntity;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.security.SecurityProperties;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.stereotype.Service;


@Service
public class UserDaoImpl implements UserDao {

    @Autowired
    private MongoTemplate mongoTemplate;

    /**
     * 创建对象
     * @param user
     */
    @Override
    public void saveUser(UserEntity user) {
        mongoTemplate.save(user);
    }

    /**
     * 根据用户名查询对象
     * @param userName
     * @return
     */
    @Override
    public UserEntity findUserByUserName(String userName) {
        Query query = new Query(Criteria.where("userName").is(userName));
        UserEntity user = mongoTemplate.findOne(query , UserEntity.class);
        return user;
    }

    /**
     * 更新对象
     * @param user
     */
    @Override
    public int updateUser(UserEntity user) {
        Query query = new Query(Criteria.where("id").is(user.getId()));
        Update update = new Update().set("userName", user.getUserName()).set("passWord", user.getPassWord());
        //更新查询返回结果集的第一条
        WriteResult result = mongoTemplate.updateFirst(query, update, UserEntity.class);
        //更新查询返回结果集的所有
        // mongoTemplate.updateMulti(query, update, UserEntity.class);
        if(result != null) {
            return result.getN();
        } else {
            return 0;
        }
    }

    /**
     * 删除对象
     * @param id
     */
    @Override
    public void deleteUserById(Long id) {
        Query query = new Query(Criteria.where("id").is(id));
        mongoTemplate.remove(query, UserEntity.class);
    }
}
```


### 多数据源MongoDB的使用

* 配置多数据源，这里配置两个数据源

application.yml 

```
mongodb:
  primary:
    host: 192.168.1.50
    port: 27017
    database: test
  secondary:
    host: 192.168.1.60
    port: 27017
    database: test1
```

* 封装读取以mongodb开头的两个配置文件

```
package com.gemantic.config;

import lombok.Data;
import org.springframework.boot.autoconfigure.mongo.MongoProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;


@Data
@ConfigurationProperties(prefix = "mongodb")
public class MultipleMongoProperties {

	private MongoProperties primary = new MongoProperties();
	private MongoProperties secondary = new MongoProperties();
}
```

* 配置不同包路径下使用不同的数据源

PrimaryMongoConfig

```
package com.gemantic.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.repository.config.EnableMongoRepositories;


@Configuration
@EnableMongoRepositories(basePackages = "com.gemantic.model.primary",
		mongoTemplateRef = PrimaryMongoConfig.MONGO_TEMPLATE)
public class PrimaryMongoConfig {

	protected static final String MONGO_TEMPLATE = "primaryMongoTemplate";
}
```

SecondaryMongoConfig

```
package com.gemantic.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.repository.config.EnableMongoRepositories;


@Configuration
@EnableMongoRepositories(basePackages = "com.gemantic.model.secondary",
		mongoTemplateRef = SecondaryMongoConfig.MONGO_TEMPLATE)
public class SecondaryMongoConfig {

	protected static final String MONGO_TEMPLATE = "secondaryMongoTemplate";
}
```

* 读取对应的配置信息并且构造对应的MongoTemplate，以下代码对两个库的配置信息已经完成。

```
package com.gemantic.config;

import com.mongodb.MongoClient;
import com.gemantic.config.MultipleMongoProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.mongo.MongoProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.mongodb.MongoDbFactory;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.SimpleMongoDbFactory;


@Configuration
public class MultipleMongoConfig {

	@Autowired
	private MultipleMongoProperties mongoProperties;

	@Primary
	@Bean(name = PrimaryMongoConfig.MONGO_TEMPLATE)
	public MongoTemplate primaryMongoTemplate() throws Exception {
		return new MongoTemplate(primaryFactory(this.mongoProperties.getPrimary()));
	}

	@Bean
	@Qualifier(SecondaryMongoConfig.MONGO_TEMPLATE)
	public MongoTemplate secondaryMongoTemplate() throws Exception {
        return new MongoTemplate(secondaryFactory(this.mongoProperties.getSecondary()));
	}

	@Bean
    @Primary
	public MongoDbFactory primaryFactory(MongoProperties mongo) throws Exception {
		return new SimpleMongoDbFactory(new MongoClient(mongo.getHost(), mongo.getPort()),
				mongo.getDatabase());
	}

	@Bean
	public MongoDbFactory secondaryFactory(MongoProperties mongo) throws Exception {
		return new SimpleMongoDbFactory(new MongoClient(mongo.getHost(), mongo.getPort()),
				mongo.getDatabase());
	}
}
```

* 创建两个库分别对应的对象和Repository

对应的Mongo的对象

```
package com.gemantic.model.primary;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Data
@AllArgsConstructor
@NoArgsConstructor
@Document(collection = "first_mongo")
public class PrimaryMongoObject {

	@Id
	private String id;

	private String value;

}
```

对应的Repository，继承了 MongoRepository 会默认实现很多基本的增删改查，省了很多自己写dao层的代码

```
public interface PrimaryRepository extends MongoRepository<PrimaryMongoObject, String> {
}
```

* 主类，注意上面的注解

```
package com.gemantic;

import com.gemantic.config.props.MultipleMongoProperties;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.scheduling.annotation.EnableScheduling;

@EnableConfigurationProperties(MultipleMongoProperties.class)
@SpringBootApplication(exclude = MongoAutoConfiguration.class)
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

* 测试

```
@RunWith(SpringRunner.class)
@SpringBootTest
public class MuliDatabaseTest {

    @Autowired
    private PrimaryRepository primaryRepository;

    @Autowired
    private SecondaryRepository secondaryRepository;

    @Test
    public void TestSave() {

        this.primaryRepository.save(new PrimaryMongoObject(null, "第一个库的对象"));

        this.secondaryRepository.save(new SecondaryMongoObject(null, "第二个库的对象"));

    }
    
    @Test
    public void TestFindAll() {

        List<PrimaryMongoObject> primaries = this.primaryRepository.findAll();
        for (PrimaryMongoObject primary : primaries) {
            System.out.println(primary.toString());
        }

        List<SecondaryMongoObject> secondaries = this.secondaryRepository.findAll();

        for (SecondaryMongoObject secondary : secondaries) {
            System.out.println(secondary.toString());
        }
    }

}
```

---

注意事项

* springboot 依赖版本，1.5.3.RELEASE及以上

```
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>1.5.3.RELEASE</version>
	<relativePath/> <!-- lookup parent from repository -->
</parent>
```

* 启动类上的注解

```
@EnableConfigurationProperties(MultipleMongoProperties.class)
@SpringBootApplication(exclude = MongoAutoConfiguration.class)
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

* 如果服务端启动时开启认证，需要修改`MongoDbFactory `代码

```
package com.gemantic.config;

import com.mongodb.MongoCredential;
import com.mongodb.ServerAddress;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.mongo.MongoProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.mongodb.MongoDbFactory;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.SimpleMongoDbFactory;

import com.mongodb.MongoClient;

import java.util.ArrayList;

@Configuration
public class MultipleMongoConfig {

    @Autowired
    private MultipleMongoProperties mongoProperties;

    @Primary
    @Bean(name = PrimaryMongoConfig.MONGO_TEMPLATE)
    public MongoTemplate primaryMongoTemplate() throws Exception {
        return new MongoTemplate(primaryFactory(this.mongoProperties.getPrimary()));
    }

    @Bean
    @Qualifier(SecondaryMongoConfig.MONGO_TEMPLATE)
    public MongoTemplate secondaryMongoTemplate() throws Exception {
        return new MongoTemplate(secondaryFactory(this.mongoProperties.getSecondary()));
    }

    @Bean
    @Primary
    public MongoDbFactory primaryFactory(MongoProperties mongo) throws Exception {
        ServerAddress addr = new ServerAddress(mongo.getHost(), mongo.getPort());
        ArrayList credentials = new ArrayList();
        credentials.add(MongoCredential.createScramSha1Credential(mongo.getUsername(), mongo.getDatabase(), mongo.getPassword()));

        return new SimpleMongoDbFactory(new MongoClient(addr, credentials), mongo.getDatabase());
    }

    @Bean
    public MongoDbFactory secondaryFactory(MongoProperties mongo) throws Exception {
        ServerAddress addr = new ServerAddress(mongo.getHost(), mongo.getPort());
        ArrayList credentials = new ArrayList();
        credentials.add(MongoCredential.createScramSha1Credential(mongo.getUsername(), mongo.getDatabase(), mongo.getPassword()));

        return new SimpleMongoDbFactory(new MongoClient(addr, credentials), mongo.getDatabase());
    }
}
```









