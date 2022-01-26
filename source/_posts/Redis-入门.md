---
title: redis入门教程
author: hinzzz
categories: redis
date: 2021/10/25
keywords: [redis概述,redis数据类型,redis常用配置]
description: 本文主要介绍redis相关入门教程
---





#### 一、概念

>   	Remote Dictionary Server(远程字典服务器)是完全开源免费的，用C语言编写的，遵守BSD协议，是一个高性能的(key-value)分布式内存数据库，基于内存运行并支持持久化的NoSQL数据库，是当前最热门的NoSql数据库之一,也被人们称为数据结构服务器。



#### 二、特点

Redis与其他key-value缓存产品有以下三个特点

1. Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用
2. Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储
3. Redis支持数据的备份，即master-slave模式的数据备份



#### 三、能做什么

1. 内存存储和持久化：redis支持异步将内存中的数据写到硬盘上，同时不影响继续服务
2. 取最新N个数据的操作，如：可以将最新的10条评论的ID放在Redis的List集合里面
3. 模拟类似于HttpSession这种需要设定过期时间的功能
4. 发布、订阅消息系统
5. 定时器、计数器



#### 四、文件说明

1. redis-benchmark:性能测试工具，可以在自己本子运行，看看自己本子性能如何
2. redis-check-aof：修复有问题的AOF文件
3. redis-check-dump：修复有问题的dump.rdb文件
4. redis-cli：客户端，操作入口
5. redis-sentinel：redis集群使用
6. redis-server：Redis服务器启动命令



#### 五、其他知识

1. 单进程：单进程模型来处理客户端的请求。对读写等事件的响应是通过对epoll函数的包装来做到的。Redis的实际处理速度完全依靠主进程的执行效率，epoll是Linux内核为处理大批量文件描述符而作了改进的epoll，是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。
2. 默认16个数据库，类似数组下表从零开始，初始默认使用零号库
3. 设置数据库的数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id databases 16
4. select命令切换数据库
5. dbsize查看当前数据库的key的数量
6. flushdb：清空当前库
7. flushall；通杀全部库
8. Redis索引都是从零开始
9. 统一密码管理，16个库都是同样密码，要么都OK要么一个也连接不上



#### 六、数据类型

1. String（字符串）string是redis最基本的类型，你可以理解成与Memcached一模一样的类型，一个key对应一个value。string类型是二进制安全的。意思是redis的string可以包含任何数据。比如jpg图片或者序列化的对象 。string类型是Redis最基本的数据类型，一个redis中字符串value最多可以是512M
2. Hash（哈希）Redis hash 是一个键值对集合。Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。类似Java里面的Map<String,Object>
3. List（列表）Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素导列表的头部（左边）或者尾部（右边）。它的底层实际是个链表
4. Set（集合）Redis的Set是string类型的无序集合。它是通过HashTable实现实现的
5. zset(sorted set：有序集合)Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。zset的成员是唯一的,但分数(score)却可以重复。



#### 七、

##### 1、通用

1、单位units

+ 配置大小单位,开头定义了一些基本的度量单位，只支持bytes，不支持bit
+ 对大小写不敏感

2、include

+ 可以通过includes包含，redis.conf可以作为总闸，包含其他

3、daemonize

+ yes为后台启动 默认为no

4、pidfile

+ 当运行daemalized时，Redis在/var/run/ redisdis中编写一个pid文件
+ pidfile /var/run/redis.pid

5、port

+ 启动端口 默认port 6379

6、tcp-backlog

+ 设置tcp的backlog，backlog其实是一个连接队列，backlog队列总和=未完成三次握手队列 + 已经完成三次握手队列。
+ 在高并发环境下你需要一个高backlog值来避免慢客户端连接问题。注意Linux内核会将这个值减小到/proc/sys/net/core/somaxconn的值，所以需要确认增大somaxconn和tcp_max_syn_backlog两个值来达到想要的效果

7、timeout

+ 客户端空闲N秒后关闭连接(禁用0秒)，默认0
+ 默认timeout 0

8、bind

+ 8、bind默认情况下，Redis监听所有网络接口的连接在服务器上可用。只听一个或多个是可能的使用“bind”配置指令的接口，后面跟着一个or更多IP地址。bind 	192.168.1.100 	10.0.0.1
+ bind     127.0.0.1

9、tcp-keepalive

+ 单位为秒，如果设置为0，则不会进行Keepalive检测，建议设置成60
+  默认tcp-keepalive 0

10、loglevel

+ 日志级别
+ debug(大量信息，对开发/测试有用)
+ verbose(许多很少有用的信息，但不像调试级别那样混乱)
+ notice(适度详细，您可能想在生产中使用)
+ warn(只记录非常重要/关键的消息)
+ 默认loglevel notice

11、logfile

+ 日志文件的存放路径指定日志文件名。
+ 空字符串也可以用来强制Redis要登录标准输出。注意，如果你使用标准日志将被发送到/dev/null
+ 默认logfile ""

12、syslog-enabled

+ 是否把日志输出到syslog中
+ 默认不配置# syslog-enabled no

13、syslog-ident

+ 指定syslog的日志标志
+ 默认不配置 # syslog-ident redis

14、syslog-facility

+ 指定syslog设备，必须是USER或介于LOCAL0-LOCAL7之间。
+ 默认syslog-facilityLOCAL0

15、database

+ 设置数据库的数量。默认数据库是DB 0，可以选择在每个连接的基础上使用SELECT <dbid> where进行不同的连接# dbid是0和'database '-1之间的数字
+ 默认database 16



##### 2、快照

1、save <seconds> <changes>

+ 如果给定的秒数和给定的秒数，将保存数据库对数据库执行的写操作数量。s
+ ave 900 1
+ save 300 10
+ save 60 10000
+ RDB是整个内存的压缩过的Snapshot，RDB的数据结构，可以配置复合的快照触发条件。
+ 禁用：save ""

2、stop-writes-on-bgsave-error yes

+ 如果配置成no，表示你不在乎数据不一致或者有其他的手段发现和控制

3、rdbcompression yes 

+ rdbcompression：对于存储到磁盘中的快照，可以设置是否进行压缩存储。如果是的话，redis会采用LZF算法进行压缩。如果你不想消耗CPU来进行压缩的话，可以设置为关闭此功能

4、rdbchecksum yes 

+ rdbchecksum：在存储快照后，还可以让redis使用CRC64算法来进行数据校验，但是这样做会增加大约10%的性能消耗，如果希望获取到最大的性能提升，可以关闭此功能

5、dbfilename

+ dump.rdbrdb文件名

6、dir ./

+ rdb文件的存放目录

##### 3、安全

1、masterauth <master-password>

+ 当master服务设置了密码保护时，slav服务连接master的密码



##### 4、限制

1、maxclients 10000

+ 设置redis同时可以与多少个客户端进行连接。**默认情况下为10000个客户端**。当你无法设置进程文件句柄限制时，redis会设置为当前的文件句柄限制值减去32，因为redis会为自身内部处理逻辑留一些句柄出来。如果达到了此限制，redis则会拒绝新的连接请求，并且向这些连接请求方发出“max number of clients reached”以作回应。

2、maxmemory <bytes>

+ 设置redis可以使用的内存量。一旦到达内存使用上限，redis将会试图移除内部数据，移除规则可以通过maxmemory-policy来指定。如果redis无法根据移除规则来移除内存中的数据，或者设置了“不允许移除”，那么redis则会针对那些需要申请内存的指令返回错误信息，比如SET、LPUSH等。但是对于无内存申请的指令，仍然会正常响应，比如GET等。如果你的redis是主redis（说明你的redis有从redis），那么在设置内存使用上限时，需要在系统中留出一些内存空间给同步队列缓存，只有在你设置的是“不移除”的情况下，才不用考虑这个因素

3、maxmemory-policy

+ volatile-lru：使用LRU算法移除key，只对设置了过期时间的键
+ allkeys-lru：使用LRU算法移除
+ keyvolatile-random：在过期集合中移除随机的key，只对设置了过期时间的键
+ allkeys-random：移除随机的key
+ volatile-ttl：移除那些TTL值最小的key，即那些最近要过期的key
+ noeviction：不进行移除。针对写操作，只是返回错误信息



4、maxmemory-samples

+ 设置样本数量，LRU算法和最小TTL算法都并非是精确的算法，而是估算值，所以你可以设置样本的大小，redis默认会检查这么多个key并选择其中LRU的那个

##### append only mode

1、appendonly no

+ 开启appendonly模式（yes）

2、 appendfilename "appendonly.aof"

+ 指定文件名称

3、appendsync

+ always同步持久化 每次发生数据变更会被立即记录到磁盘 性能较差但数据完整性比较好
+ everysec出厂默认推荐，异步操作，每秒记录  如果一秒内宕机，有数据丢失
+ no

4、no-appendfsync-on-rewrite

+ 重写时是否可以运用Appendfsync，用默认no即可，保证数据安全性。

5、auto-aof-rewrite-min-size 64mb

+ 设置重写的大小

6、auto-aof-rewrite-percentage 100

+ 设置重写的百分比，如100则为等于128mb就开始重写

#### 八、redis.conf

```shell

1. Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程  daemonize no
2. 当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid文件，可以通过pidfile指定  pidfile /var/run/redis.pid
3. 指定Redis监听端口，默认端口为6379，作者在自己的一篇博文中解释了为什么选用6379作为默认端口，因为6379在手机按键上MERZ对应的号码，而MERZ取自意大利歌女Alessia Merz的名字  port 6379
4. 绑定的主机地址  bind 127.0.0.1
5.当 客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能  timeout 300
6. 指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，默认为verbose  loglevel verbose
7. 日志记录方式，默认为标准输出，如果配置Redis为守护进程方式运行，而这里又配置为日志记录方式为标准输出，则日志将会发送给/dev/null  logfile stdout
8. 设置数据库的数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id  databases 16
9. 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合  save <seconds> <changes>  Redis默认配置文件中提供了三个条件：  save 900 1  save 300 10  save 60 10000  分别表示900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改。 
10. 指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，可以关闭该选项，但会导致数据库文件变的巨大  rdbcompression yes
11. 指定本地数据库文件名，默认值为dump.rdb  dbfilename dump.rdb
12. 指定本地数据库存放目录  dir ./
13. 设置当本机为slav服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步  slaveof <masterip> <masterport>
14. 当master服务设置了密码保护时，slav服务连接master的密码  masterauth <master-password>
15. 设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过AUTH <password>命令提供密码，默认关闭  requirepass foobared
16. 设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息  maxclients 128
17. 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，Value会存放在swap区  maxmemory <bytes>
18. 指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为 redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no  appendonly no
19. 指定更新日志文件名，默认为appendonly.aof   appendfilename appendonly.aof
20. 指定更新日志条件，共有3个可选值：   no：表示等操作系统进行数据缓存同步到磁盘（快）   always：表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全）   everysec：表示每秒同步一次（折衷，默认值）  appendfsync everysec 
21. 指定是否启用虚拟内存机制，默认值为no，简单的介绍一下，VM机制将数据分页存放，由Redis将访问量较少的页即冷数据swap到磁盘上，访问多的页面由磁盘自动换出到内存中（在后面的文章我会仔细分析Redis的VM机制）   vm-enabled no
22. 虚拟内存文件路径，默认值为/tmp/redis.swap，不可多个Redis实例共享   vm-swap-file /tmp/redis.swap
23. 将所有大于vm-max-memory的数据存入虚拟内存,无论vm-max-memory设置多小,所有索引数据都是内存存储的(Redis的索引数据 就是keys),也就是说,当vm-max-memory设置为0的时候,其实是所有value都存在于磁盘。默认值为0   vm-max-memory 0
24. Redis swap文件分成了很多的page，一个对象可以保存在多个page上面，但一个page上不能被多个对象共享，vm-page-size是要根据存储的 数据大小来设定的，作者建议如果存储很多小对象，page大小最好设置为32或者64bytes；如果存储很大大对象，则可以使用更大的page，如果不 确定，就使用默认值   vm-page-size 32
25. 设置swap文件中的page数量，由于页表（一种表示页面空闲或使用的bitmap）是在放在内存中的，，在磁盘上每8个pages将消耗1byte的内存。   vm-pages 134217728
26. 设置访问swap文件的线程数,最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟。默认值为4   vm-max-threads 4
27. 设置在向客户端应答时，是否把较小的包合并为一个包发送，默认为开启  glueoutputbuf yes
28. 指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法  hash-max-zipmap-entries 64  hash-max-zipmap-value 512
29. 指定是否激活重置哈希，默认为开启（后面在介绍Redis的哈希算法时具体介绍）  activerehashing yes
30. 指定包含其它的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各个实例又拥有自己的特定配置文件  include /path/to/local.conf

```

