---
title: Optional入门教程
author: hinzzz
categories: 
	- Java1.8新特性
	- Optional
	
date: 2020/06/14
keywords: [Optional,Java1.8新特性]
description: java1.8 Optional

---

```java
package com.hinz.optional;

import java.util.Optional;

import com.hinz.bean.Employee;
import com.hinz.bean.Godness;
import com.hinz.bean.Man;
import com.hinz.bean.NewMan;
import org.junit.Test;

/*
 * 一、Optional 容器类：用于尽量避免空指针异常
 * 	Optional.of(T t) : 创建一个 Optional 实例
 * 	Optional.empty() : 创建一个空的 Optional 实例
 * 	Optional.ofNullable(T t):若 t 不为 null,创建 Optional 实例,否则创建空实例
 * 	isPresent() : 判断是否包含值
 * 	orElse(T t) :  如果调用对象包含值，返回该值，否则返回t
 * 	orElseGet(Supplier s) :如果调用对象包含值，返回该值，否则返回 s 获取的值
 * 	map(Function f): 如果有值对其处理，并返回处理后的Optional，否则返回 Optional.empty()
 * 	flatMap(Function mapper):与 map 类似，要求返回值必须是Optional
 */
public class TestOptional {

	@Test
	public void test4(){
		Optional<Employee> op = Optional.of(new Employee(101, "张三", 18, 9999.99,null));
		System.out.println(op.get());

		Optional<String> op2 = op.map(Employee::getName);
		System.out.println(op2.get());

		Optional<String> op3 = op.flatMap((e) -> Optional.of(e.getName()));
		System.out.println(op3.get());
	}

	@Test
	public void test3(){
		Optional<Employee> op = Optional.ofNullable(new Employee());

		if(op.isPresent()){
			System.out.println(op.get());
		}

		Employee emp = op.orElse(new Employee("张三"));
		System.out.println(emp);

		Employee emp2 = op.orElseGet(() -> new Employee());
		System.out.println(emp2);
	}

	@Test
	public void test2(){
		Optional<Employee> op = Optional.ofNullable(null);
		System.out.println(op.get());

//		Optional<Employee> op = Optional.empty();
//		System.out.println(op.get());
	}

	@Test
	public void test1(){
		Optional<Employee> op = Optional.of(new Employee());
		Employee emp = op.get();
		System.out.println(emp);
	}

	@Test
	public void test5(){
		Man man = new Man();

		String name = getGodnessName(man);
		System.out.println(name);
	}

	//需求：获取一个名字
	public String getGodnessName(Man man){
		if(man != null){
			Godness g = man.getGod();

			if(g != null){
				return g.getName();
			}
		}

		return "hinzzz";
	}

	//运用 Optional 的实体类
	@Test
	public void test6(){
		Optional<Godness> godness = Optional.ofNullable(new Godness("林志玲"));

		Optional<NewMan> op = Optional.ofNullable(new NewMan(godness));
		String name = getGodnessName2(op);
		System.out.println(name);
	}

	public String getGodnessName2(Optional<NewMan> man){
		return man.orElse(new NewMan())
				  .getGodness()
				  .orElse(new Godness("hinzzz"))
				  .getName();
	}
}

```