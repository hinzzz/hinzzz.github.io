---
title: Redis 集群版的安装
tags: Redis 集群
categories: Redis
date: 2019/12/27
keywords: [redis集群安装]
description: redis集群安装
---



#### 1.安装ruby

<!--more-->

```powershell
yum install ruby
yum install rubygems
#上传redis-3.0.0.gem gem 
install redis-3.0.0.gem
```

#### 2.规划集群配置

1. 创建6个redis实例，端口号从7001~7006

2. 修改redis的配置文件 修改端口号   /usr/local/reids/bin/redis.conf

3. cd /usr/loca/  mkdir redis-cluste

4. cp -r /usr/local/redis/bin /usr/local/redis-cluster/redis01~~~redis07

5. 把创建集群的ruby脚本复制到redis-cluster目录下

   ```powershell
   cp /redis/src/ redis-trib.rb /usr/local/redis-cluster
   ```

6. 启动6个redis实例1)vim startall.sh

   ```powershell
   #1、chmod u+x startall.sh
   #2、./startall.sh
   #3、脚本内容
   cd redis01
   ./redis-server redis.conf
   cd ..
   cd redis02
   ./redis-server redis.conf
   cd ..
   cd redis03
   ./redis-server redis.conf
   cd ..
   cd redis04
   ./redis-server redis.conf
   cd ..
   cd redis05
   ./redis-server redis.conf
   cd ..
   cd redis06
   ./redis-server redis.conf
   ```

7. 创建集群

   ```powershell
   ./redis-trib.rb create --replicas 1 192.168.141.125:7001 192.168.141.125:7002 192.168.141.125:7003 192.168.141.125:7004 192.168.141.125:7005  192.168.141.125:7006
   ```

8. 测试集群

   ```powershell
   cd redis01 ./redis-cli -c -h 192.168.141.125 -p 7001
   ```

9. 关闭redis

   ```powershell
   cd redis01. /redis.cli -p 7001 shutdown
   ```

   

