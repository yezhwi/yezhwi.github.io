---
layout:     post
title:      使用Java 8 Stream像操作SQL一样处理数据（下）
subtitle:   
date:       2018-11-29
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
---

> 原文出自：https://my.oschina.net/liuyatao19921025/blog/1609539

### 背景

在[上一篇文章](https://yezhwi.github.io/java/2018/11/28/%E4%BD%BF%E7%94%A8Java-8-Stream%E5%83%8F%E6%93%8D%E4%BD%9CSQL%E4%B8%80%E6%A0%B7%E5%A4%84%E7%90%86%E6%95%B0%E6%8D%AE-%E4%B8%8A/)中，我们介绍了 Stream 可以像操作数据库一样来操作集合，但是我们没有介绍 flatMap 和 collect 操作。这两种操作对实现复杂的查询是非常有用的。比如你可以结果 flatMap 和 collect 计算 Stream 中的单词的字符数，像下面代码那样。

```
import static java.util.function.Function.identity;
import static java.util.stream.Collectors.*;

Stream<String> words = Stream.of("Java", "Magazine", "is", "the", "best");

Map<String, Long> letterToCount =words.map(w -> w.split(""))
                .flatMap(Arrays::stream)
                .collect(groupingBy(identity(), counting()));
```

上述代码的运行结果是：

```
[a:4, b:1, e:3, g:1, h:1, i:2, ..]
```

这篇文章将会介绍 flatMap 和 collect 这两种操作的更多细节。

### flatMap 操作

假设你在一个文章中查找一个单词，你会怎么做？

我们可以使用 Files.lines() 方法，因为它可以返回一个文章一行一行信息组成的 Stream。我们可以使用 map() 把文章的每行分割是很多单词，最后，使用 distinct() 移除重复的。我们将想法转化为代码：

```
Files.lines(Paths.get("stuff.txt"))
              .map(line -> line.split("\\s+")) 
              .distinct() // Stream<String[]>
              .forEach(System.out::println);
```

很不幸，这样并不正确。如果你运行得到这样的结果：

```
[Ljava.lang.String;@7cca494b
[Ljava.lang.String;@7ba4f24f
…
```

到底发生了什么事呢？问题出在使用的 lambda 表达式将会把文件的每行转化成一个字符串数组(String[])。这就导致 map 返回的是一个 Stream<String[]> 类型的结果，我们实际上需要的是一个 Stream<String> 类型的结果。

我们需要一串的单词，而不是一串的数组。对于数组可以使用 Arrays.stream() 将数组变成一个 Stream。看下面的实现：

```
String[] arrayOfWords = {"Java", "Magazine"};
Stream<String> streamOfwords = Arrays.stream(arrayOfWords);
```

如果我们使用下面方式的话其实还有不起作用的，这是因为使用 map(Arrays::stream) 后返回的其实是 Stream<Stream<String>> 类型。

```
Files.lines(Paths.get("stuff.txt"))
            .map(line -> line.split("\\s+")) // Stream<String[]>
            .map(Arrays::stream) // Stream<Stream<String>>
            .distinct() // Stream<Stream<String>>
            .forEach(System.out::println);
```

我们可以使用 flatMap 来解决这种问题,像下面这样。使用 flatMap 方法的作用是返回的是 Stream 中的内容而不是一个 Stream。

```
Files.lines(Paths.get("stuff.txt"))
            .map(line -> line.split("\\s+")) // Stream<String[]>
            .flatMap(Arrays::stream) // Stream<String>
            .distinct() // Stream<String>
            .forEach(System.out::println);
```

### collect 操作

我们来具体看一下 collect 操作。上面文章中看到了返回 Stream 的操作（说明该操作是一个中间操作）和返回一个值、boolean 型值、int 型值和 Optional 型值的操作（说明该操作是终结操作） 。

### 将 Stream 中的元素转化到集合中

使用 toSet() 你可以把一个 Stream 转化成一个不包含重复项的集合。下面的代码展示了怎么生成高消费（单笔交易>1000$）城市的集合。

```
Set<String> cities = transactions.stream()
                   .filter(t -> t.getValue() > 1000)
                   .map(Transaction::getCity)
                   .collect(toSet());
```

注意这样你不能保证返回什么类型的 Set，你可以使用 toCollection()来提高可控性。比如你可以像下面代码这样将一个 HashSet 的构造方法作为参数。

```
Set<String> cities = transactions.stream()
                    .filter(t -> t.getValue() > 1000)
                    .map(Transaction::getCity)
                    .collect(toCollection(HashSet::new));
```

collect 操作方法不止这些，上面介绍的只是很小一部分，还可以实现这些功能：

* 通过货币类型进行分组，计算各种获取类型的交易总金额（将会返回一个  Map<Currency, Integer>）
* 将所有交易分类两组：大金额的和非大金额的（将会返回一个 Map<Boolean, List<Transaction>>）
* 创建多级分组，比如先根据城市分组，然后再根据是否为大金额交易分组（ 将会返回一个 Map<String, Map<Boolean, List<Transaction>>>）

让我们看一下 Stream API 和集合器怎么实现这些查询，我们先对一个 Stream 中的数据进行计算平均值，最大值和最小值。接下来我们再看如果实现简单的分组，最后我们我们将多个集合器放在一起实现强大的查询功能，比如多级分组。

### Summarizing

有很多预定义的集合器和是很方便的使用，比如使用 counting() 计算个数：

```
long howManyTransactions = transactions.stream().collect(counting());
```

你可以对 Double, Int, 或者 Long 属性的元素进行 summingDouble(), summingInt(), and summingLong() 操作，像下面这样：

```
int totalValue = transactions.stream().collect(summingInt(Transaction::getValue));
```

类似的你还可以使用 averagingDouble(), averagingInt(), and averagingLong() 计算平均值，像下面这样：

```
double average = transactions.stream().collect(averagingInt(Transaction::getValue));
```

还可以通过使用 maxBy()和minBy() 计算元素中的最大值和最小值，不过你需要定义一个做比较的 比较器，所以 maxBy 和 minBy 需要一个 Comparator 对象最为参数：

下面的例子中我们使用了静态方法 comparing()，它将根据传递进去的参数生成一个 Comparator 对象。这个方法根据提取 Stream 中元素的可以做比较的 key 来做判断。在这个例子中是通过银行交易的金额大小来做比较的。

```
Optional<Transaction> highestTransaction = transactions.stream()
                .collect(maxBy(comparing(Transaction::getValue)));
```

还有一个叫 reducing() 的集合器，它可以通过重复地对 Stream 中的所有元素进行一种操作指导产生一个结果。它和 reduce() 有点类似。比如下面的代码使用 reducing() 方法计算交易的总金额。

```
int totalValue = transactions.stream().collect(reducing(0, Transaction::getValue, Integer::sum));
```

reducing() 有三个参数：

* 初始值（如果 Stream 是空也将返回该值）：这里是 0
* 一个会被应用到各个元素的方法
* 结合两个提取出来的值，这是是将两个值加起来

### Grouping

一个常规的数据库操作就是根据一个属性对数据进行分组。比如根据货币对交易进行分组，如果使用迭代那简直太复杂了：

```
Map<Currency, List<Transaction>> transactionsByCurrencies = new HashMap<>();

for(Transaction transaction : transactions) { 
        Currency currency = transaction.getCurrency();
        List<Transaction> transactionsForCurrency = transactionsByCurrencies.get(currency);
        if (transactionsForCurrency == null) {
            transactionsForCurrency = new ArrayList<>();
            transactionsByCurrencies.put(currency, transactionsForCurrency);
        }
        transactionsForCurrency.add(transaction);
}
```

Java 8 中有一个叫 groupingBy() 的集合器，我们可以像这样做查询：

```
Map<Currency, List<Transaction>> transactionsByCurrencies =
    transactions.stream().collect(groupingBy(Transaction::getCurrency));
```

groupingBy() 方法有一个提取分类 key 的函数做参数，我们可以叫它为分类函数。在这个例子中我们使用的是 Transaction::getCurrency 来实现根据货币分组。

### Partitioning

还有一个叫做 partitioningBy() 的函数，这个可以看做是groupingBy() 的特例。它需要一个 predicate（返回一个 boolean 的函数）作为参数，将会对 Stream 中的元素根据是否满足 predicate 进行分类。partitioning 可以将 Stream 变成一个 Map<Boolean, List<Transaction>>。使用代码如下：

```
Map<Boolean, List<Transaction>> partitionedTransactions =transactions.stream().collect(partitioningBy( t -> t.getValue() > 1000));
```

如果要要对不同货币的金额进行求和操作在 SQL 中可以结合使用 SUM 和GROUP BY。那我们使用 Stream API 也能这么做吗？当然可以了，像下面这样使用：

```
Map<String, Integer> cityToSum = transactions.stream()
                                .collect(groupingBy(Transaction::getCity, summingInt(Transaction::getValue)));
```

之前使用的 groupingBy (Transaction::getCity)其实是groupingBy (Transaction::getCity, toList())的速写方式。

再看一个例子，如果你要统计每个城市的交易最大值，可以做这样实现：

```
Map<String, Optional<Transaction>> cityToHighestTransaction = 
           transactions.stream().collect(groupingBy(
             Transaction::getCity, maxBy(comparing(Transaction::getValue))));
```

再看一个更加复杂的例子，在刚才的例子中我们给 groupingBy 传递了另外一个集合器作为参数来进一步对元素进行分组。由于 groupingBy 本身是一个集合器，我们可以通过传递其他 groupingBy 集合器来创建多级分组，被传递进来的这个 groupingBy 定义了一个二级标准可以对 Stream 中的元素进行再分组。

下面代码中我们先对城市进行分组，然后我们再根据每个城市的交易不同货币的平均值进行分组

```
Map<String, Map<Currency, Double>> cityByCurrencyToAverage = 
        transactions.stream().collect(groupingBy(Transaction::getCity,groupingBy(Transaction::getCurrency, averagingInt(Transaction::getValue))));
```

### 自定义集合器

我们看到的这些集合器都实现了 java.util.stream.Collector 接口。这就意味着你可以自定义集合器。

### 总结

这篇文章中，我们探索了两个 Stream API 的高级操作：flatMap 和 colelct。通过这两个操作你可以创建更加复杂的数据处理查询。

我们还通过 collect 方法实现了 summarizing, grouping, 和 partitioning 操作。这些操作还可以被结合起来创建更加复杂的查询。