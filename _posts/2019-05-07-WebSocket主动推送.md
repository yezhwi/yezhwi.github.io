---
layout:     post
title:      WebSocket 主动推送
subtitle:   
date:       2019-05-07
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - WebSocket
---


### 背景

最近，公司在开发一款主动服务机器人，将产生的消息数据通过服务端推送到客户端（H5、iOS、Android）；同时支持用户进行提问，通过 NLP 识别用户意图，然后找到答案并回答用户。本次仅记录在进行部署主动服务通信服务时遇到的问题。

### 服务器向 Web 页面推送消息的方式

* 非阻塞轮询（短轮询）：客户端以固定的频率（比如10秒钟一次）向服务端发送请求，如果服务端没有数据响应，就直接响应一个空，如果有数据响应，就将响应数据作为结果返回给客户端。特点是每次请求后，都会立即给响应。
* 阻塞长轮询（长轮询）：客户端像传统轮询一样从服务器请求数据。如果服务器没有可以立即返回给客户端的数据，则不会立刻返回一个空结果，而是保持这个请求等待数据到来（请求阻塞或者超时），等有响应数据之后将数据作为结果返回给客户端。特点是一次请求后直到有响应数据时才会给返回，否则阻塞等待。

### 轮询并不是最好的解决方案

#### 短轮询
优点是实现逻辑简单，但是当间隔太短时，会有大量的请求发送到服务器，会对服务器负载造成影响；而间隔太长，业务数据的实时性得不到保证无效请求的数量多。在用户量较大的情况下，服务器负载较高。

#### 长轮询
优点是消息实时性高，无消息的情况下不会进行频繁的请求；缺点是服务端维持和客户端的连接会消耗掉一部分资源。

### WebSocket ，一种高效节能的双向通信机制来保证数据的实时传输

* WebSocket 是 HTML5 一种新的协议（Web TCP）。它建立在 TCP 之上，实现了客户端和服务端全双工异步通信.
* 它和 HTTP 最大不同是：

	WebSocket 是一种双向通信协议，WebSocket 服务器和 Browser/Client Agent 都能主动的向对方发送或接收数据；
	
	WebSocket 需要类似 TCP 的客户端和服务器端通过握手连接，连接成功后才能相互通信。
	
### 服务端与客户端通信架构

![](https://tva2.sinaimg.cn/large/006tNc79ly1g2rnwhopiyj30j40eit9k.jpg)

### 技术选型

* socketio
* 前端 socket.io.js
* 服务端 Netty-SocketIO
* 基础框架 SpringBoot

### 客户端自动断开连接，怎么办？

> 在自己电脑上把服务端启动后，然后客户端之间进行交流发送消息没有任何问题，但是通过 Nginx 代理后，发现客户端自动断开连接。

#### 确认HTTP 1.1 版本

通过查看 Nginx 的日志，发现 HTTP 协议为 1.0 版本，因为在HTTP1.0 中，客户端发送请求，服务器接收请求，双方建立连接，服务器响应资源，请求结束；而在HTTP 1.1中，客户端发出请求，服务端接收请求，双方建立连接，在服务端没有返回之前保持连接，当客户端再发送请求时，它会使用同一个连接。这一直继续到客户端或服务器端认为会话已经结束，其中一方中断连接。参考〈完整的配置 #1〉

#### 增加 WebSocket 的配置

Nginx 通过在客户端和后端服务器之间建立起一条隧道来支持WebSocket。为了使 Nginx 可以将来自客户端的 Upgrade 请求发送给后端服务器，Upgrade 和 Connection 的头信息必须被显式的设置。参考〈完整的配置 #2〉

#### 完整的配置

```
location /  {
        proxy_pass http://chat-server:31001/;
        proxy_redirect off;
        #1
        proxy_http_version 1.1;

        #2
        proxy_set_header Upgrade $http_upgrade; #WebSocket配置
        proxy_set_header Connection "upgrade"; #WebSocket配置

        proxy_read_timeout 90;
}
```

### 问题总结

#### H5、iOS、Android 支持

* iOS，Android 端的 WebView 绝大部分支持。
       
	不支持的有：
	
		iOS7 之前的手机（可以不用考虑），iOS7 是2013年9月发布的。
		
		Android5.0 之前的手机，2015年3月发布的系统。

* PC端浏览器

	一些版本的浏览器不支持，大部分的发布时间在2012年之前的，其余主流浏览器都支持
	
	支持度可以参考这个：https://www.caniuse.com/#search=websocket












