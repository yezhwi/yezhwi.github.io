---
layout:     post
title:      RESTful API设计基础认识
subtitle:   
date:       2017-11-16
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
tags:
    - RESTful API
---

我们知道一旦API发布后，就很难对它做很大的改动并且保持像先前一样的正确性。所以在这之前我们要做一些规范。

### 协议

API与用户的通信协议，总是使用HTTPS协议。

所有的访问API行为，都需要用 TLS 通过安全连接来访问。没有必要搞清或解释什么情况需要 TLS 什么情况不需要 TLS，直接强制任何访问都要通过 TLS。

理想状态下，通过拒绝所有非 TLS 请求，不响应 http 或80端口的请求以避免任何不安全的数据交换。如果现实情况中无法这样做，可以返回403 Forbidden响应。

把非 TLS 的请求重定向(Redirect)至 TLS 连接是不明智的，这种含混/不好的客户端行为不会带来明显好处。依赖于重定向的客户端访问不仅会导致双倍的服务器负载，还会使 TLS 加密失去意义，因为在首次非 TLS 调用时，敏感信息就已经暴露出去了。

### 域名

应该尽量将API部署在专用域名之下。`https://api.example.com`

### 版本

应该将API的版本号放入URL。`https://api.example.com/v1/`，[Github](https://developer.github.com/v3/media/#request-specific-version)就是这么做的。

### 路径，URI标识资源

在RESTful架构中，每个网址代表一种资源（resource），所以网址中不能有动词，只能有名词，而且所用的名词往往与数据库的表格名对应。一般来说，数据库中的表都是同种记录的"集合"（collection），所以API中的名词也应该使用复数。如：查询员工的信息`https://api.example.com/v1/employees`

### 对资源操作

对于资源的具体操作类型，由HTTP动词表示。

常用的操作：

```
GET：从服务器查询资源（一项或多项）。
POST：在服务器新建一个资源。
PUT：在服务器更新资源（客户端提供改变后的完整资源）。
PATCH：在服务器更新资源（客户端提供改变的属性）。
DELETE：从服务器删除资源。
```
如：

```
GET /employees/ID：获取某个员工信息
PUT /employees/ID：更新某个员工信息（提供该员工的全部信息）
```

注：必填参数ID在路径中体现。

### 过滤信息（查询筛选条件）

常用的查询筛选条件：

```
?size=10：指定返回记录的数量
?start=10：指定返回记录的开始位置。
?page=2&per_page=100：指定第几页，以及每页的记录数。
?sortby=name&order=asc：指定返回结果按照哪个属性排序，以及排序顺序。
?type_id=1：指定筛选条件
```

注：参数名多个单词间使用下划线；同时注意参数名，避免使用关键字，有可能被waf、网关等拦截。

### 自定义接口返回数据

为了保障前后端的数据交互的顺畅，规范数据的返回，并采用固定的数据格式封装。

接口返回模板:

```
{
  "message": {
    "code": 0,
    "message": "success"
  },
  "data": {...}
}
```

### 自定义组合api

把客户端需请求的数据内容加载的多个接口合并成一个请求发送到服务端，服务端根据请求内容，一次性把所有数据合并返回，相比于页面级API（页面排版修改，API就需要一起修改），具备更高的组合数据灵活性，同时又能很容易的实现页面级的API功能。

```
合并接口入参

data:[
    {url:'api1',type:'get',data:{...}},

    {url:'api2',type:'get',data:{...}},

    {url:'api3',type:'get',data:{...}},

    {url:'api4',type:'get',data:{...}}
]

返回数据：

{
  "message": {
    "code": 0,
    "message": "success"
  },
  "data": [
	  {
		  "message": {
		    "code": 0,
		    "message": "success"
		  },
		  "data": {...}
		}
		{
		  "message": {
		    "code": -1,
		    "message": "failure"
		  },
		  "data": null
		}
  ]
}

```

### 使用“链接”关联相关的资源的API

RESTful API最好做到Hypermedia，即返回结果中提供链接，连向其他API方法，使得用户不查文档，也知道下一步应该做什么。

Github的API就是这种设计，访问[api.github.com](https://api.github.com/)会得到一个所有可用API的网址列表。

```
{
  "current_user_url": "https://api.github.com/user",
  "authorizations_url": "https://api.github.com/authorizations",
  // ...
}
```
从上面可以看到，如果想获取当前用户的信息，应该去访问[api.github.com/user](https://api.github.com/user)，然后就得到了下面结果。表示，服务器给出了提示信息，以及文档的网址。

```
{
  "message": "Requires authentication",
  "documentation_url": "https://developer.github.com/v3"
}
```

优势：服务端返回url访问地址，此地址中可以由服务器分配一个类似随机字符串的参数，做为服务端验证请求的有效性、访问频率、幂等操作等。

### HTTP状态码

常用的有：

```
200 OK - [GET]：服务器成功返回用户请求的数据，该操作是幂等的（Idempotent）。
201 CREATED - [POST/PUT/PATCH]：用户新建或修改数据成功。
202 Accepted - [*]：表示一个请求已经进入后台排队（异步任务）
204 NO CONTENT - [DELETE]：用户删除数据成功。
400 INVALID REQUEST - [POST/PUT/PATCH]：用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的。
401 Unauthorized - [*]：表示用户没有权限（令牌、用户名、密码错误）。
403 Forbidden - [*] 表示用户得到授权（与401错误相对），但是访问是被禁止的。
404 NOT FOUND - [*]：用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的。
406 Not Acceptable - [GET]：用户请求的格式不可得（比如用户请求JSON格式，但是只有XML格式）。
410 Gone -[GET]：用户请求的资源被永久删除，且不会再得到的。
422 Unprocesable entity - [POST/PUT/PATCH] 当创建一个对象时，发生一个验证错误。
500 INTERNAL SERVER ERROR - [*]：服务器发生错误，用户将无法判断发出的请求是否成功。
```
