---
layout:     post
title:      SpringCloud智能网关-核心功能过滤器
subtitle:   
date:       2017-12-14
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springcloud
tags:
    - SpringCloud 
    - Zuul 
---

### 过滤器

在上一篇我们通过使用Spring Cloud Zuul构建了一个基础的API网关服务，同时也演示了Spring Cloud Zuul基于服务的自动路由功能。然而，目前的服务路由并没有限制权限这样的功能，所有请求都会被毫无保留地转发到具体的应用并返回结果，为了实现对客户端请求的安全校验和权限控制，需要为微服务实现一套用于校验签名和鉴别权限的过滤器或拦截器。由于网关服务的加入，外部客户端访问有统一入口，Zuul允许开发者在API网关服务通过定义过滤器来实现对请求的拦截与过滤，实现的方法非常简单，只需要继承ZuulFilter抽象类并实现它定义的四个抽象函数就可以完成对请求的拦截和过滤了。


### 过滤器实现

定义一个简单的Zuul过滤器，它实现的功能是把请求被路由之前检查HttpServletRequest中是否有accessToken参数，如果有就进行路由，调用真正的服务。否则就拒绝访问，并返回401 Unauthorized错误。

通过继承ZuulFilter抽象类并重写了下面的四个方法来实现自定义的过滤器，代码如下：

```

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * @author Yezhiwei
 * @date 17/12/13
 */
@Slf4j
@Component
public class AccessFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        log.info(String.format("%s request to %s", request.getMethod(), request.getRequestURL().toString()));
        Object accessToken = request.getParameter("accessToken");
        if(accessToken == null) {
            log.warn("access token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
//            ctx.set("error.status_code", HttpServletResponse.SC_UNAUTHORIZED);
//            ctx.set("error.message", "UNAUTHORIZED");
//            ctx.set("error.exception", "accessToken is null");            			   throw new RuntimeException();
        }
        log.info("access token ok");
        return null;
    }
}

```

* `filterType`：过滤器的类型，它决定过滤器在请求的哪个生命周期中执行。在Zuul中默认定义了四种不同生命周期的过滤器类型，这里定义为pre，代表会在请求被路由之前执行。

>pre：可以在请求被路由之前调用。
>
>routing：在路由请求时候被调用。
>
>post：在routing和error过滤器之后被调用。
>
>error：处理请求时发生错误时被调用。

* `filterOrder`：过滤器的执行顺序。当请求在一个阶段中存在多个过滤器时，需要根据该方法返回的值来依次执行。通过int值来定义过滤器的执行顺序，数值越小优先级越高。

* `shouldFilter`：判断该过滤器是否需要被执行。这里我们直接返回了true，因此该过滤器对所有请求都会生效。实际运用中我们可以利用该函数来指定过滤器的有效范围。

* `run`：过滤器的具体逻辑。这里我们通过`ctx.setSendZuulResponse(false)`令zuul过滤该请求，不对其进行路由，然后通过`ctx.setResponseStatusCode(401)`设置了其返回的错误码。

### 关键代码解释

* 在实现了自定义过滤器之后，它并不会直接生效，我们还需要为其创建具体的Bean才能启动该过滤器，如`@Component`

* 核心过滤器中的`SendErrorFilter`是用来处理异常信息的，详细看看SendErrorFilter源码的shouldFilter方法发现

```
public boolean shouldFilter() {
    RequestContext ctx = RequestContext.getCurrentContext();
    return ctx.getThrowable() != null && !ctx.getBoolean("sendErrorFilter.ran", false);
}
```

可以看到该方法的返回值中有一个重要的判断依据`ctx.getThrowable()`。

* `SendErrorFilter` 源码中错误路径

```
@Value("${error.path:/error}")
private String errorPath;
```

* 所以还需要对原来的统一异常处理类进行扩展，并且自定义异常信息。

```
@RestController
public class HttpNotFountConfig implements ErrorController {

    private final static String ERROR_PATH = "/error";

    @RequestMapping(value = ERROR_PATH)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ResultData error(HttpServletRequest request) {
        RequestContext ctx = RequestContext.getCurrentContext();
        if (null != ctx) {
            return ResultData.builder().message(Message.builder().code(ctx.getResponse().getStatus()).message(HttpStatus.valueOf(ctx.getResponse().getStatus()).getReasonPhrase()).build()).build();
        }
        return ResultData.builder().message(Message.builder().code(HttpStatus.NOT_FOUND.value()).message(HttpStatus.NOT_FOUND.getReasonPhrase()).build()).build();
    }

    @Override
    public String getErrorPath() {
        return ERROR_PATH;
    }
}
```

在原来的基础上增加了如下代码

```
RequestContext ctx = RequestContext.getCurrentContext();
if (null != ctx) {
    return ResultData.builder().message(Message.builder().code(ctx.getResponse().getStatus()).message(HttpStatus.valueOf(ctx.getResponse().getStatus()).getReasonPhrase()).build()).build();
}
```

### 测试

* url中有规定的`accessToken `参数，正常返回结果，如下图

![success](/img/unauthorized-0.png)

* url中无规定的`accessToken `参数，返回401，如下图

![401](/img/unauthorized-1.png)



