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

```java
List<Employee> emps = Arrays.asList(
			new Employee(102, "李四", 59, 6666.66, Employee.Status.BUSY),
			new Employee(101, "张三", 18, 9999.99, Employee.Status.FREE),
			new Employee(103, "王五", 28, 3333.33, Employee.Status.VOCATION),
			new Employee(104, "赵六", 8, 7777.77, Employee.Status.BUSY),
			new Employee(104, "赵六", 8, 7777.77, Employee.Status.FREE),
			new Employee(104, "赵六", 8, 7777.77, Employee.Status.FREE),
			new Employee(105, "田七", 38, 5555.55, Employee.Status.BUSY)
	);

@Test
	public void test1(){
			boolean bl = emps.stream()
				.allMatch((e) -> e.getStatus().equals(Employee.Status.BUSY));

			System.out.println(bl);

			boolean bl1 = emps.stream()
				.anyMatch((e) -> e.getStatus().equals(Employee.Status.BUSY));

			System.out.println(bl1);

			boolean bl2 = emps.stream()
				.noneMatch((e) -> e.getStatus().equals(Employee.Status.BUSY));

			System.out.println(bl2);
	}
@Test
	public void test2(){
		Optional<Employee> op = emps.stream()
			.sorted((e1, e2) -> Double.compare(e1.getSalary(), e2.getSalary()))
			.findFirst();

		System.out.println(op.get());

		System.out.println("--------------------------------");

		Optional<Employee> op2 = emps.parallelStream()
			.filter((e) -> e.getStatus().equals(Employee.Status.FREE))
			.findAny();

		System.out.println(op2.get());
	}

	@Test
	public void test3(){
		long count = emps.stream()
						 .filter((e) -> e.getStatus().equals(Employee.Status.FREE))
						 .count();

		System.out.println(count);

		Optional<Double> op = emps.stream()
			.map(Employee::getSalary)
			.max(Double::compare);

		System.out.println(op.get());

		Optional<Employee> op2 = emps.stream()
			.min((e1, e2) -> Double.compare(e1.getSalary(), e2.getSalary()));

		System.out.println(op2.get());
	}

	//注意：流进行了终止操作后，不能再次使用
	//java.lang.IllegalStateException: stream has already been operated upon or closed
	@Test
	public void test4(){
		Stream<Employee> stream = emps.stream()
		 .filter((e) -> e.getStatus().equals(Employee.Status.FREE));

		long count = stream.count();

		stream.map(Employee::getSalary)
			.max(Double::compare);
	}
```





**收集**

| 方法                 | 描述                                                         |
| -------------------- | ------------------------------------------------------------ |
| collect(Collector c) | 将流转换成其他形式。接收一个Collector接口的实现，用于给Stream中元素做汇总的方法 |

Collector接口中方法的实现决定了如何对流执行收集操作（如收集到List、Set、Map）。但是Collectors实用类提供了很多静态方法，可以方便的创建常见收集器实例，如下

```java
package com.hinz.streamapi;

import com.hinz.bean.Employee;
import org.junit.Test;

import java.util.*;
import java.util.stream.Collectors;

/**
 * @author ：quanhz
 * @date ：Created in 2021/1/19 10:43
 * @Description : No Description
 */
public class StreamApiTest {

    @Test
    public void collect(){
        List<Employee> emps = Arrays.asList(
                new Employee(1,"hinzzz",18,100),
                new Employee(1,"hinzzz",18,100),
                new Employee(1,"zs",19,200),
                new Employee(1,"l4",20,500));
        //把流中元素收集到List中
        List<Employee> collect = emps.stream().collect(Collectors.toList());
        System.out.println("collect = " + collect);

        //把流中元素收集到Set中
        Set<Employee> collect1 = emps.stream().collect(Collectors.toSet());
        System.out.println("collect1 = " + collect1);

        //把流中元素收集到指定集合中
        ArrayList<Employee> collect2 = emps.stream().collect(Collectors.toCollection(ArrayList::new));
        System.out.println("collect2 = " + collect2);

        //计算流中元素的个数
        Long collect3 = emps.stream().collect(Collectors.counting());
        System.out.println("collect3 = " + collect3);

        //计算流中数据的求和
        Integer collect4 = emps.stream().collect(Collectors.summingInt(Employee::getAge));
        System.out.println("collect4 = " + collect4);

        //计算流中数据的平均值
        Double collect5 = emps.stream().collect(Collectors.averagingDouble(Employee::getSalary));
        System.out.println("collect5 = " + collect5);

        //收集流中数据的统计值：总个数，最大、小值，平均值
        IntSummaryStatistics collect6 = emps.stream().collect(Collectors.summarizingInt(Employee::getAge));
        System.out.println("collect6 = " + collect6);

        //连接流中每个字符串
        String collect7 = emps.stream().map(Employee::getName).collect(Collectors.joining());
        System.out.println("collect7 = " + collect7);

        //根据比较器选择最大值
        Optional<Employee> collect8 = emps.stream().collect(Collectors.maxBy(Comparator.comparingInt(Employee::getAge)));
        System.out.println("collect8 = " + collect8.get());

        //根据比较器选择最小值
        Optional<Employee> collect9 = emps.stream().collect(Collectors.minBy(Comparator.comparingInt(Employee::getAge)));
        System.out.println("collect9 = " + collect9.get());

        //以某个值为开始累加，逐个执行BinaryOperator 得到最终结果
        Integer collect10 = emps.stream().collect(Collectors.reducing(0, Employee::getAge, Integer::max));
        System.out.println("collect10 = " + collect10);

        //包含另一个收集器，对其结果进行转换
        Integer collect11 = emps.stream().collect(Collectors.collectingAndThen(Collectors.toList(), List::size));
        System.out.println("collect11 = " + collect11);

        //根据某个属性值对流进行分组，属性为k，值为value
        Map<String, List<Employee>> collect12 = emps.stream().collect(Collectors.groupingBy(Employee::getName));
        System.out.println("collect12 = " + collect12);

        //根据true、false进行分区
        Map<Boolean, List<Employee>> collect13 = emps.stream().collect(Collectors.partitioningBy((emp) -> {
            return emp.getAge() == 20;
        }));
        System.out.println("collect13 = " + collect13);

        
    }
}


```



实例二、

```java
public class TestStreamAPI3 {

	List<Employee> emps = Arrays.asList(
			new Employee(102, "李四", 79, 6666.66, Employee.Status.BUSY),
			new Employee(101, "张三", 18, 9999.99, Employee.Status.FREE),
			new Employee(103, "王五", 28, 3333.33, Employee.Status.VOCATION),
			new Employee(104, "赵六", 8, 7777.77, Employee.Status.BUSY),
			new Employee(104, "赵六", 8, 7777.77, Employee.Status.FREE),
			new Employee(104, "赵六", 8, 7777.77, Employee.Status.FREE),
			new Employee(105, "田七", 38, 5555.55, Employee.Status.BUSY)
	);

	//3. 终止操作
	/*
		归约
		reduce(T identity, BinaryOperator) / reduce(BinaryOperator) ——可以将流中元素反复结合起来，得到一个值。
	 */
	@Test
	public void test1(){
		List<Integer> list = Arrays.asList(1,2,3,4,5,6,7,8,9,10);

		Integer sum = list.stream()
			.reduce(0, (x, y) -> x + y);

		System.out.println(sum);

		System.out.println("----------------------------------------");

		Optional<Double> op = emps.stream()
			.map(Employee::getSalary)
			.reduce(Double::sum);

		System.out.println(op.get());
	}

	//需求：搜索名字中 “六” 出现的次数
	@Test
	public void test2(){
		Optional<Integer> sum = emps.stream()
			.map(Employee::getName)
			.flatMap(TestStreamAPI1::filterCharacter)
			.map((ch) -> {
				System.out.println("ch = " + ch);
				if(ch.equals('六'))
					return 1;
				else
					return 0;
			}).reduce(Integer::sum);

		System.out.println(sum.get());


	}

	//collect——将流转换为其他形式。接收一个 Collector接口的实现，用于给Stream中元素做汇总的方法
	@Test
	public void test3(){
		List<String> list = emps.stream()
			.map(Employee::getName)
			.collect(Collectors.toList());

		list.forEach(System.out::println);

		System.out.println("----------------------------------");

		Set<String> set = emps.stream()
			.map(Employee::getName)
			.collect(Collectors.toSet());

		set.forEach(System.out::println);

		System.out.println("----------------------------------");

		HashSet<String> hs = emps.stream()
			.map(Employee::getName)
			.collect(Collectors.toCollection(HashSet::new));

		hs.forEach(System.out::println);
	}

	@Test
	public void test4(){
		Optional<Double> max = emps.stream()
			.map(Employee::getSalary)
			.collect(Collectors.maxBy(Double::compare));

		System.out.println(max.get());

		Optional<Employee> op = emps.stream()
			.collect(Collectors.minBy((e1, e2) -> Double.compare(e1.getSalary(), e2.getSalary())));

		System.out.println(op.get());

		Double sum = emps.stream()
			.collect(Collectors.summingDouble(Employee::getSalary));

		System.out.println(sum);

		Double avg = emps.stream()
			.collect(Collectors.averagingDouble(Employee::getSalary));

		System.out.println(avg);

		Long count = emps.stream()
			.collect(Collectors.counting());

		System.out.println(count);

		System.out.println("--------------------------------------------");

		DoubleSummaryStatistics dss = emps.stream()
			.collect(Collectors.summarizingDouble(Employee::getSalary));

		System.out.println(dss.getMax());
	}

	//分组
	@Test
	public void test5(){
		Map<Employee.Status, List<Employee>> map = emps.stream()
			.collect(Collectors.groupingBy(Employee::getStatus));

		System.out.println(map);
	}

	//多级分组
	@Test
	public void test6(){
		Map<Employee.Status, Map<String, List<Employee>>> map = emps.stream()
			.collect(Collectors.groupingBy(Employee::getStatus, Collectors.groupingBy((e) -> {
				if(e.getAge() >= 60)
					return "老年";
				else if(e.getAge() >= 35)
					return "中年";
				else
					return "成年";
			})));

		System.out.println(map);
	}

	//分区
	@Test
	public void test7(){
		Map<Boolean, List<Employee>> map = emps.stream()
			.collect(Collectors.partitioningBy((e) -> e.getSalary() >= 5000));

		System.out.println(map);
	}

	//
	@Test
	public void test8(){
		String str = emps.stream()
			.map(Employee::getName)
			.collect(Collectors.joining("," , "----", "----"));

		System.out.println(str);
	}

	@Test
	public void test9(){
		Optional<Double> sum = emps.stream()
			.map(Employee::getSalary)
			.collect(Collectors.reducing(Double::sum));

		System.out.println(sum.get());
	}
}

```



