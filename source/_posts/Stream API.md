---
title: 强大的Stream API
author: hinzzz
categories: 
	- Java1.8新特性
	- Stream API
	
date: 2020/06/14
keywords: [Stream API,Java1.8新特性]
description: Stream API详解
---

### 概述

> Java8中有两个最为重要的改变。第一个是Lambda表达式，另外一个则是Stream API（java.util.stream.*）Stream是Java8中处理集合的关键抽象概念，它可以指定你希望对集合进行的操作，可以执行非常复杂的查找、过滤和映射数据库等操作。使用Stream API对集合数据进行操作，就类似于使用SQL执行的数据库查询。也可以使用Stream API来并行执行操作。简而言之，Stream API提供了一种高效且易于使用的处理数据的方式。

### 一、什么是Stream

流（Stream）：数据渠道，用于操作数据源（集合，数组等）所生成的元素列。

**集合讲的是数据，流讲的是计算**

**注意**

- Stream自己不会存储元素
- Stream不会改变原对象。相反，他们会返回一个持有结果的新Stream。
- Stream操作是延迟执行的。意味着他们会等到需要结果的时候才执行。

### 二、Stream的操作三个步骤

1. 创建Stream

   一个数据源（如：集合，数组），获取一个流

2. 中间操作

   一个中间操作链，对数据源的数据进行处理

3. 终止操作（终端操作）

   ![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/stream.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=hNFhyQnkB%2BSzL%2FaoZQAQaY67cQ4%3D)



### 三、创建Stream

#### 1、创建Stream

Java8中的Collection接口被扩展，提供了两个获取流的方法：

```java
//返回一个串行流
default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }
//返回一个并行流
default Stream<E> parallelStream() {
        return StreamSupport.stream(spliterator(), true);
    }
```

由数组创建流

Java8中的Arrays的静态方法stream()获取Stream

```java
public static <T> Stream<T> stream(T[] array) {}
public static <T> Stream<T> stream(T[] array, int startInclusive, int endExclusive){}
public static IntStream stream(int[] array) {}
public static IntStream stream(int[] array, int startInclusive, int endExclusive) {}
public static LongStream stream(long[] array) {}
public static LongStream stream(long[] array, int startInclusive, int endExclusive) {}
public static DoubleStream stream(double[] array) {}
public static DoubleStream stream(double[] array, int startInclusive, int endExclusive){}
```

由值创建流

```java
public static<T> Stream<T> of(T... values) {
        return Arrays.stream(values);
    }
```

由函数创建流：创建无限流

```java
//迭代
public static<T> Stream<T> iterate(final T seed, final UnaryOperator<T> f) {}
//生成
public static<T> Stream<T> generate(Supplier<T> s) {}
```



#### 2、Stream的中间操作

> 多个**中间操作可以连接起来形成一个流水线**，除非流水线上触发终止操作，否则**中间操作不会执行任何的处理**！**而在终止操作时一次性全部处理**，称为”**惰性求职**“

筛选与切片

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Stream<T> filter(Predicate<? super T> predicate);            | 接受Lambda，从流中排除某些元素                               |
| Stream<T> distinct();                                        | 筛选，通过流所生成元素的hashCode()和equals()去除重复值       |
| Stream<T> limit(long maxSize);                               | 截断流，使其元素不超过给定数量                               |
| Stream<T> skip(long n);                                      | 跳过元素，返回一个扔掉了前n个元素的流。若流中元素不足n个，则返回一个空刘。与limit互补 |
| Stream<R> map(Function<? super T, ? extends R> mapper);      | 接收一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素 |
| IntStream mapToInt(ToIntFunction<? super T> mapper);         | 接收一个函数作为参数，该函数会被应用到每个元素上，产生的一个新的IntStream |
| LongStream mapToLong(ToLongFunction<? super T> mapper);      | 接收一个函数作为参数，该函数会被应用到每个元素上，产生的一个新的LongStream |
| DoubleStream mapToDouble(ToDoubleFunction<? super T> mapper); | 接收一个函数作为参数，该函数会被应用到每个元素上，产生的一个新的DoubleStream |
| Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper); | 接收一个函数作为参数，将流中的每个值都转换成另一个流，然后把所有的流连接成一个流 |
| Stream<T> sorted();                                          | 产生一个新流，其中按自然顺序排序                             |
| Stream<T> sorted(Comparator<? super T> comparator);          | 产生一个新流，其中按比较器顺序排序                           |

#### 3、Stream的终止操作

终端操作会从流的流水线生成结果。其结果可以是任何不是流的值。例如：List ,Integer,甚至是void

查找与匹配

| 方法                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| allMatch(Predicate p)  | 检查是否匹配所有元素                                         |
| anyMatch(Predicate p)  | 检查是否至少匹配一个元素                                     |
| noneMatch(Predicate p) | 检查是否没有匹配所有元素                                     |
| findFirst()            | 返回第一个元素                                               |
| findAny()              | 返回当前流中的任意元素                                       |
| count()                | 返回流中元素总数                                             |
| max(Comparator c)      | 返回流中最大值                                               |
| min(Comparator c)      | 返回流中最小值                                               |
| forEarch(Consumer c)   | 内部迭代（使用Collection接口需要用户去做迭代，称为外部迭代。相反，Stream API使用内部迭代） |

**收集**







