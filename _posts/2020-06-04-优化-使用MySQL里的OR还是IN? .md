---
layout:     post
title:      使用 MySQL 里的 OR 还是 IN? 
subtitle:   
date:       2020-06-04
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
    - MySQL
    - 优化
---

### 异常日志

```
org.springframework.web.util.NestedServletException: Handler dispatch failed; nested exception is java.lang.StackOverflowError
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:978)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:897)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970)
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:861)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:687)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:846)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:790)
	at io.undertow.servlet.handlers.ServletHandler.handleRequest(ServletHandler.java:85)
	at io.undertow.servlet.handlers.FilterHandler$FilterChainImpl.doFilter(FilterHandler.java:129)
	at org.springframework.boot.web.filter.ApplicationContextHeaderFilter.doFilterInternal(ApplicationContextHeaderFilter.java:55)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
	at io.undertow.servlet.core.ManagedFilter.doFilter(ManagedFilter.java:61)
	......
	
Caused by: java.lang.StackOverflowError: null
	at org.hibernate.hql.internal.antlr.HqlSqlBaseWalker.logicalExpr(HqlSqlBaseWalker.java:2042)
	at org.hibernate.hql.internal.antlr.HqlSqlBaseWalker.logicalExpr(HqlSqlBaseWalker.java:2081)
	at org.hibernate.hql.internal.antlr.HqlSqlBaseWalker.logicalExpr(HqlSqlBaseWalker.java:2081)
	at org.hibernate.hql.internal.antlr.HqlSqlBaseWalker.logicalExpr(HqlSqlBaseWalker.java:2081)
	at org.hibernate.hql.internal.antlr.HqlSqlBaseWalker.logicalExpr(HqlSqlBaseWalker.java:2081)
	......
```

### 问题分析

* 接口是 `GET` 请求
* 请求参数有个字段是数组类型，并且没有限制参数个数大小
* 执行查询的 SQL 对这个参数使用 `or` 操作
* 测试
	* 参数传入少量的数据，请求正常
	* 参数传入大量的数据，报上面的异常 


### 定位原因

一定是这个 `or` 操作出现的问题，使用 `in` 进行优化，[参考规范](https://yezhwi.github.io/2018/04/02/MySQL%E4%BD%BF%E7%94%A8%E8%A7%84%E8%8C%83/#sql%E8%A7%84%E8%8C%83)

### 解决方案

* 修改为 `in` 操作
* 进行测试问题解决
	* 接口不报错，能正常返回结果
	* 再次加大此参数的个数，再次出现的新问题 `Nginx 414 Request-URI Too Large` ，原因是 `GET` 请求头过大，可通过修改 `Nginx` 参数解决问题，但是 `Nginx` 生产上都是统一的配置，所以不推荐修改，解决方案为将接口请求方式改为 `POST`

### or 和 in 效率的问题

> 具体测试方法和数据请参考下文
> 
> 原文：http://blog.chinaunix.net/uid-20639775-id-3416737.html
> 
> 结论：
> 
>     1）从测试结果看，如果 in 和 or 所在列有索引或者主键的话，or 和 in 没啥差别，执行计划和执行时间都几乎一样。二者平手
> 
>     2）如果 in 和 or 所在列没有索引的话，性能差别就很大了。in 胜出！
> 
>     3）在没有索引的情况下，随着 in 或者 or 后面的数据量越多，in 的效率不会有太大的下降，但是 or 会随着记录越多的话性能下降非常厉害，从第三中测试情况中可以很明显地看出了，基本上是指数级增长。
> 
>     4）因此在给 in 和 or 的效率下定义的时候，应该再加上一个条件，就是所在的列是否有索引或者是否是主键。如果有索引或者主键性能没啥差别，如果没有索引，性能差别不是一点点！




--

> 如果觉得还有帮助的话，你的关注和转发是对我最大的支持，O(∩_∩)O:



