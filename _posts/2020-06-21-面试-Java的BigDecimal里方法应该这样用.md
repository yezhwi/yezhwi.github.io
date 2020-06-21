---
layout:     post
title:      Java的BigDecimal里方法应该这样用
subtitle:   
date:       2020-06-21
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
    - 面试
---

### 预测一下运行的结果

在运行下面的代码之前，先把自己预测的结果写下来，看看能对几个

```
System.out.println(BigDecimal.ZERO.equals(BigDecimal.ZERO));
System.out.println(BigDecimal.ZERO.equals(new BigDecimal(0)));
System.out.println(BigDecimal.ZERO.equals(new BigDecimal(0.00)));
System.out.println("=====================");
System.out.println(BigDecimal.ZERO.equals(new BigDecimal("0.0")));
System.out.println(BigDecimal.ZERO.equals(new BigDecimal("0.00")));
System.out.println("=====================");
System.out.println(BigDecimal.ZERO.compareTo(new BigDecimal(0)));
System.out.println(BigDecimal.ZERO.compareTo(new BigDecimal(0.00)));
System.out.println(BigDecimal.ZERO.compareTo(new BigDecimal("0.00")));

```

运行结果：

```
true
true
true
=====================
false
false
=====================
0
0
0
```

### BigDecimal 最佳实践

`BigDecimal` 是为了实现进行 double 和 long 的数值运算引入的，但是在实际使用过程中发现与预想的结果并不一样。

示例：

```
BigDecimal a = new BigDecimal(0.1);
System.out.println("a values is:"+a);
```

运行结果：

```
a values is:0.1000000000000000055511151231257827021181583404541015625

```

> 原因请参考上篇文章：[Java中的使用BigDecimal就不会丢失精度了吗？](https://yezhwi.github.io/java/2020/06/19/%E9%9D%A2%E8%AF%95-Java%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8BigDecimal%E5%B0%B1%E4%B8%8D%E4%BC%9A%E4%B8%A2%E5%A4%B1%E7%B2%BE%E5%BA%A6%E4%BA%86%E5%90%97/)


#### equals 和 compareTo

比较是否相等应该使用 `compareTo` 来实现，而不是 `equals `，通过最开始的测试示例代码可以看到两者的区别

#### 判断是否为 0

通过 `BigDecimal.ZERO` 和 `compareTo` 方法来实现：

```
System.out.println(BigDecimal.ZERO.compareTo(new BigDecimal(0)) == 0);
System.out.println(BigDecimal.ZERO.compareTo(new BigDecimal(0.00)) == 0);
System.out.println(BigDecimal.ZERO.compareTo(new BigDecimal("0.00")) == 0);
```

#### 除法

通过 `BigDecimal` 的 `divide` 方法进行除法时当不整除，出现无限循环小数时，就会抛异常：java.lang.ArithmeticException: Non-terminating decimal expansion; no exact representable decimal result.

需要通过对 `divide` 方法设置精确的小数点解决。

```
public BigDecimal divide(BigDecimal divisor, int scale, int roundingMode)
```

> （1）scale 小数点后几位
> 
> （2）roundingModel 保留位后面小数舍去的方式
> 
> 	* BigDecimal.ROUND_DOWN 直接舍去
> 	* BigDecimal.ROUND_UP 直接向上进位向上取整。如：1.1处理后是2，-1.1处理后是-2。
> 	* BigDecimal.ROUND_HALF_UP 四舍五入

#### 小数相关操作

当需要使用小数构造一个 `BigDecimal` 时，建议使用参数类型为 `String ` 的构造函数或 `BigDecimal.valueOf(double val);`，源码如下，内部先做了 String 转化。

```
public static BigDecimal valueOf(double val) {
    // Reminder: a zero double returns '0.0', so we cannot fastpath
    // to use the constant ZERO.  This might be important enough to
    // justify a factory approach, a cache, or a few private
    // constants, later.
    return new BigDecimal(Double.toString(val));
}
```

#### BigDecimal 的不可变性

进行每一次四则运算时，都会产生一个新的对象 ，所以在做加减乘除运算时要记得要保存操作后的值

```
BigDecimal b1 = new BigDecimal("1");
BigDecimal b2 = new BigDecimal("2");
System.out.println(b1.add(b2));
System.out.println(b1);
```

### 总结

因为在一些需要计算金钱或着财经类（股票、基金等）数据时可能使用 `BigDecimal` 的场景较多

1. 在需要精确的小数计算时再使用 `BigDecimal`，性能比 `double` 和 `float` 差，在处理庞大，复杂的运算时尤为明显。故一般精度的计算没必要使用 `BigDecimal`。
1. 比较是否相等应该使用 `compareTo` 来实现。
1. 通过 `BigDecimal` 的 `divide` 方法进行除法时，要设置保底小数位数。
1. 尽量使用参数类型为 `String` 的构造函数或 `BigDecimal.valueOf(double val);`。
1. `BigDecimal` 都是不可变的，在进行每一次四则运算时，都会产生一个新的对象，所以在做加减乘除运算时要记得要保存操作后的值。

相关推荐

[Java中的使用BigDecimal就不会丢失精度了吗？](https://yezhwi.github.io/java/2020/06/19/%E9%9D%A2%E8%AF%95-Java%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8BigDecimal%E5%B0%B1%E4%B8%8D%E4%BC%9A%E4%B8%A2%E5%A4%B1%E7%B2%BE%E5%BA%A6%E4%BA%86%E5%90%97/)


> 如果觉得还有帮助的话，你的关注和转发是对我最大的支持，O(∩_∩)O:



