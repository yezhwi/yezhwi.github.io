---
layout:     post
title:      Java的POJO类为什么要实现Serializable接口
subtitle:   
date:       2020-06-24
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
    - 架构
---


### 遇到过的问题

在分布式架构中，项目结构一般会把公用的部分抽取到一个单独的项目中，比如，与数据库映射的类 `User` 放到 `xx-core` 工程中，对 `User` 的操作（CRUD）封装成一个服务，如 `xx-service`，在这个 service 中引入 `xx-core` 依赖，然后对外提供接口服务能力。下面看一下 `User` 类的代码：

```
public class User {

    private String name;
    private Integer age;

    // getter and setter
    // ...

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```

#### 问题一

在 `xx-service` 项目中进行单元测试，没有问题，一旦另一个项目通过网络（RPC或 HTTP）远程调用 `xx-service` 中的接口，就会报序列化异常，通过日志很快能发现 `User` 类没有实现 `Serializable` 接口，修正代码如下：

```
public class User implements Serializable {

    private String name;
    private Integer age;

    // getter and setter
    // ...

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```

再次调用一切正常。

#### 问题二

随着需求的迭代，需要对 `User` 类进行字段扩展以满足新的需求，如对 `User` 增加手机号字段，代码如下：

```
public class User implements Serializable {

    private String name;
    private Integer age;
    // 新增字段
    private String phone;

    // getter and setter
    // ...

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```

新服务 `xx-service` 发布上线后，发布有部分依赖 `xx-core` 工程的报出

```
Exception in thread "main" java.io.InvalidClassException: com.xxx.User; local class incompatible: stream classdesc serialVersionUID = 1035612825366362028, local class serialVersionUID = -1830850955695931978
```

报错原因为序列化与反序列化产生的 `serialVersionUID` 不一致，接下来在上面 `User` 类的基础上显示指定一个 `serialVersionUID` 问题解决，利用开发工具自动生成。如果在使用过程中修改这个 `serialVersionUID` 的值也会报同样的异常，所以**请不要随意修改这个字段**。 修改后的代码如下：

```
public class User implements Serializable {

    // 不用关心具体的数字，也不要随意修改
    private static final long serialVersionUID = 1035612825366362028L;

    private String name;
    private Integer age;
    private String phone;

    // getter and setter
    // ...

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

### 在阿里 Java 开发手册也有这样的强制规范

【强制】序列化类新增属性时，请不要修改 `serialVersionUID` 字段，避免反序列失败; 如果完全不兼容升级，避免反序列化混乱，那么请修改 `serialVersionUID` 值。
说明:注意 `serialVersionUID` 不一致会抛出序列化运行时异常。


### 总结

#### 什么是序列化和反序列化？

序列化：把对象转换为字节序列的过程称为对象的序列化。
反序列化：把字节序列恢复为对象的过程称为对象的反序列化。

#### 什么时候需要用到序列化和反序列化呢?

当我们只在本地 JVM 里运行下 Java 实例，这个时候是不需要序列化和反序列化的，但当我们需要将内存中的对象持久化到磁盘, 数据库中时， 当我们需要与浏览器进行交互时， 当我们需要实现 RPC 时， 这个时候就需要序列化和反序列化了。

总之，只要我们对内存中的对象进行持久化或网络传输，这个时候都需要序列化和反序列化。

#### 为什么还要显示指定 serialVersionUID 的值?

如果不显示指定 serialVersionUID，JVM 在序列化时会根据属性自动生成一个 `serialVersionUID`，然后与属性一起序列化，再进行持久化或网络传输。在反序列化时，JVM 会再根据属性自动生成一个新版`serialVersionUID`， 然后将这个新版 `serialVersionUID` 与序列化时生成的旧版 `serialVersionUID` 进行比较，如果相同则反序列化成功，否则报错。

如果显示指定了 `serialVersionUID`， JVM 在序列化和反序列化时仍然都会生成一个 `serialVersionUID`，但值为我们显示指定的值，这样在反序列化时新旧版本的 `serialVersionUID` 就一致了.

在实际开发中，我们的类会不断迭代，一旦类被修改了，那旧对象反序列化就会报错。所以在实际开发中，我们都会显示指定一个`serialVersionUID`。

#### Java序列化的其他特性

被 `transient`、 `static` 关键字修饰的属性不会被序列化。

### 最佳实践

在定义 POJO 类时都实现 `Serializable` 接口，并且生成 `serialVersionUID` 属性，不管值是什么，都不要进行修改。

> 如果觉得还有帮助的话，你的关注和转发是对我最大的支持，O(∩_∩)O:



