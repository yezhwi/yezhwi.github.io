---
layout:     post
title:      Spring Boot Web应用
subtitle:   Thymeleaf介绍及使用
date:       2017-10-23
author:     Yezhiwei
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring Boot
---


## Spring Boot开发Web应用

> 上一篇进行了一个Spring Boot快速入门，完成了一个简单的RESTful服务，本篇介绍Web应用的开发。

### 静态资源默认配置

Spring Boot默认提供静态资源目录位置需置于classpath下，目录名需符合如下规则：

* /static
* /public
* /resources
* /META-INF/resources

可以在`src/main/resources/`目录下创建`static`，在该位置放置一个图片文件。启动程序后，尝试访问`http://localhost:8080/avatar.jpg`。如能显示图片，配置成功。

### 模板引擎-Thymeleaf

Spring Boot提供了默认配置的模板引擎主要有以下几种：

* Thymeleaf
* FreeMarker
* Velocity
* Groovy
* Mustache

Spring Boot建议使用这些模板引擎，避免使用JSP。

当你使用上述模板引擎中的任何一个，它们默认的模板配置路径为：`src/main/resources/templates`。

本篇主要介绍Thymeleaf，它提供了一个用于整合Spring MVC的可选模块，在应用开发中，可以使用Thymeleaf来完全代替JSP或其他模板引擎，如Velocity、FreeMarker等。Thymeleaf的主要目标在于提供一种可被浏览器正确显示的、格式良好的模板。可以使用它创建经过验证的XML与HTML模板。相对于编写逻辑或代码的开发者只需将标签属性添加到模板中即可，然后这些标签属性就会在DOM上执行预先制定好的逻辑。

示例模板：

```
<table>
  <thead>
    <tr>
      <th th:text="#{msgs.headers.name}">Name</td>
      <th th:text="#{msgs.headers.price}">Price</td>
    </tr>
  </thead>
  <tbody>
    <tr th:each="prod : ${allProducts}">
      <td th:text="${prod.name}">Oranges</td>
      <td th:text="${#numbers.formatDecimal(prod.price,1,2)}">0.99</td>
    </tr>
  </tbody>
</table>
```

可以看到Thymeleaf主要以属性的方式加入到html标签中，浏览器在解析html时，当检查到没有的属性时候会忽略，所以Thymeleaf的模板可以通过浏览器直接打开展现，这样非常有利于前后端的分离。

在Spring Boot中使用Thymeleaf，只需要引入下面依赖，并在默认的模板路径`src/main/resources/templates`下编写模板文件即可完成。


```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```
将上面的代码添加到pom.xml文件之后，举一个简单的例子。在之前的示例中，是通过`@RestController`来接收请求，返回的内容为json对象。那么如果现在需要渲染html页面的时候，要如何实现呢？ **`@Controller`**

```
@Controller
public class HelloController {
    @RequestMapping("/")
    public String index(ModelMap map) {
        // 加入一个属性，用来在模板中读取
        map.addAttribute("host", "http://yezhwi.github.com");
        // return模板文件的名称，对应src/main/resources/templates/index.html
        return "index";  
    }
}
```

Thymeleaf页面`index.html`模板：

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8" />
    <title></title>
</head>
<body>
<h1 th:text="${host}">Hello World</h1>
</body>
</html>

```

如上页面，直接打开html页面展现`Hello World`，但是启动程序后，访问`http://localhost:8080/`，则是展示Controller中host的值：`http://yezhwi.github.com`，做到了不破坏HTML自身内容的数据逻辑分离。

### Thymeleaf的默认参数配置

```
# Enable template caching.
spring.thymeleaf.cache=true 
# Check that the templates location exists.
spring.thymeleaf.check-template-location=true 
# Content-Type value.
spring.thymeleaf.content-type=text/html 
# Enable MVC Thymeleaf view resolution.
spring.thymeleaf.enabled=true 
# Template encoding.
spring.thymeleaf.encoding=UTF-8 
# Comma-separated list of view names that should be excluded from resolution.
spring.thymeleaf.excluded-view-names= 
# Template mode to be applied to templates. See also StandardTemplateModeHandlers.
spring.thymeleaf.mode=HTML5 
# Prefix that gets prepended to view names when building a URL.
spring.thymeleaf.prefix=classpath:/templates/ 
# Suffix that gets appended to view names when building a URL.
spring.thymeleaf.suffix=.html  
# Order of the template resolver in the chain. 
spring.thymeleaf.template-resolver-order= 
# Comma-separated list of view names that can be resolved.
spring.thymeleaf.view-names= 
```

到这里已经完成了Web应用的简单开发。有关Thymeleaf的语法，可以参考 [thymeleaf官网](http://www.thymeleaf.org/)，相信有JSTL经验的人看到这些语法很熟悉。

关于页面模板就到这里，因为我们现在的工作方式是为前端提供json接口，前端通过调用接口返回的json数据来渲染页面，当然前端可以有很多的选择，可以选择团队里最熟悉的框架来完成类似的工作。

下一篇计划总结一下工程的目录结构，约定优于配置。

同时希望自己能坚持总结下去，未来能把下面这张图上的内容总结完。

![Spring Cloud组件架构](/img/spring-cloud.jpg)
