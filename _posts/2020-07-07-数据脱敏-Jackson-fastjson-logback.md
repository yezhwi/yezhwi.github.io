---
layout:     post
title:      数据脱敏-Jackson-Fastjson-Logback
subtitle:   
date:       2020-07-07
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
    - 架构
---

> 原文地址：https://blog.csdn.net/qq_26418435/article/details/103620548

### 学习思路

* 分析脱敏场景
* 基于 Fastjson、Jackson、Logback 的各种实现
* 总结
* 文末有代码实现 git 地址、小星星

### 一、分析脱敏场景

生产数据，为了保护用户信息，防止用户信息泄露，我们通常需要对数据进行脱敏主要有（手机号、身份证、姓名等）

1. 打印日志脱敏，日志中看到的信息不是完整的比如：183****0001
1. 接口返回信息脱敏，比如用户手机号、银行卡号、身份证等
1. 数据库脱敏存储

题外话：数据库存储目前用的比较多的是密码加密，其它的数据牵扯到查询，加密成本比较高，`sharding-jdbc` 自带数据加密存储及查询，有兴趣的可以了解一下，我们本节主要讲解前两种

### 二、基于 Fastjson、Jackson、Logback 的各种实现

#### 1、Fastjson 实现

* 实现思路：

1.  自定义注解，可让用户自定义脱敏方式，用于实体类的属性
2.  基于 `ValueFilter` 进行属性注解拦截，并对 `value` 进行替换脱敏
3.  使用 `json` 序列化对象是指定自定义序列化 `Filter`

题外话：`ValueFilter`：对象值过滤器，将要序列化对象的值进行统一处理

* 代码实现：

首先自定义注解

```  
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Desensitization {​
    /**
     * 脱敏规则类型
     * @return
     */
    DesensitionType type();​
    /**
     * 附加值, 自定义正则表达式等
     * @return
     */
    String[] attach() default "";
​}
```

解释：

1. 脱敏规则类型：主要定义了一些常用的类型（手机号、身份证等）
2. 自定义正则表达式，如果常用的不能满足时可自定义

看一下脱敏规则类型

```
public enum DesensitionType { 

    /**
     * 手机号脱敏
     * "(13[0-9]|14[579]|15[0-3,5-9]|16[6]|17[0135678]|18[0-9]|19[89])\\d{8}"
     * "(\\d{3})\\d{4}(\\d{4})"
     */
    PHONE("mobile", "11位手机号", "(13[0-9]|14[579]|15[0-3,5-9]|16[6]|17[0135678]|18[0-9]|19[89])\\d{4}(\\d{4}}", "$1****$2"), 
    IDENTITYNO("identityNo", "15或者18身份证号", "(\\w{4})\\w{7,10}(\\w{4})", "$1****$2"), 
    BANKCARDNO("bankCardNo", "银行卡号", "(\\d{4})\\d*(\\d{4})", "$1****$2"), 
    REALNAME("realname","真实姓名Json类型","(\"realname\":)(\"[\u4E00-\u9FA5]{1})[\u4E00-\u9FA5]{1,}(\")","$1$2**$3"), 
    REALNAME2("realname","真实姓名toString类型","(realname=)([\u4E00-\u9FA5]{1})[\u4E00-\u9FA5]{1,}","$1$2**"), 
    CUSTOM("custom", "自定义正则处理", ""), 
    TRUNCATE("truncate", "字符串截取处理", ""), 
    ;  

    String type; 
    String describe; 
    String[] regular; 

    DesensitionType(String type, String describe, String... regular) { 
        this.type = type; 
        this.describe = describe; 
        this.regular = regular; 
    }
}
```

这里主要注意一下第三个参数：是一个数组，通常 0 位：要脱敏数据的正则匹配，1 位：要脱敏成的格式

然后在我们的需要脱敏的对象字段上加上该注解

```
@Data
public class UserDTO implements Serializable {
    @Desensitization(type = DesensitionType.IDENTITYNO)
    private String identityNo;
    private String name;
    private String realname;
}
```

接下来编写我们的自定义值过滤器，实现 `ValueFilter`，实现方法 `process()`

```
@Log4j2
public class FastjsonDesensitizeFilter implements ValueFilter,DesensitizeService { 

@Override 
public Object process(Object object, String name, Object value) {
    if (null == value || !(value instanceof String) || ((String) value).length() == 0) {
        return value;
    }

    try {
        Field field = object.getClass().getDeclaredField(name);
        Desensitization desensitization;
            
        if (String.class != field.getType() || (desensitization = field.getAnnotation(Desensitization.class)) == null) {
            return value;
        }
        DesensitionType type = desensitization.type();
        List<String> regular = this.desensitize(type, desensitization);
    
        if (regular.size() > 1) {
            String match = regular.get(0);
            String result = regular.get(1);
            if (null != match && result != null && match.length() > 0) {
                return ((String) value).replaceAll(match, result);
            }
        }
    } catch (Exception e) {
        log.warn("FastJsonDesensitizeFilter the class {} has no field {}", object.getClass(), name);
    }
    return value;
}
```

解释：

1. 这里目前只支持 `String` 类型的 value，大家可以根据需要自定义
2. 获取属性上的注解，根据属性得到相应的脱敏规则类型
3. 按照规则类型进行 value 替换

然后只要我们在使用 `fastjson` 进行序列化的时候指定我们的自定义过滤器即可

```
public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) { 
    log.info("requestUrl 【{}】 request body 【{}】", request.getURI(), JSONObject.toJSONString(body, new FastjsonDesensitizeFilter())); 
    return body;
}
```

打印出来就是这样

```
requestUrl 【http://localhost:8080/idNo】 request body 【{"code":"000000","data":{"identityNo":"1111****1111","name":"dsf","realname":"张**"},"message":"SUCCESS"}】
```

#### 2、基于 Jackson 实现数据脱敏

思路跟用 Fastjson 基本一样，只是实现的类不同而已

```
public class JacksonDesensitize extends JsonSerializer<String> implements ContextualSerializer, DesensitizeService {

    private DesensitionType type;

    @Override
    public void serialize(String value, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
    
        if (type != null){
            try {
                List<String> regular = this.desensitize(type, null);
                if (regular.size() > 1) {
                    String match = regular.get(0);
                    String result = regular.get(1);
                    if (null != match && result != null && match.length() > 0) {
                        jsonGenerator.writeString(value.replaceAll(match, result));
                    }
                }
            } catch (Exception e) {
                log.warn("JacksonDesensitize has no field {}",  value);
            }
        }
    }​

    @Override
    public JsonSerializer<?> createContextual(SerializerProvider serializerProvider, BeanProperty beanProperty) throws JsonMappingException {
        type = beanProperty.getAnnotation(Desensitization.class).type();
        return this;
    }
}
```

解释：

1. 获取对象属性上的注解，根据属性得到相应的脱敏规则类型
2. 按照规则类型进行 value 替换

题外话：这里主要牵扯到要继承 `JsonSerializer` 重写 `serialize()` 方法实现对象序列化，实现 `ContextualSerializer` 接口，实现方法，这个主要是拿到属性上的注解

接下来引用我们的自定义序列化器即可，直接在自定义注解上引用即可

```
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@JsonSerialize(using = JacksonDesensitize.class)
@JacksonAnnotationsInside
public @interface Desensitization {
​
    /**
     * 脱敏规则类型
     * @return
     */
    DesensitionType type();
​
    /**
     * 附加值, 自定义正则表达式等
     * @return
     */
    String[] attach() default "";
​
}
```

解释：

1. 这里主要看 `@JsonSerialize(using = JacksonDesensitize.class)` 就是让 json 序列化使用我们自定义的
2. `@JacksonAnnotationsInside` 这个是注解组合出现时用的

我们现在的项目基本都是前后端分离，`Controller` 返回的时候一般都是 `ResponseBody` 的，正好我们 `SpringMVC` 后台是默认使用 `Jackson` 作为序列化的，所以这时候就可以直接使用

返回就是这样的

```
{
    "code": "000000",
    "message": "SUCCESS",
    "data": {
        "identityNo": "1111****1111",
        "name": "dsf",
        "realname": "张三"
    }
}
```

#### 3、基于 Logback 进行全局日志脱敏

思路

1. 我们先定义需要脱敏的属性名，就是你真正要打印到日志的属性名字
2. 然后继承 Logback 的 `MessageConverter` 重写 `convert` 方法
3. 通过正则进行身份证、姓名、手机号的匹配
4. 匹配成功后按规则替换

```
public class LogDesensitizeConverter extends MessageConverter {

    /**
     * 日志脱敏开关
     */
    private static Boolean converterCanRun = Boolean.TRUE;

    /**
     * 日志脱敏关键字
     */
    @Override
    public String convert(ILoggingEvent event) {
        // 获取原始日志
        String oriLogMsg = event.getFormattedMessage();

        if (!converterCanRun){
            return oriLogMsg;
        }
        // 获取脱敏后的日志
        DesensitionType[] values = DesensitionType.values();
        for (DesensitionType value : values) {
            if (value.getRegular() != null && value.getRegular().length > 0 && oriLogMsg.contains(value.getType())) {
                Matcher matcher = Pattern.compile(value.getRegular()[0]).matcher(oriLogMsg);
                oriLogMsg = matcher.replaceAll(value.getRegular()[1]);
            }
        }
        return oriLogMsg;
    }
}
```    

然后将我们编写完的 `Converter` 添加到 `logback.xml` 文件引用即可

![](https://img-blog.csdnimg.cn/2019121919244285.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2NDE4NDM1,size_16,color_FFFFFF,t_70)

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xbG9nby5jbi9tbWJpel9wbmcvWUxEaWJxR3k3bUs2OUlaaE5FY25FNmdoWmtjUmEwWDBVSmVaMEM3bFdGS0J1YjJOd3N5bnBtNk05aWNJN2hNZ2hZNW83bzdRZElDa3ZiSmRlQXhuZTZ2US8w?x-oss-process=image/format,png)

之后接口用了，日志文件里是这样的

![](https://img-blog.csdnimg.cn/20191219192417854.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2NDE4NDM1,size_16,color_FFFFFF,t_70)

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xbG9nby5jbi9tbWJpel9wbmcvWUxEaWJxR3k3bUs2OUlaaE5FY25FNmdoWmtjUmEwWDBVTFJ3TlE4QWVFREJ6ejQ3eXpJN1J2c2QyVTB4WGcxd1ZlQ3BsdDA1dXk0SFlMa0ZGWktCODRBLzA?x-oss-process=image/format,png)

### 三、总结

1. `SpringMVC` 默认使用 `Jackson` 作为对象序列化，如果想要使用 `Fastjson` 需要单独配置，然后指定我们的自定义序列化器就可以了
2. 如果单纯的只用 `Fastjson` 打印日志那么建议在拦截器，或者像本文代码中实现 `ResponseBodyAdvice` 去集中打印，不然还要每次都加我们的自定义序列化器
3. 我们使用 `Jackson` 和 `Fastjson` 时使用了自定义注解，当然也可以根据自己的业务提前定义属性值，像 `Logback` 的方式一样实现也可
4. 复杂的正则很耗 cpu 但我们的非常简单，并且都有提前过滤

### 四、代码

git 地址：https://gitee.com/carpentor/spring-cloud-example.git

> 如果觉得还有帮助的话，你的关注和转发是对我最大的支持，O(∩_∩)O:



