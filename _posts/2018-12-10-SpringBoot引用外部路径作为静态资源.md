---
layout:     post
title:      SpringBoot 引用外部路径作为静态资源
subtitle:   
date:       2018-12-10
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - SpringBoot
---


### 静态资源访问

我们在开发 Web 应用的时候，需要引用大量的 js、css、图片等静态资源。

### 默认配置

SpringBoot 默认提供静态资源目录位置需置于 classpath 下（推荐使用默认配置），目录名需符合如下规则：

* classpath:/META-INF/resources
* classpath:/resources
* classpath:/static
* classpath:/public

![](https://tva2.sinaimg.cn/large/006tNbRwly1fy0teik9jaj30ke0cydhg.jpg)

上面几个静态资源映射路径的优先级顺序为：META-INF/resources > resources > static > public

### SpringBoot 涉及到静态资源配置的两个参数说明

* spring.mvc.static-path-pattern

表示我们应该以什么样的路径模式来访问静态资源，即：只有静态资源满足什么样的匹配条件，Spring Boot 才会处理静态资源请求

![](https://tva2.sinaimg.cn/large/006tNbRwly1fy0tqa03p2j31xs0lajtt.jpg)

* spring.resources.static-locations

指定 SpringBoot 应该在哪些路径下查找静态资源文件，这是一个列表性的配置，查找文件时会依赖于配置的先后顺序依次进行，默认的官方配置如下：

```
spring.resources.static-locations=classpath:/META-INF/resources/,classpath:/resources/,classpath:/static/,classpath:/public/
```

### 示例：单机版资源服务器

> 可以借助阿里云 OSS 存储服务 (官方文档 https://help.aliyun.com/product/31815.html?spm=a2c4g.11186623.6.540.50f369cb2kTnAQ)搭建更加强大的分布式资源存储服务器
 
场景：我需要在客户端上传图片或视频到服务器，然后提供 HTTP 的访问方式查看到这些资源。如果上传的图片可以直接保存到 resources 目录下，就可以通过 http://ip:port/xxx.jpg 访问到，但是代码发布到服务器是以 jar 包的方式，所以保存文件的目录只能是一个外部目录，如：/data/resources/ 。通过学完上面的两个参数，可以很快的实现这个功能，如下配置代码：

```
application.yml

spring:
  application:
    name: xxx-xxx-server
  resources:
    static-locations: file:${audio.path}

audio:
  path: /data/resources/audio
```
