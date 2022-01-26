---
title: redis 单机版的安装
date: 2019/12/27
tags: redis
categories: redis
keywords: [redis单机版安装]
description: redis单机版安装
---



1、把源码包上传到linux服务器

<!--more-->

2、解压源码包  tar -zxvf redis-3.0.0.tar.gz

3、cd redis-3.0.0

4、make

5、make install PREFIX=/usr/local/redis 

6、将redis.conf从 redis的源码包中复制到、usr/local/redis/bin 下

7、前端启动 ./redis-server redis.conf

8、修改redis.conf配置文件 daemonize yes

9、后端启动 ./redis-cli (要先启动服务端才能启动客户端)

10、ps –aux | grep redis

11、关闭 ./redis-cli -p 6379 shutdown

12、默认会拦截6379端口   需要放行 

```shell
/sbin/iptables -I INPUT -p tcp --dport 6379 -j ACCEPT/etc/rc.d/init.d/iptables save
```

