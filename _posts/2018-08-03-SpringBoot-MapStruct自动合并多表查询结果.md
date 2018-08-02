---
layout:     post
title:      SpringBoot-MapStruct自动合并多表查询结果
subtitle:   Mapstruct进行实体与模型之间的自动映射操作
date:       2018-08-03
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - SpringBoot
    - JPA
---

### 背景

在我们开发的规范中，MySQL 数据库不使用多表查询，也就是 `@OneToMany` 、 `@ManyToMany` 这样的注解不会出现在我们的实体中。但在实际的工作中，返回给前端的 Model 数据往往来自多个表的数据拼装。

### 使用 Mapstruct 来进行实体与 Model 之间自动映射

`MapStruct` 是一个代码生成器的工具类，简化了不同的 Java Bean 之间映射的处理，指的就是从一个实体变化成一个实体。在实际项目中，我们经常会将多个表的查询结构封装成页面显示需要的 Model 。在转换时大部分属性都是相同的，只有少部分的不同，这时我们可以通过 `MapStruct` 的一些注解来匹配不同属性，可以让不同实体之间的转换变的简单。 

### 举例

* 添加依赖包

```
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
        <org.mapstruct.version>1.2.0.Final</org.mapstruct.version>
    </properties>
...
    <!--mapStruct依赖-->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-jdk8</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>${org.mapstruct.version}</version>
        <scope>provided</scope>
    </dependency>
    
...
```

* 场景：

> 假如查看订单详情页的时候需要显示订购人的基本信息，包括名字、电话和地址。这些信息分别保存在两个数据表中 `Person` 和 `Address` ，每个实体的字段情况如下：

```
// Person.java

@Data
public class Person implements Serializable {
    private static final long serialVersionUID = 3737812848430885920L;
    private String name;
    private int age;
    private String phone;

}


// Address.java
@Data
public class Address implements Serializable {

    private static final long serialVersionUID = 3119022747763731665L;
    private String address;
    private String postcode;

}

// 接口返回到页面的 Model， PersonModel.java
@Data
public class PersonModel implements Serializable {

    private static final long serialVersionUID = -7833986883909748561L;
    private String personName;
    private String phone;
    private String addr;


}
```


* 创建 Mapper（映射）接口，MapStruct 就会自动实现该接口

```
@Mapper(componentModel = "spring")
public interface PersonMapper {

    @Mappings({
            @Mapping(source = "person.name", target = "personName"),
            @Mapping(source = "address.address", target = "addr")
    })
    PersonModel map2Dto(Person person, Address address);
}
```

* 说明：

> @Mapper注解是用于标注接口、抽象类是被 MapStruct 自动映射的标识，只有存在该注解才会将内部的接口方法自动实现。 
MapStruct 为我们提供了多种的获取 Mapper 的方式，比较常用的两种分别是:

> 默认配置，我们不需要做过多的配置内容，获取 Mapper 的方式就是采用Mappers通过动态工厂内部反射机制完成Mapper实现类的获取。
> 
> ```
> PersonMapper INSTANCE = Mappers.getMapper(PersonMapper.class);
> PersonMapper.INSTANCE.map2Dto(person, address);
> ```

> Spring方式，需要在 `@Mapper` 注解内添加 componentModel 属性值，配置后在外部可以采用 `@Autowired` 方式注入 Mapper 实现类完成映射方法调用。 Spring方式获取Mapper如下所示：
> 
> ```
> @Mapper(componentModel = "spring")
> 
> @Resource
> private PersonMapper personMapper;
> 
> PersonModel personModel = personMapper.map2Dto(person, address);
> 
> ```

* 测试

```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class PersonModelTest {

//    @Autowired
    @Resource
    private PersonMapper personMapper;

    @Test
    public void testMapstruct() {
    
    // 模拟从数据库中取值
        Person person = new Person();
        person.setName("Yezhiwei");
        person.setAge(18);
        person.setPhone("150");

        Address address = new Address();
        address.setAddress("BeiJing");
        address.setPostcode("10086");

    // 将多个表的数据拼装成页面需要的 Model
        PersonModel personModel = personMapper.map2Dto(person, address);
//        PersonModel personModel = PersonMapper.INSTANCE.map2Dto(person, address);

        System.out.println("personModel=>" + personModel);
    }
}
```

* 遇到的错误

> Error:(12, 1) java: Couldn't retrieve @Mapper annotation

由于我的项目中使用了 swagger2 ，在 pom.xml 引入的swagger2配置本身也依赖 mapstruct 包，所以，需要 exclusions，如下配置：

```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.6.1</version>
    <exclusions>
        <exclusion>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

* 运行输出结果

```
...
2018-08-02 22:33:26.743  INFO 49608 --- [           main] com.gemantic.push.core.PersonModelTest   : Started PersonModelTest in 15.303 seconds (JVM running for 18.165)
personModel=>PersonModel(personName=Yezhiwei, phone=150, addr=BeiJing)
...
```

***













