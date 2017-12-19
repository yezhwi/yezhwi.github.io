---
layout:     post
title:      Spring Boot Swagger集成
subtitle:   Swagger用于定义API文档
date:       2017-11-01
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - SpringBoot 
    - Swagger
---


## API文档产出方式

* 传统方式一般使用wiki或者文档，但是每次修改时操作很繁琐，同时调用方也不一定及时了解接口变化情况。
* 在互联网公司, 前后端分离开发, 后端对外提供接口文档，让别人理解接口是必不可少的。 效率是一方面，能及时的反馈给调用方文档；文档的准确性也是一方面，使用Swagger可以在部署的时候生成在线文档，同时UI也特别漂亮清晰，Swagger让维护接口文档、部署管理和使用功能强大的API如此简单。

## Swagger优势
* 前后端分离开发
* API文档非常明确，非常漂亮的UI
* 测试的时候不需要再使用URL输入浏览器的方式来访问Controller
* 传统的输入URL的测试方式对于post请求的传参比较麻烦（当然，可以使用postman这样的浏览器插件）
* Swagger的集成非常简单

## 添加依赖

```
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <springfox.version>2.2.2</springfox.version>
</properties>

<!-- swagger -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>${springfox.version}</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>${springfox.version}</version>
</dependency>
```

## 开启Swagger功能

仅需要在启动类上增加注解 `@EnableSwagger2 `，启动该注解使得用在Controller中的Swagger注解生效

```
@SpringBootApplication
@EnableSwagger2
public class Server {

    public static void main(String[] args) {
        SpringApplication.run(Server.class, args);
    }
}
```










