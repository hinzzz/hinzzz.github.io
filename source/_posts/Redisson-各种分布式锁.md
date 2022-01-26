---
title: Redisson入门教程
author: hinzzz
categories: redis
date: 2021/04/30
keywords: [Redisson入门教程,Redisson各种分布式锁,Redisson看门狗机制,Redisson读写锁]
description: 本文主要介绍Redisson入门教程
---



### 各种分布式锁的各种实现

#### 1、可靠性

为了确保分布式锁可用，我们至少要确保锁的条件同时满足一下四个条件

1. 互斥性，在任意时刻，只有一个客户端能够持有锁
2. 不会发生死锁，即时有一个客户端因为宕机没有释放锁，也不会影响别的客户端持有锁
3. 容错性，只有要大部分实例正常运行，客户端就可以加解锁
4. 解铃还须系铃人，保证加锁和解锁都在同一个客户端，客户端不能把别人的锁给解了，造成锁丢失

#### 一、普通实现

常用的方法：set-nx + lua 或者set key value px milliseconds nx

```shell
# 获取锁（unique_value可以是UUID等）
SET resource_name unique_value NX PX 30000

# 释放锁（lua脚本中，一定要比较value，防止误解锁）
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

##### 1、实现要点

1. set命令要用`set key value px milliseconds nx`
2. value要具有唯一性
3. 释放锁时要验证value值，不能误解锁

##### 2、缺点

加锁时只作用在一个Redis节点上，如果这个节点由于某些原因出现了主从切换，那么就会出现锁丢失的情况

1. 在Redis的节点上拿到了锁
2. 但是加锁的key还没有同步到slave
3. master发生故障，故障转移，slave节点升级为master节点
4. 导致锁丢失

##### 3、用法

```java
// 构造redisson实现分布式锁必要的Config
Config config = new Config();
config.useClusterServers()
                .setScanInterval(2000) // 集群状态扫描间隔时间，单位是毫秒 默认值： 5000
                .setCheckSlotsCoverage(false)
                .addNodeAddress("redis://ip:6379")//节点地址
                .addNodeAddress("redis://ip:6380")//节点地址
                .addNodeAddress("redis://ip:6381")//节点地址
// 构造RedissonClient
RedissonClient redissonClient = Redisson.create(config);
// 设置锁定资源名称
RLock disLock = redissonClient.getLock("DISLOCK");
boolean isLock;
try {
    //尝试获取分布式锁
    isLock = disLock.tryLock(500, 15000, TimeUnit.MILLISECONDS);
    if (isLock) {
        //TODO 业务
        Thread.sleep(15000);
    }
} catch (Exception e) {
} finally {
    // 无论如何, 最后都要解锁
    disLock.unlock();
}
```



##### 4、为什么不在finally里面加个判断再解锁？

```java
 if(lock.isLocked()){
     lock.unlock();
 }
```

防止Redis实际上已经加锁成功但是还没有响应给客户端



#### 二、看门狗机制

> 默认加锁的情况下，会在Redis实例中key有个默认30秒ttl，并且每过10秒就会重置为30秒，直到业务结束释放锁。

![image-20210906114525214](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210906114525214.png)

```java
@RequestMapping("/nomalLock/{orderNo}")
@ResponseBody
public String nomalLock(@PathVariable String orderNo)  {
    RLock lock = redissonClient.getLock(orderNo);//只要锁的名字一样就是同一把锁
    lock.lock();//阻塞式等待，释放锁时间为30秒
    			//如果一个线程在获取锁成功之后因为某些原因导致了锁没有被释放，会不会产生死锁问题？不会，因为加锁的时候默认锁的释放时间是30秒
    //lock.lock(10,TimeUnit.SECONDS);//锁的释放时间一定要大于业务的执行时间 只要释放锁的时间不为-1 就不会开启看门狗机制
    try {
        TimeUnit.SECONDS.sleep(40);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }finally {
        lock.unlock();
    }
    return "ok";
}
```



##### 1、源码

```java
@Override
public void lock() {
    try {
        lock(-1, null, false);
    } catch (InterruptedException e) {
        throw new IllegalStateException();
    }
}
```

```java
private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
        long threadId = Thread.currentThread().getId();
        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);//获取锁
        // lock acquired
        if (ttl == null) {
            return;
        }
        RFuture<RedissonLockEntry> future = subscribe(threadId);
        if (interruptibly) {
            commandExecutor.syncSubscriptionInterrupted(future);
        } else {
            commandExecutor.syncSubscription(future);
        }
        try {
            while (true) {//获取不到锁 继续循环直到获取到为止
                ttl = tryAcquire(-1, leaseTime, unit, threadId);
                // lock acquired
                if (ttl == null) {
                    break;
                }

                // waiting for message
                if (ttl >= 0) {
                    try {
                        future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    } catch (InterruptedException e) {
                        if (interruptibly) {
                            throw e;
                        }
                        future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    }
                } else {
                    if (interruptibly) {
                        future.getNow().getLatch().acquire();
                    } else {
                        future.getNow().getLatch().acquireUninterruptibly();
                    }
                }
            }
        } finally {
            unsubscribe(future, threadId);
        }
    }
```

```java
private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        return get(tryAcquireAsync(waitTime, leaseTime, unit, threadId));
    }
```

```java
//从这个方法可以发现tryLock()和lock()底层调用方法都是一样的 知不是根据leaseTime做了不同的处理 继续往下走
private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        if (leaseTime != -1) {//tryLock调用
            return tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        }
    	//lock继续往下走  默认加锁时间30秒：commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout()解决了死锁问题
        RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(waitTime,
                                                commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(),
                                                TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
        //只要占锁成功，就会启动一个定时任务重新设置锁的过期时间
        ttlRemainingFuture.onComplete((ttlRemaining, e) -> {//类似于监听的回调
            if (e != null) {
                return;
            }

            // lock acquired
            if (ttlRemaining == null) {
                scheduleExpirationRenewal(threadId);
            }
        });
        return ttlRemainingFuture;
    }
```

```java
private void scheduleExpirationRenewal(long threadId) {
        ExpirationEntry entry = new ExpirationEntry();
        ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
        if (oldEntry != null) {
            oldEntry.addThreadId(threadId);
        } else {
            entry.addThreadId(threadId);
            renewExpiration();//重新设置锁的过期时间
        }
    }
```

```java
private void renewExpiration() {
        ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (ee == null) {
            return;
        }
    	//定时任务
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
                if (ent == null) {
                    return;
                }
                Long threadId = ent.getFirstThreadId();
                if (threadId == null) {
                    return;
                }
                RFuture<Boolean> future = renewExpirationAsync(threadId);
                future.onComplete((res, e) -> {
                    if (e != null) {
                        log.error("Can't update lock " + getName() + " expiration", e);
                        return;
                    }
                    
                    if (res) {
                        // reschedule itself
                        renewExpiration();
                    }
                });
            }
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
        ee.setTimeout(task);
    }
```

```java
 this.internalLockLeaseTime = commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout();//看门后时间默认为30秒，所以上面的task每隔10秒执行一次 刷新锁的过期时间
```



##### 2、使用推荐

指定锁的过期时间，省略了看门狗机制，只要将锁的过期时间设置好就行

```java
lock.lock(10,TimeUnit.SECONDS);
boolean tryLock = lock.tryLock(6, 11, TimeUnit.SECONDS);
```



#### 三、读写锁

> 基于Redis的Redisson分布式可重入读写锁[`RReadWriteLock`](http://static.javadoc.io/org.redisson/redisson/3.4.3/org/redisson/api/RReadWriteLock.html) Java对象实现了`java.util.concurrent.locks.ReadWriteLock`接口。其中读锁和写锁都继承了[RLock](https://github.com/redisson/redisson/wiki/8.-分布式锁和同步器#81-可重入锁reentrant-lock)接口。
>
> 分布式可重入读写锁允许同时有多个读锁和一个写锁处于加锁状态。



```java
@RequestMapping("/write/{orderNo}")
@ResponseBody
public String write(@PathVariable String orderNo) {
    RReadWriteLock rReadWriteLock = redissonClient.getReadWriteLock("rw-lock");
    RLock rLock = rReadWriteLock.writeLock();
    rLock.lock();
    try {
        stringRedisTemplate.opsForValue().set("write", orderNo);
        TimeUnit.SECONDS.sleep(30);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }finally {
        rLock.unlock();
    }
    return orderNo;
}


@RequestMapping("/read/{orderNo}")
@ResponseBody
public String read(@PathVariable String orderNo) {
    RReadWriteLock rReadWriteLock = redissonClient.getReadWriteLock("rw-lock");
    RLock rLock = rReadWriteLock.readLock();
    rLock.lock();
    try {
        TimeUnit.SECONDS.sleep(10);
        return stringRedisTemplate.opsForValue().get("write");
    } catch (Exception e) {
        e.printStackTrace();
    }finally {
        rLock.unlock();
    }
    return "xx";
}
```

##### 1、原理

1. 保证一定能读到最新数据，修改期间，写锁是一个排他锁，读锁是一个共享锁 ，写锁没释放读锁就必须等待
   1. 读 + 读 ：相当于无锁，并发读，只会在redis中记录好所有当前的锁，他们都会同时加锁成功
   2. 写 + 读 ：读锁必须等写锁释放
   3. 写 + 写 ：阻塞性等待
   4. 读 + 写 ： 有读锁，写锁也需要等待





#### 四、红锁

> Redis作者antirez提出的redlock算法大概是这样的：
>
> 在Redis的分布式环境中，我们假设有N个Redis master。这些节点**完全互相独立，不存在主从复制或者其他集群协调机制**。我们确保将在N个实例上使用与在Redis单实例下相同方法获取和释放锁。**现在我们假设有5个Redis master节点**，同时我们需要在5台服务器上面运行这些Redis实例，这样保证他们不会同时都宕掉。

##### 1、步骤

为了获取锁，客户端应该进行下面操作：

1. 获取当前UNIX时间，毫秒为单位
2. 尝试从五个实例，使用相同的key和具有唯一性的value(例如UUID)获取锁。当向Redis请求获取锁时，客户端应该设置一个网络连接和相应超时时间，这个超时时间应该小于锁的失效时间。例如你的失效时间为10秒，那么超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端仍在苦苦等待。如果服务端在时间内没有响应，客户端会尽快尝试去另一个服务端请求锁。
3. 客户端使用当前时间减去开始获取锁的时间（步骤1的时间）就得到获取锁所需要的时间，当且仅当（n/2+1,这里是3个节点）的Redis节点都获取到锁，并且使用的时间小于锁失效的时间，锁才算获取成功。
4. 如果拿到了锁，key真正的有小时间 = 有效时间 - 获取锁使用的时间
5. 如果因为某些原因，获取锁失败（没有在n/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该**在所有的Redis实例上进行解锁**（即便某些实例根本就没有加锁成功，防止某些节点获取到锁之后，没有响应给客户端而导致一段时间内获取不到锁）



##### 2、用法

```java
Config config1 = new Config();
config1.useSingleServer().setAddress("redis://192.168.0.1:5378")
        .setPassword("a123456").setDatabase(0);
RedissonClient redissonClient1 = Redisson.create(config1);

Config config2 = new Config();
config2.useSingleServer().setAddress("redis://192.168.0.1:5379")
        .setPassword("a123456").setDatabase(0);
RedissonClient redissonClient2 = Redisson.create(config2);

Config config3 = new Config();
config3.useSingleServer().setAddress("redis://192.168.0.1:5380")
        .setPassword("a123456").setDatabase(0);
RedissonClient redissonClient3 = Redisson.create(config3);

String resourceName = "REDLOCK_KEY";

RLock lock1 = redissonClient1.getLock(resourceName);
RLock lock2 = redissonClient2.getLock(resourceName);
RLock lock3 = redissonClient3.getLock(resourceName);
// 向3个redis实例尝试加锁
RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
boolean isLock;
try {
    // isLock = redLock.tryLock();
    // 500ms拿不到锁, 就认为获取锁失败。10000ms即10s是锁失效时间。
    isLock = redLock.tryLock(500, 10000, TimeUnit.MILLISECONDS);
    System.out.println("isLock = "+isLock);
    if (isLock) {
        //TODO if get lock success, do something;
    }
} catch (Exception e) {
} finally {
    // 无论如何, 最后都要解锁
    redLock.unlock();
}
```

##### 3、唯一性id

实现分布式锁的一个非常重要的点就是set的value要具有唯一性，redisson的value是怎样保证value的唯一性呢？答案是**UUID+threadId**。入口在redissonClient.getLock("REDLOCK_KEY")，源码在Redisson.java和RedissonLock.java中：

```java
protected final UUID id = UUID.randomUUID();
String getLockName(long threadId) {
    return id + ":" + threadId;
}
```



##### 4、获取锁

获取锁的代码为redLock.tryLock()或者redLock.tryLock(500, 10000, TimeUnit.MILLISECONDS)，两者的最终核心源码都是下面这段代码，只不过前者获取锁的默认租约时间（leaseTime）是LOCK_EXPIRATION_INTERVAL_SECONDS，即30s：

```java
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);
    // 获取锁时需要在redis实例上执行的lua命令
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
              // 首先分布式锁的KEY不能存在，如果确实不存在，那么执行hset命令（hset REDLOCK_KEY uuid+threadId 1），并通过pexpire设置失效时间（也是锁的租约时间）
              "if (redis.call('exists', KEYS[1]) == 0) then " +
                  "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              // 如果分布式锁的KEY已经存在，并且value也匹配，表示是当前线程持有的锁，那么重入次数加1，并且设置失效时间
              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                  "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              // 获取分布式锁的KEY的失效时间毫秒数
              "return redis.call('pttl', KEYS[1]);",
              // 这三个参数分别对应KEYS[1]，ARGV[1]和ARGV[2]
                Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
}
```

获取锁的命令中，

- **KEYS[1]**就是Collections.singletonList(getName())，表示分布式锁的key，即REDLOCK_KEY；
- **ARGV[1]**就是internalLockLeaseTime，即锁的租约时间，默认30s；
- **ARGV[2]**就是getLockName(threadId)，是获取锁时set的唯一值，即UUID+threadId：



##### 5、释放锁

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    // 释放锁时需要在redis实例上执行的lua命令
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            // 如果分布式锁KEY不存在，那么向channel发布一条消息
            "if (redis.call('exists', KEYS[1]) == 0) then " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; " +
            "end;" +
            // 如果分布式锁存在，但是value不匹配，表示锁已经被占用，那么直接返回
            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                "return nil;" +
            "end; " +
            // 如果就是当前线程占有分布式锁，那么将重入次数减1
            "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
            // 重入次数减1后的值如果大于0，表示分布式锁有重入过，那么只设置失效时间，还不能删除
            "if (counter > 0) then " +
                "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                "return 0; " +
            "else " +
                // 重入次数减1后的值如果为0，表示分布式锁只获取过1次，那么删除这个KEY，并发布解锁消息
                "redis.call('del', KEYS[1]); " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; "+
            "end; " +
            "return nil;",
            // 这5个参数分别对应KEYS[1]，KEYS[2]，ARGV[1]，ARGV[2]和ARGV[3]
            Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(threadId));

}
```

##### 6、获取锁源码

```java

/**
 * 当前源码以5个Redis实例来进行讲解
 * Params:
 *      waitTime – the maximum time to acquire the lock
 *		leaseTime – lease time
 *		unit – time unit
 */
@Override
    public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
//        try {
//            return tryLockAsync(waitTime, leaseTime, unit).get();
//        } catch (ExecutionException e) {
//            throw new IllegalStateException(e);
//        }
        long newLeaseTime = -1;
        if (leaseTime != -1) {
            if (waitTime == -1) {
                newLeaseTime = unit.toMillis(leaseTime);//转毫秒
            } else {
                newLeaseTime = unit.toMillis(waitTime)*2;//等待时间*2
            }
        }
        
        long time = System.currentTimeMillis();//当前时间
        long remainTime = -1;//剩余时间
        if (waitTime != -1) {
            remainTime = unit.toMillis(waitTime);//初始化 剩余时间 = 等待时间，随着每个实例获取锁 剩余时间会逐渐减少： 剩余时间 = 等待时间 - 每个实例获取锁的时间
        }
        long lockWaitTime = calcLockWaitTime(remainTime);//Math.max(remainTime / locks.size(), 1) 每个实例获取锁最大的等待时间 例如 当前有五个实例，lockWaitTime算出的结果就是每个实例平均的等待时间
        
        int failedLocksLimit = failedLocksLimit();//locks.size() - minLocksAmount(locks) = 2;  minLocksAmount{locks.size()/2 + 1} 允许多少个实例获取锁失败
        List<RLock> acquiredLocks = new ArrayList<>(locks.size());
        for (ListIterator<RLock> iterator = locks.listIterator(); iterator.hasNext();) {//开始遍历 逐个实例获取锁
            RLock lock = iterator.next();
            boolean lockAcquired;
            try {
                if (waitTime == -1 && leaseTime == -1) {
                    lockAcquired = lock.tryLock();//马上获取锁
                } else {
                    long awaitTime = Math.min(lockWaitTime, remainTime);//因为remainTime会随着获取锁的过程逐渐减少 所以awaitTime到最后可能awaitTime = remainTime
                    lockAcquired = lock.tryLock(awaitTime, newLeaseTime, TimeUnit.MILLISECONDS);//获取锁
                }
            } catch (RedisResponseTimeoutException e) {
                unlockInner(Arrays.asList(lock));
                lockAcquired = false;
            } catch (Exception e) {
                lockAcquired = false;
            }
            
            if (lockAcquired) {//获取锁成功并记录
                acquiredLocks.add(lock);
            } else {//获取失败
                if (locks.size() - acquiredLocks.size() == failedLocksLimit()) {//如果已经达到最大失败次数 就不需要继续获取了
                    break;
                }

                if (failedLocksLimit == 0) {//最大失败次数为0
                    unlockInner(acquiredLocks);//释放之前已经获取成功的锁
                    if (waitTime == -1) {
                        return false;
                    }
                    failedLocksLimit = failedLocksLimit();//重置最大失败次数 failedLocksLimit = 2
                    acquiredLocks.clear();
                    // reset iterator
                    while (iterator.hasPrevious()) {
                        iterator.previous();
                    }
                } else {
                    failedLocksLimit--;//最大失败次数为-1
                }
            }
            
            if (remainTime != -1) {
                remainTime -= System.currentTimeMillis() - time;//剩余时间 = 剩余时间 - 获取锁所需要的的时间
                time = System.currentTimeMillis();//重置开始时间
                if (remainTime <= 0) {//如果剩余时间为0 释放锁 获取所失败
                    unlockInner(acquiredLocks);
                    return false;
                }
            }
        }

        if (leaseTime != -1) {
            acquiredLocks.stream()
                    .map(l -> (RedissonLock) l)
                    .map(l -> l.expireAsync(unit.toMillis(leaseTime), TimeUnit.MILLISECONDS))//异步设置锁的过期时间
                    .forEach(f -> f.syncUninterruptibly());
        }
        
        return true;
    }
```



#### 五、信号量

> 基于Redis的Redisson的分布式信号量（[Semaphore](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphore.html)）Java对象`RSemaphore`采用了与`java.util.concurrent.Semaphore`相似的接口和用法。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreRx.html)的接口。

```java
//初始化信号量 set car 3
@RequestMapping("/park")
@ResponseBody
public String park() {
    RSemaphore semaphore = redissonClient.getSemaphore("car");
    try {
        semaphore.acquire();//阻塞性等待
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "park success";
}

@RequestMapping("/park")
    @ResponseBody
    public String park() {
        RSemaphore semaphore = redissonClient.getSemaphore("car");
        boolean b = semaphore.tryAcquire();//立即返回
        return "park success > "+b;
    }

@RequestMapping("/leave")
@ResponseBody
public String leave() {
    RSemaphore semaphore = redissonClient.getSemaphore("car");
    semaphore.release();
    return "leave success";
}
```



#### 六、闭锁

```java
/**
     * 学校放假锁门 5个班级的人全部走完 才能锁门
     * @return
     */
    @GetMapping("/lockDoor")
    public String lockDoor() throws Exception{
        RCountDownLatch door = redissonClient.getCountDownLatch("door");
        door.trySetCount(5);
        door.await();//等待五个班走完才能关门
        return "放假了  关门了。。。。。。";
    }

    @GetMapping("/gogogo/{classId}")
    public String gogogo(@PathVariable String classId){
        RCountDownLatch door = redissonClient.getCountDownLatch("door");
        door.countDown();//计数减一
        return classId+" 班的人全走了。。。。";
    }

```



#### 七、公平锁

> 基于Redis的Redisson分布式可重入公平锁也是实现了`java.util.concurrent.locks.Lock`接口的一种`RLock`对象。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockRx.html)的接口。它保证了当多个Redisson客户端线程同时请求加锁时，优先分配给先发出请求的线程。所有请求线程会在一个队列中排队

```java
/**公平锁*/
@GetMapping("/fairLock/{orderNo}")
public String fairLock(@PathVariable String orderNo){
    RLock fairLock = redissonClient.getFairLock("fairLock");
    System.out.println("公平锁 订单号：" + orderNo + "等待处理。。。。。");
    fairLock.lock(12,TimeUnit.SECONDS);
    try {
        TimeUnit.SECONDS.sleep(10);
        System.out.println("订单号：" + orderNo + "处理成功");
        return "订单号："+orderNo+"处理成功";
    } catch (Exception e) {
        e.printStackTrace();
    }finally {
        fairLock.unlock();
    }
    return "订单号："+orderNo+"处理失败";
}

/**普通锁*/
@GetMapping("/noFairLock/{orderNo}")
public String noFairLock(@PathVariable String orderNo){
    RLock lock = redissonClient.getLock("noFairLock");
    System.out.println("普通锁 订单号：" + orderNo + "等待处理。。。。。");
    lock.lock(12,TimeUnit.SECONDS);
    try {
        TimeUnit.SECONDS.sleep(10);
        System.out.println("订单号：" + orderNo + "处理成功");
        return "订单号："+orderNo+"处理成功";
    } catch (Exception e) {
        e.printStackTrace();
    }finally {
        lock.unlock();
    }
    return "订单号："+orderNo+"处理失败";
}
```



![image-20210907094725705](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210907094725705.png)



![image-20210907094906593](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210907094906593.png)

