---
title: Mysql 主从复制
categories: Mysql
tags: Mysql
date: 2019/11/01
keywords: [mysql主从复制,mysql主从复制优点,mysql主从复制基本原则]
description: 本文主要介绍mysql主从复制的相关知识，涉及mysql主从复制的优缺点，以及mysql主从复制的基本原则
---



### Mysql 主从复制



#### 1、概述

> 主从复制 用来建立一个和主数据库一样的环境 做数据备份，成为从数据库，主数据库一般是实时数据，用来做增删改，从数据库只做查询

<!--more-->

#### 2、主从复制的优点（不仅仅适用于mysql）

1. 做数据备份，在主节点故障的情况下，能迅速切换到从节点，避免数据丢失
2. 减少单机的I/O频率，提高单个机器的I/O效率
3. 读写分离，减少慢查询sql导致的锁表，影响增删改的效率，保证主库的效率



#### 3、原理

![mysql主从原理](http://www.hinzzz.cn/notes/img/1.jpg)

1. mysql将所有的数据改变记录到二进制日志（binary log），这些记录过程叫做二进制日志时间，binary log events;
2. 从库slave将主库master的binary log拷贝到它的中继日志relay log
3. 从库slave重写中继日志的事件，将改变应用到自己的数据库中。**mysql的赋值是异步的且串行的。**



**具体步骤**

1. 主库insert , update , delete操作被写到binary log
3. 主库会创建一个线程，将binary log发送到从库
4. 从库启动后，创建一个I/O线程，将binary log的内容拷贝到relay log中
5. 从库还会创建一个sql线程，将relay log的内容重做，然后更新到从库中， MySQL复制是异步的且串行化的。



#### 4、主从复制基本原则

1. 每个slave只有一个master
2. 每个slave的服务器id必须唯一
3. 每个master可以有多个slave





