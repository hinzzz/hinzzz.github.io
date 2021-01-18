---
title: Lambda表达式详解
author: hinzzz
categories: 
	- Java1.8新特性
	- Lambda表达式
	- 函数式接口
date: 2020/06/10
keywords: [Lambda,Java1.8新特性,java双冒号,函数式接口]
description: Lambda表达式简介，Lambda表达式基本使用，Lambda表达式常用实例
---

### 为什么使用Lambda表达式

Lambda是一个**匿名函数**，我们可以理解Lambda表达式理解为一段可以传递的代码（将代码像数据一样传递）。可以写出更简洁，更灵活的代码。作为一种更紧凑的代码风格，使Java语言表达能力得到了提升



### 一、Lambda表达式详解

Lambda 表达式是 JDK8 的一个新特性，可以取代大部分的匿名内部类，写出更优雅的 Java 代码，尤其在集合的遍历和其他集合操作中，可以极大地优化代码结构。

JDK 也提供了大量的内置函数式接口供我们使用，使得 Lambda 表达式的运用更加方便、高效。

### 二、函数式接口@FunctionalInterface

> 所谓函数式接口，首先得是一个接口，并且在这个接口里面只能有一个抽象方法

主要用于编译级别错误检查，加上该注解，当你写的注解不符合规范的时候编译就会报错

```java
@FunctionalInterface
public interface HelloFunctionalInterface {
    void say();
    void say1();
}
Error:(8, 1) java: 意外的 @FunctionalInterface 注释
  com.hinz.functionalinterface.HelloFunctionalInterface 不是函数接口
    在 接口 com.hinz.functionalinterface.HelloFunctionalInterface 中找到多个非覆盖抽象方法
```



### 三、对接口的要求

虽然使用 Lambda 表达式可以对某些接口进行简单的实现，但并不是所有的接口都可以使用 Lambda 表达式来实现。**Lambda 规定接口中只能有一个需要被实现的方法，不是规定接口中只能有一个方法**

> jdk 8 中有另一个新特性：default， 被 default 修饰的方法会有默认实现，不是必须被实现的方法，所以不影响 Lambda 表达式的使用。

```java
//函数是接口中也使用泛型
@FunctionalInterface
public interface MyFun1<T> {
	T getValue(T t);
}

@FunctionalInterface
public interface MyFun {
	Integer getValue(Integer num);
}

```

### 四、基本语法

```java
//左侧指定了Lambda表达式所需要的的参数，右侧指定了Lambda体，即Lambda表达式执行的功能
(parameters) -> expression
(parameters) -> {statements}
```

- **可选类型声明：**不需要声明参数类型，编译器可以统一识别参数值。
- **可选的参数圆括号：**一个参数无需定义圆括号，但多个参数需要定义圆括号。
- **可选的大括号：**如果主体包含了一个语句，就不需要使用大括号。
- **可选的返回关键字：**如果主体只有一个表达式返回值则编译器会自动返回值，大括号需要指定明表达式返回了一个数值。

#### 1、常见Lambda表达式语法

```java
//格式一、无参无返回值，Lambda体只需一条语句
Runnable run1 = () -> {System.out.println("hinz");};
//格式二、无返回值，Lambda体{}可以省略
Runnable run2 = () -> System.out.println("hinz");
//格式三、单个参数,无返回值
Consumer<String> c1 = (String str) -> { System.out.println(str); };
//格式四、单个参数时参数的小括号可以省略,参数类型也可以省略
Consumer<String> c2 = str -> System.out.println(str); 
//格式五、多个参数，并且有返回值
BinaryOperator<Integer> b = (x,y) -> { return x+y ;};

```

#### 2、Lambda表达式作为参数传递

```java
public static Integer toAdd(MyFun myFun,Integer num){
        return myFun.getValue(num);
}
Integer num = LambdaDemo.toAdd(x -> x+100, 1);
System.out.println("num = " + num);
```

Lambda作为参数时：接受Lambda表达式的参数类型必须是与Lambda表达式兼容的函数式接口的类型。

### 五、Java内置四大核心函数式接口

**java.util.function包下**

| 函数式接口     | 参数类型 | 返回类型 | 用途                                                         |
| :------------- | :------: | :------: | ------------------------------------------------------------ |
| Consumer<T>    |    T     |   void   | 对类型为T的对象应用操作，包含方法void accept(T t);           |
| Supplier<T>    |    无    |    T     | 返回类型为T的对象,，包含方法T get();                         |
| Function<T, R> |    T     |    R     | 对类型为T的对象应用操作，返回类型为R的对象，包含方法R apply(T t); |
| Predicate<T>   |    T     | boolean  | 确定类型为T的对象是否满足约束，并返回boolean值，包含方法boolean test(T t); |

### 六、方法引用与构造器引用

当要传递给Lambda体的操作，已经有方法实现了，可以使用方法引用。（实现抽象方法的参数列表，必须与方法引用的参数列表保持一致）

使用操作符双冒号"::"，将方法名和对象或者类的名字分开。

#### 1、方法引用

三种主要适应情况：

- 对象::实例方法
- 类::静态方法
- 类::实例方法

```java
Consumer<String> c1 = (String s) -> {System.out.println(s);} ;
c1.accept("hello ::");

//使用双冒号
Consumer<String> c2 = System.out::println;
c2.accept("hi ::");
```



#### 2、构造器引用

**格式：ClassName::New**

```java
Function<Integer, Employee> f1 = (n) -> new Employee();
System.out.println("f1 = " + f1.apply(1));

//使用双冒号
Function<Integer, Employee> f2 = Employee::new;
System.out.println("f2 = " + f2.apply(2));
```

#### 3、数组引用

**格式：type[]::new**

```java
Function<Integer,Integer[]> f3 = n -> new Integer[n];
System.out.println("f3 = " + f3.apply(3));

//使用双冒号
Function<Integer,Integer[]> f4 = Integer[]::new;
System.out.println("f5 = " + f4.apply(4));
```



