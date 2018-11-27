---
layout:     post
title:      使用Java 8 Stream像操作SQL一样处理数据（上）
subtitle:   
date:       2018-11-28
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
---

> 原文出自：https://my.oschina.net/liuyatao19921025/blog/1608232

### 背景

几乎每个 Java 应用都要创建和处理集合。集合对于很多编程任务来说是一个很基本的需求。举个例子，在银行交易系统中你需要创建一个集合来存储用户的交易请求，然后你**需要遍历整个集合**才能找到这个客户这段时间总共花费了多少金额。尽管集合非常重要，但是在 Java 中对集合的操作并不完美。

首先，**对一个集合处理的模式应该像执行 SQL 语言操作一样可以进行比如查询（一行交易中最大的一笔）、分组（用于消费日常用品总金额）这样的操作**。大多数据库也是可以有明确的相关操作指令，比如 "SELECT id, MAX(value) from transactions" SQL 查询语句可以让你找到所有交易中最大的一笔交易和其 ID。

正如你所看到的，我们**不需要去实现怎样计算最大值（比如循环和变量跟踪得到最大值）。我们只需要表达我们期待什么**。那么为什么我们不能实现与数据库查询方式相似的方式来设计实现集合呢？

其次，我们应该怎么有效处理很大数据量的集合呢？**要加速处理的理想方式是采用多核架构 CPU，但是编写并行代码很难而且会出错。**

Java 8 将能够完美解决这这个问题！**_Stream 的设计可以让你通过陈述式的方式来处理数据。Stream 还能让你不写多线程代码也是可以使用多核架构_**。听起来很棒不是吗？这将是这系列文章将要探索的主要内容。

在我们探索我们怎么样使用 Stream 之前，我们先看一个使用 Java 8 Stream 的新的编程模式。我们需要找出所有银行交易中类型是 grocery 的，并且以交易金额的降序的方式返回交易ID。在 Java 7 中我们需要这样实现：

```
List<Transaction> groceryTransactions = new Arraylist<>();
for(Transaction t: transactions){
  if(t.getType() == Transaction.GROCERY){
    groceryTransactions.add(t);
  }
}
Collections.sort(groceryTransactions, new Comparator(){
  public int compare(Transaction t1, Transaction t2){
    return t2.getValue().compareTo(t1.getValue());
  }
});
List<Integer> transactionIds = new ArrayList<>();
for(Transaction t: groceryTransactions){
  transactionsIds.add(t.getId());
}
```

在 Java 8 中这样就可以实现：

```
List<Integer> transactionsIds =
    transactions.stream()
                .filter(t -> t.getType() == Transaction.GROCERY)
                .sorted(comparing(Transaction::getValue).reversed())
                .map(Transaction::getId)
                .collect(toList());
```

下图展示了 Java 8 的实现代码，首先，我们使用 stream() 函数从一个交易明细列表中获取一个 Stream 对象。接下来是一些操作（filter，sorted，map，collect）连接在一起形成了一个管道，**管道可以被看做是类似数据库查询数据的一种方式**。

![](https://ws1.sinaimg.cn/large/006tNbRwly1fxn1iqd8jkj30x20aa0td.jpg)

**那么怎么处理并行代码呢**？在 Java 8 中非常简单：只需要使用 parallelStream() 取代 stream() 就可以了，如下面所示，Stream API 将在内部将你的查询条件分解应用到多核上。

```
List<Integer> transactionsIds =
    transactions.parallelStream()
                .filter(t -> t.getType() == Transaction.GROCERY)
                .sorted(comparing(Transaction::getValue).reversed())
                .map(Transaction::getId)
                .collect(toList());
```

你可以把 Stream 看做是一种对集合数据提高效能、提供像 SQL 操作一样的抽象概念，这个像 SQL 一样的操作可以使用 lambda 表达式表示。

在这一系列关于 Java 8 Stream 文章的结尾，你将会使用 Stream API 写类似于上述代码来实现强大的查询功能。

### 开始使用Stream

我们先以一些理论作为开始。Stream 的定义是什么？一个简单的定义是："对一个源中的一系列元素进行聚合操作。"把概念拆分一下：

* 一系列元素：Stream 对一组有特定类型的元素提供了一个接口。但是 Stream 并不真正存储元素，元素根据需求被计算出结果。
* 源：Stream 可以处理任何一种数据提供源，比如集合、数组，或者 I/O 资源。
* 聚合操作：Stream 支持类似 SQL 一样的操作，常规的操作都是函数式编程语言，比如 filter，map，reduce，find，match，sorted，等等。

Stream 操作还具备两个基本特性使它与集合操作不同：

* 管道：许多 Stream 操作会返回一个 Stream 对象本身。这就允许所有操作可以连接起来形成一个更大的管道。这就就可以进行特定的优化了，比如懒加载和短回路，我们将在下面介绍。
* 内部迭代：和集合的显式迭代（外部迭代）相比，Stream 操作不需要我们手动进行迭代。

让我们再次看一下之前的代码的一些细节：

![](https://ws1.sinaimg.cn/large/006tNbRwly1fxn1m7dm2wj30x00mu421.jpg)

我们首先通过 stream() 函数从一个交易列表中获取一个 Stream 对象。这个数据源是一个交易的列表，将会为 Stream 提供一系列元素。接下来，我们对 Stream 对象应用一些列的聚合操：filter（通过给定一个谓词来过滤元素），sorted（通过给定一个比较器实现排序），和 map(用于提取信息)。除了 collect 其他操作都会返回 Stream，这样就可以形成一个管道将它们连接起来，我们可以把这个链看做是一个对源的查询条件。

在 collect 被调用之前其实什么实质性的东西都都没有被调用。 collect 被调用后将会开始处理管道，最终返回结果（结果是一个list）。

在我们探讨 Stream 的各种操作前，我们还是看一个 Stream 和Collection 的概念层面的不同之处吧。

### Stream VS Collection

Collection 和 Stream 都对一些列元素提供了一些接口。他们的不同之处是：**Collection 是和数据相关的，Stream 是和计算相关**的。

想一下存在 DVD 中的电影，这是一个 Collection，因为他包含了所有的数据结构。然而网络上的电影是一种流数据。流媒体播放器只需要在用户观看前先下载一些帧就可以观看了，不必全都下载下来。

简单点说，Collection 是一个内存中的数据结构，Collection 包括数据结构中的所有值——每个 Collection 中的元素在它被添加到集合中之前已经被计算出来了。相反，Stream 是一种当需要的时候才会被计算的数据结构。

使用 Collection 接口需要用户做迭代（比如使用foreach），这种方式叫外部迭代。相反，Stream 使用的是内部迭代——它会自己为你做好迭代，并且帮助做好排序。你只需要提供一个函数说明你想要干什么。下面代码使用 Collection 做外部迭代：

```
List<String> transactionIds = new ArrayList<>();
for(Transaction t: transactions){
    transactionIds.add(t.getId());
}
```

下面代码使用 Stream 做内部迭代

```
List<Integer> transactionIds =
    transactions.stream()
                .map(Transaction::getId)
                .collect(toList());
```

### 使用 Stream 处理数据

Stream 接口定义了许多操作，可以被分为两类。

* filter，sorted，和 map，这些可以连接起来形成一个管道的操作
* collect，可以关闭管道返回结果的操作

可以被连接起来的操作叫做中间操作。你可以把他们连接起来，因为他们返回都类型都是 Stream。关闭管道的操作叫做终结操作。他们可以从管道中产生一个结果，比如一个 List，一个 Integer，甚至一个 void。

**中间操作其实不执行任何处理直到一个终结操作被调用；他们很“懒”。因为终结操作通常可以被合并，并且被终结操作一次性执行。**

```
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8);
List<Integer> twoEvenSquares = 
    numbers.stream()
           .filter(n -> {
                    System.out.println("filtering " + n); 
                    return n % 2 == 0;
                  })
           .map(n -> {
                    System.out.println("mapping " + n);
                    return n * n;
                  })
           .limit(2)
           .collect(toList());
```

上面的代码会计算集合中的前两个偶数，执行结果如下：

```
filtering 1
filtering 2
mapping 2
filtering 3
filtering 4
mapping 4
```

这是因为 limit(2) 使用了**短回路**；我们只需要处理 Stream 的一部分，然后并返回结果。这就像要计算一个很大的 Boolean 表达式：只要一个表达式返回 false，我们就可以断定这个表达式将会返回 false 而不需要计算所有。这里 limit 操作返回一个大小为 2 的 Stream。还有就是 filter 操作和 map 操作合并起来一起传给给了 Stream。

总结一下我们现已经已经学到的东西：Stream 的操作包括如下三个东西：

* 一个需要进行数据查询的数据源（比如一个 Collection）
* 一连串组成管道的中间操作
* 一个执行管道并产生结果的终结操作

Stream 提供的操作可分为如下四类：

过滤: 有如下几种可以过滤操作

* filter(Predicate)：使用一个谓词java.util.function.Predicate 作为参数，返回一个满足谓词条件的 Stream。
* distinct: 返回一个没有重复元素的 Stream（根据 equals 的实现）
* limit(n): 返回一个不超过给定长度的 Stream
* skip(n): 返回一个忽略前n个的 Stream

查找和匹配: 一个通常的数据处理模式是判断一些元素是否满足给定的属性。可以使用 anyMatch, allMatch, 和 noneMatch 操作来帮助你实现。他们都需要一个 predicate 作为参数，并且返回一个 Boolean 作为作为结果（因此他们是终结操作）。比如，你可以使用 allMatch 来检车在 Stream 中的所有元素是否有一个值大于 100，像下面代码中表示的那样。

```
boolean expensive =
    transactions.stream()
                .allMatch(t -> t.getValue() > 100);
```

另外，Stream 提供了 findFirst 和 findAny，可以从 Stream 中获取任意元素。它们可以和 Stream 的其他操作连接在一起，比如 filter。findFirst 和 findAny 都返回一个 Optional 对象，像下面这样：

```
Optional<Transaction> = 
    transactions.stream()
                .filter(t -> t.getType() == Transaction.GROCERY)
                .findAny();
```

Optional<T> 类可以存放一个存在或者不存在的值。在下面代码中，findAny 可能没有返回一个交易类型是 grocery 类的信息。Optional 存在好多方法检测元素是否存在。比如，如果一个交易信息存在，我们可以使用相关函数处理 Optional 对象。

```
 transactions.stream()
              .filter(t -> t.getType() == Transaction.GROCERY)
              .findAny()
              .ifPresent(System.out::println);
```

映射：Stream 支持 map 方法，map 使用一个函数作为一个参数,你可以使用 map 从 Stream 的一个元素中提取信息。在下面的例子中，我们返回列表中每个单词的长度。

```
List<String> words = Arrays.asList("Oracle", "Java", "Magazine");
 List<Integer> wordLengths = 
    words.stream()
         .map(String::length)
         .collect(toList());
```

你可以定制更加复杂的查询，比如“交易中最大值的id”或者“计算交易金额总和”。这种处理需要使用 reduce 操作，reduce 可以将一个操作应用到每个元素上，知道输出结果。reduce 也经常被叫做折叠操作，因为你可以看到这种操作像把一个长的纸张（你的 Stream）不停地折叠直到想成一个小方格，这就是折叠操作。

看一下一个例子：

```
int sum = 0;
for (int x : numbers) {
    sum += x;
}
```

列表中的每个元素使用加号都迭代地进行了结合，从而产生了结果。我们本质上是“j减少”了集合中的数据，最终变成了一个数。上面的代码有两个参数：初始值和结合 list 中元素的操作符“+”

当使用 Stream 的 reduce 方法时，我们可以使用下面的代码将集合中的数字元素加起来。reduce 方法有两个参数：

```
int sum = numbers.stream().reduce(0, (a, b) -> a + b);
```

* 初始值,这里是0。
* 一个将连个数相加返回一个新值的 BinaryOperator<T>

reduce 方法本质上抽象了重复的模式。其他查询比如“计算产品”或者“计算最大值”是 reduce 方法的常规使用场景。

### 数值型 Stream

你已经看到了你可以使用 reduce 方法来计算一个 Integer 的 Stream了。然而，我们却执行了很多次的开箱操作去重复地把一个 Integer 对象添加到另一个上。如果我们调用 sum 方法岂不是很好？像下面代码那样，这样代码的意图也更加明确。

```
int statement = 
    transactions.stream()
                .map(Transaction::getValue)
                .sum(); // 这里是会报错的
```

在 Java 8 中引入了三种原始的特定数值型 Stream 接口来解决这个问题，它们是 IntStream, DoubleStream, 和 LongStream。它们各自可以数值型 Stream 变成一个 int、double、long。

可以使用 mapToInt, mapToDouble, and mapToLong 将通用 Stream 转化成一个数值型 Stream，我们可以将上面代码改成下面代码。当然你可以使用通用 Stream 类型取代数值型 Stream，然后使用开箱操作。

```
int statementSum =
    transactions.stream()
                .mapToInt(Transaction::getValue)
                .sum(); // 可以正确运行
```

数值类型 Stream 的另一个用途就是获取一个区间的数。比如你可能想要生成 1 到 100 之前的所有数。Java 8 在 IntStream, DoubleStream, 和 LongStream 中引入了两个静态方法来帮助生成一个区间，它们是range 和 rangeClosed.

这两个方法以区间开始的数为第一个参数，以区间结束的数为第二个参数。但是 range 的区间是开区间的，rangeClosed 是闭区间的。下面是一个使用 rangeClosed 返回 10 到 30 之间的奇数的 Stream。

```
IntStream oddNumbers =
    IntStream.rangeClosed(10, 30)
             .filter(n -> n % 2 == 1);
```

### 创建 Stream

有几种方式可以创建 Stream。你已经知道了可以从一个集合中获取一个 Stream，还你使用过数值类型 Stream。你可以使用数值、数组或者文件创建一个 Stream。另外，你甚至可以使用一个函数生成一个无穷尽的 Stream。

通过数值或者数组创建 Stream 可以很直接：对于数值是要使用静态方法Stream.of，对于数组使用静态方法 Arrays.stream ，像下面代码这样：

```
Stream<Integer> numbersFromValues = Stream.of(1, 2, 3, 4);
int[] numbers = {1, 2, 3, 4};
IntStream numbersFromArray = Arrays.stream(numbers);
```

你可以使用 Files.lines 静态方法将一个文件转化为一个 Stream。比如，下面代码计算一个文件的行数。

```
long numberOfLines =
    Files.lines(Paths.get(“yourFile.txt”), Charset.defaultCharset())
         .count();
```

### 无穷 Stream

到现在为止你知道了 Stream 元素是根据需求产生的。有两个静态方法Stream.iterate 和 Stream.generate 可以让你从从一个函数中创建一个 Stream，因为元素是根据需求计出来的，这两个方法可以一直产生元素。这也是我们叫无穷 Stream 的原因：Stream 没有一个固定的大小，但是它和从固定大小的集合中创建的 Stream 是一样的。

下面代码是一个使用 iterate 创建了包含一个 10 的倍数的 Stream。iterate 的第一个参数是初始值，第二个至是用于产生每个元素的 lambda 表达式（类型是 UnaryOperator<T>）。

```
Stream<Integer> numbers = Stream.iterate(0, n -> n + 10);
```

我们可以使用 limit 操作将一个无穷的 Stream 转化为一个大小固定的 Stream，像下面这样：

```
numbers.limit(5).forEach(System.out::println); // 0, 10, 20, 30, 40
```

### 总结

Java 8 引入了 Stream API，这可以让你实现复杂的数据查询处理。在这片文章中，我们已经看到了 Stream 支持很多操作，比如 filter、mpa，reduce 和 iterate，这些操作可以方便我们写简洁的代码和实现复杂的数据处理查询。这和 Java 8 之前使用的集合有很大的不同。Stream 有很多好处。首先，Stream API 使用了注入懒加载和短回路的技术优化了数据处理查询。第二，Stream 可以自动地并行运行，充分使用多核架构。在下一篇文章中，我们将探讨更多高级操作，比如 flatMap 和 collect ，请持续关注。