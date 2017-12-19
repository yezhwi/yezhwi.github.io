---
layout:     post
title:      SpringBoot-服务端参数验证-JSR-303验证框架
subtitle:   参数的合法性验证是一个必不可少的步骤
date:       2017-11-17
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - Validator
    - BeanValidation
---

作为服务端开发，验证前端传入的参数的合法性是一个必不可少的步骤，但是验证参数基本上是一个体力活，而且冗余代码繁多，也影响代码的可阅读性，所以有没有一个比较优雅的方式来解决这个问题？

JSR-303验证框架，JSR-303 是Java EE 6 中的一项子规范，叫做BeanValidation，官方参考实现是Hibernate Validator（与Hibernate ORM 没有关系），JSR 303 用于对Java Bean 中的字段的值进行验证，确保输入进来的数据在语义上是正确的，使验证逻辑从业务代码中脱离出来。JSR303是运行时数据验证框架，验证之后验证的错误信息会马上返回。

### 试一试

基于spring-boot的验证参数比较简单，在`spring-boot-starter-web`包里面有`hibernate-validator`包，它提供了一系列验证各种参数的方法，所以说spring-boot已经帮我们想好要怎么解决这个问题了。

![validation](/img/validation-api-1.png)

* 在Bean中添加标签

```
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class EntrustQueue implements Serializable {

    @ApiModelProperty(value = "ID", example = "1", allowableValues = "range[0, infinity]")
    private Long id;

    @ApiModelProperty(value = "交易帐号ID", example = "10001", required = true)
    @NotNull(message = "accountId:不能为空")
    private Long accountId;

    @ApiModelProperty(value = "股票代码", example = "600100", required = true)
    @NotNull(message = "stockCode:不能为空")
    private String stockCode;

    @ApiModelProperty(value = "委托数量", example = "100", required = true)
    @NotNull(message = "volume:不能为空")
    private Integer volume;

    @ApiModelProperty(value = "委托价格", example = "10.00", required = true)
    @NotNull(message = "price:不能为空")
    private BigDecimal price;

    @ApiModelProperty(value = "委托类型", example = "1", allowableValues = "1, 2", required = true)
    @Range(min = 1, max = 2, message = "entrustType:数据范围只能取1或2")
    private Integer entrustType;

    private Long createAt;
    private Long updateAt;

}
```

* Controller中开启验证

在Controller 中 请求参数上添加@Validated 标签开启验证

```
@PostMapping("/entrusQueueFeign")
public ResponseEntity<ResultData> addEntrust2(@RequestBody @Validated EntrustQueue entrustQueue) {

    log.info("entrusQueueFeign:" + entrustQueue.toString());

    ResponseEntity<ResultData> resultData = entrustQueueClient.addEntrust(entrustQueue);
    return  resultData;
}
```

* 自定义统一的验证异常处理器，捕获错误信息

```
@Slf4j
@RestController
@ControllerAdvice
public class DefaultExceptionHandler {
    
    ...

    @ExceptionHandler(value = { MethodArgumentNotValidException.class })
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ResultData validException(HttpServletRequest request, Exception ex) {
        log.error(request.getRequestURI()+"?"+request.getQueryString(), ex);
        MethodArgumentNotValidException c = (MethodArgumentNotValidException) ex;
        List<ObjectError> errors =c.getBindingResult().getAllErrors();
        StringBuffer errorMsg = new StringBuffer();
        for (ObjectError error : errors) {
            errorMsg.append(error.getDefaultMessage()).append(";");
        }
        ResultData resultData = ResultData.builder().message(Message.builder().code(HttpStatus.BAD_REQUEST.value()).message(errorMsg.toString()).build()).build();
        return resultData;
    }
}
```

* 运行看看效果

![运行效果](/img/validation-api-2.png)

### 如果参数太少，还需要定义一个对象吗？

比如，接口请求中，就有两个参数再定义对象就太麻烦了！确实，如果只有少数对象，能直接把参数写到Controller层，然后在Controller层进行验证该多好。

JSR和Hibernate validator的校验只能对Object的属性进行校验，不能对单个的参数进行校验，但Spring 在此基础上进行了扩展，添加了MethodValidationPostProcessor拦截器，可以实现对方法参数的校验。

```
@GetMapping(value = "/user")
public User getUser( 
	@Pattern(regexp="^1[3|4|5|7|8][0-9]{9}$",message="请填写合法的手机号")      
	@RequestParam(name = "phone", required = true)    
	String phone) {
	
	return userService.getUser(phone);
}

需要创建一个Bean

@Bean
public MethodValidationPostProcessor methodValidationPostProcessor() {  
    return new MethodValidationPostProcessor();
}

然后在类方法上面加上注解@Validated

@RestController
@RequestMapping("/user")
@Validated
public class UserController {

	...
	
}

```




### 还可以自定义参数验证逻辑，后面再说。

***

## 附上常用标签及含义

| 标签 | 说明 |
| ---- | ---- |
| @Null | 限制只能为null |
| @NotNull | 限制必须不为null |
| @AssertFalse | 限制必须为false |
| @AssertTrue | 限制必须为true |
| @DecimalMax(value) | 限制必须为一个不大于指定值的数字 |
| @Digits(integer,fraction) | 限制必须为一个小数，且整数部分的位数不能超过integer，小数部分的位数不能超过fraction |
| @Future | 限制必须是一个将来的日期 |
| @Past | 限制必须是一个过去的日期，验证注解的元素值（日期类型）比当前时间早 |
| @Max(value) | 限制必须为一个不大于指定值的数字 |
| @Min(value) | 限制必须为一个不小于指定值的数字 |
| @Pattern(value) | 限制必须符合指定的正则表达式 |
| @Size(max,min) | 限制字符长度必须在min到max之间 |
| @NotEmpty | 验证注解的元素值不为null且不为空（字符串长度不为0、集合大小不为0） |
| @NotBlank | 验证注解的元素值不为空（不为null、去除首位空格后长度为0），不同于@NotEmpty，@NotBlank只应用于字符串且在比较时会去除字符串的空格 |
| @Email | 验证注解的元素值是Email，也可以通过正则表达式和flag指定自定义的email格式 |









