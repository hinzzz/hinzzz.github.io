---
title: RabbitMQ 入门教程
author: hinzzz
categories: RabbitMQ
date: 2021/08/06
keywords: [RabbitMQ安装,RabbitMQ环境搭建,RabbitMQ整合springboot]
description: 本文主要介绍RabbitMQ相关入门教程，包括RabbitMQ的安装及使用，RabbitMQ的应用场景，RabbitMQ与springboot进行整合
---



### RabbitMQ

#### 安装与开发环境搭建

##### 1、docker安装

```shell
docker pull rabbitmq:3.8.5-management

docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 -v /mydata/rabbitmq/data:/var/lib/rabbitmq --hostname myRabbit -e RABBITMQ_DEFAULT_VHOST=my_vhost  -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin rabbitmq:3.8.5-management
```

安装成功访问控制台http://IP:15672/

![image-20210806151628237](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210806151628237.png)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.4.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.hinz</groupId>
    <artifactId>rabbitmq</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>rabbitmq</name>
    <description>RabbitMQ DEMO</description>

    <properties>
        <java.version>1.8</java.version>
        <project.build.jdk>1.8</project.build.jdk>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.14</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-rabbit-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```







#### 一、工作队列

在实际生产中，当消息队列中有过多的消息等待消费时，我们使用一个消费者来处理消息显然是不够的，我们可以增加消费者，来共享消息队列中的消息，进行任务处理

消费一条消息往往比产生一条消息慢很多，为了防止消息积压，一般需要开启多个工作线程同时消费消息。在 RabbitMQ 中，我们可以创建多个 Consumer 消费同一队列

##### 1、轮询分发消息

###### 生产者

```java
public class Task01 {
    private final static String QUEUE_NAME="work";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            channel.queueDeclare(QUEUE_NAME,false,false,false,null);
            System.out.println("请输入消息：");
            Scanner sc = new Scanner(System.in);
            while (sc.hasNext()){
                String msg = sc.next();
                channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
                System.out.println("发送消息完成："+new String(msg.getBytes()));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

###### 消费者1

```java
public class Woker01 {
    private final static String QUEUE_NAME="work";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        DeliverCallback deliverCallback =  (consumerTag, message) ->{
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("consumerTag = " + consumerTag);
            System.out.println("工作线程01：" + new String (message.getBody())+" "+ LocalDateTime.now().toString());
        };

        CancelCallback cancelCallback = consumerTag ->{
            System.out.println("consumerTag = " + consumerTag);
        };
        try {
            System.out.println("工作线程01 等待消费");
            channel.basicConsume(QUEUE_NAME,true,deliverCallback,cancelCallback);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

###### 消费者2

```java
public class Woker02 {
    private final static String QUEUE_NAME="work";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        DeliverCallback deliverCallback =  (consumerTag, message) ->{
             try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("consumerTag = " + consumerTag);
            System.out.println("工作线程02：" + new String (message.getBody())+" "+ LocalDateTime.now().toString());
        };
        CancelCallback cancelCallback = consumerTag ->{
            System.out.println("consumerTag = " + consumerTag);
        };
        try {
            System.out.println("工作线程02 等待消费");
            channel.basicConsume(QUEUE_NAME,true,deliverCallback,cancelCallback);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

###### 运行结果

消费者1和2消费消息的速度不一样，按道理来说处理速度快的消费者应该消费更多的消息，但是他们各自轮流消费了所有的消息，why?

RabbitMQ 默认将消息顺序发送给下一个消费者，不管每个消费者的消费效率，这样，每个消费者会得到相同数量的消息。即，**轮询分发模式**。

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/work_producer.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=5MAh2pqoXVz7S2s%2BXDqKFV4QDao%3D)

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/work_consumer1.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=apFRKpqLeETl0Of6Oyvmg1XNOlg%3D)

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/work-consumer2.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=A3Nxcn6KmtVasQUIFkw2Jaq%2B39o%3D)





#### 二、消息应答

> 消费者在处理一个任务的过程中，处理失败或者服务器宕机，会发生什么情况？
>
> RabbitMQ一旦向消费者传递了一条消息，变立即将改消息标志位已删除。在这种情况下，突然有某个消费者挂掉了，我们将丢失正在处理的消息。以及后续发送给该消费者的消息，它都无法收到。

##### 1、概念

​	为了保证消息在传递的过程中不丢失，RabbitMQ引入了**消息应答机制**，消息应答就是：消费者在接收到消息并完成消息处理之后，告诉RabbitMQ它已经处理了，可以将该条消息删掉。

##### 2、自动应答

​	消息发送后立即被认为已传递成功，这种模式需要在**高吞吐量和数据传输安全方面做权衡**，因为这种模式如果**消息在接收到之前**，消费者那边出现连接关闭，那么消息就会丢失了。当然另一方面这种消费模式消费者那边可以传递过载的消息，没有对传递消息的数量进行限制，	最终会导致消息堆积过多，内存耗尽。所以这种模式只适合消费者能够高效处理消息的情况下使用

##### 3、消息重新入队

​	如果消费者在处理消息的过程中，由于某些原因失去连接或者宕机，导致消息为发送ack确认，RabbitMQ将了解到消息未完全处理，并将其重新排队。如果此时其他消费者可以处理，它将很快将消息发给另一个消费者。这样，即时某个消费者偶尔死亡，也不会影响消息丢失。

##### 4、手动应答

```java
public class AckConsumer {
    public static final String TASK_QUEUE = "task";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            boolean autoAck = false;
            DeliverCallback deliverCallback = (consumerTag,  message) -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("消费消息："+new String(message.getBody()));
                /**
                 * 1、消息标记
                 * 2、false:只应答接收到的那个传递的消息 有5,6,7,8个消息 当前消息是8 只会对8进行应答
                 *    true:应答所.有消息 包括传过来的消息 会对5,6,7,8消息进行应答
                 *
                 */
                channel.basicAck(message.getEnvelope().getDeliveryTag(),false);
            };
            channel.basicConsume(TASK_QUEUE,autoAck,deliverCallback,a -> {});
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

##### 

#### 三、持久化

> 消息应答解决了任务处理的过程中不丢失的情况，但是如何保障当RabbitMQ宕机之后，消费者发送的过来的消息不丢失。默认情况下RabbitMQ退出或者由于某种情况宕机时，它忽略队列和消息，除非告诉他不这样做。**确保消息和队列不丢失，需要将队列和消息都标志为持久化**



##### 1、队列如何实现持久化

​		之前我们创建的队列都是非持久话的，如果RabbitMQ重启，队列就会被删掉，我们需要将队列声明为持久化

```java
boolean durable = true;
channel.queueDeclare(QUEUE_NAME,durable,false,false,null);
```

注意，如果之前已经声明过该队列，需要把该队列先删除，否则会报下面的错

```java
Caused by: com.rabbitmq.client.ShutdownSignalException: channel error; protocol method: #method<channel.close>(reply-code=406, reply-text=PRECONDITION_FAILED - inequivalent arg 'durable' for queue 'hello' in vhost '/': received 'true' but current is 'false', class-id=50, method-id=10)
```

在控制台我们可以看到该队列多了个持久化的标志，这时候重启RabbitMQ之后，**该队列依然存在，但是队列里的消息已经被删除了**

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/durable.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=ffIm0fKRQ582lahVy9KVE48F1JA%3D)



##### 2、消息持久化

要想让消息实现持久化，推送消息的时候好需要声明一个属性**MessageProperties.PERSISTENT_TEXT_PLAIN**

```java
channel.basicPublish("",QUEUE_NAME, MessageProperties.PERSISTENT_TEXT_PLAIN,msg.getBytes());
```

重启之后消息依然存在



将消息标志位持久化并不能保证完全不丢失消息，尽管告诉RabbitMQ将消息保存到磁盘，但是这里依然存在当消息刚准被存储在磁盘的时候，但是还没有存储完，消息还在一个持久化的间隔点，此时并没有真正写入磁盘。持久性保证并不强，但是对于简单的队列而言，已经绰绰有余了，



#### 四、不公平分发

> 从上面我们知道，RabbitMQ默认的消息分发策略是轮训分发，这种策略并不是很好。比如说，存在两个消费者A（处理速度快）,B（处理速度慢）,当他们在处理任务的时候，如果我们还是使用轮训分发的策略，A就会很快处理完任务，并处于空闲时间，而B则一直在干活，这种分发策略其实就不太好，如果队列还在不断的添加消息，就会导致B产生消息堆积，而A消费完之后处于空闲状态。我们想要的效果实际是 **能者多劳**



##### 1、预取值

```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

意思是消费者当前任务没有处理完或者没有应答，RabbitMQ先别分配新的消息给我，我目前只能处理一个任务，等我处理完先。然后RabbitMQ就

本身消息的发送就是异步发送的，所以在任何时候， channel 上肯定不止只有一个消息另外来自消费
者的手动确认本质上也是异步的。因此这里就存在一个未确认的消息缓冲区，因此希望开发人员能限制此
缓冲区的大小，以避免缓冲区里面无限制的未确认消息问题。这个时候就可以通过使用 basic.qos 方法设
置“预取计数”值来完成的。 该值定义通道上允许的未确认消息的最大数量。一旦数量达到配置的数量，
RabbitMQ 将停止在通道上传递更多消息，除非至少有一个未处理的消息被确认，例如，**假设在通道上有**
**未确认的消息 5、 6、 7， 8，并且通道的预取计数设置为 4，此时 RabbitMQ 将不会在该通道上再传递任何**
**消息，除非至少有一个未应答的消息被 ack。比方说 tag=6 这个消息刚刚被确认 ACK， RabbitMQ 将会感知**
**这个情况到并再发送一条消息。**消息应答和 QoS 预取值对用户吞吐量有重大影响。通常，**增加预取将提高**
**向消费者传递消息的速度**。 虽然自动应答传输消息速率是最佳的，但是，在这种情况下已传递但尚未处理
的消息的数量也会增加，从而增加了消费者的 RAM 消耗(随机存取存储器)应该小心使用具有无限预处理
的自动确认模式或手动确认模式，**消费者消费了大量的消息如果没有确认的话，会导致消费者连接节点的**
**内存消耗变大，所以找到合适的预取值是一个反复试验的过程，不同的负载该值取值也不同 100 到 300 范**
**围内的值通常可提供最佳的吞吐量，并且不会给消费者带来太大的风险**。**预取值为 1 是最保守的。当然这**
**将使吞吐量变得很低，特别是消费者连接延迟很严重的情况下，特别是在消费者连接等待时间较长的环境**
**中**。对于大多数应用来说，稍微高一点的值将是最佳的。  





![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/prefetch.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=1nXf3JauH9lGsZnkfnPyfq%2BiMo8%3D)

![]( http://hinzzz.oss-cn-shenzhen.aliyuncs.com/prefetch2.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=vh6RPipkqrv9dQOru6J9n98LUZM%3D)





#### 五、发布确认

##### 1、发布确认原理

> ​	生产者将信道设置成confirm模式，一旦信道进入confirm模式，所有在该信道发布的消息将会被指派一个唯一ID（从1开始）,一旦消息被投递到所匹配的消息之后，broker就会发送一个确认消息给生产者（包括消息的唯一ID）,这就使得生产者已经知道消息成功投递到目的队列了。如果消息是持久化的，那么确认消息会在将消息写入磁盘之后发送给生产者，此外也可以设置basic.ack的mutiple，表示到这个序号之前的的所有消息都已经应答。confirm模式最大的好处是，它是异步的，一旦发布一条消息，生产者应用程序就可以在等待返回确认的时候继续发送下一条消息，当消息最终确认之后，生产应用就可以通过回调来处理该确认消息。如果RabbitMQ因为内部原因导致消息丢失，就会发送一条nack消息，生产者同样可以在回调中处理该消息。

##### 2、发布确认策略

##### 3、开启发布确认的方法

```java
channel.confirmSelect();//开启确认
boolean confirm = channel.waitForConfirms();//等待确认
if(confirm){
    System.out.println("消息发送成功");
}
```

##### 4、单个确认

> ​	一种同步确认发布方式，也就是生产者发布一条消息之后，必须等待这条消息确认之后才能发布下一条消息，boolean waitForConfirms(long timeout)这个方法只有在消息被确认的时候才能返回，如果指定时间内没返回，抛出异常
>
> 缺点：发布速度特别慢，这种方式最多提供每秒百条数据的吞吐量



```java
package com.hinz.rabbitmq.msgConfirm;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;

public class SingleConfirm {
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            channel.confirmSelect();
            channel.queueDeclare("single-confirm",false,false,false,null);
            channel.confirmSelect();
            long begin = System.currentTimeMillis();
            for (int i = 0; i < 1000; i++) {
                String msg = i+"";
                channel.basicPublish("","single-confirm",null,msg.getBytes());
                boolean confirm = channel.waitForConfirms();
                if(confirm){
                    System.out.println("消息发送成功");
                }
            }
            long end = System.currentTimeMillis();
            System.out.println("共发送1000条消息，共耗时" + (end - begin));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

运行结果：单个共发送1000条消息，共耗时15395



##### 5、批量发布确认

> 上面单个确认的方式处理消息太慢，与单个消息确认相比，先发布一批消息，最后一起确认，可以极大的提高吞吐量。
>
> 缺点：当发生故障导致发布问题时，不知道是哪个信息出现问题，必须将整批消息保存到内存中，以记录消息。这种方案仍然是同步的，也一样阻塞消息的发布。



```java
package com.hinz.rabbitmq.msgConfirm;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;

public class BatchConfirm {
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            channel.queueDeclare("batch-confirm", false, false, false, null);
            channel.confirmSelect();
            int confirmCount = 200;
            int msgCount = 0;
            long begin = System.currentTimeMillis();
            for (int i = 0; i < 1000; i++) {
                String msg = i + "";
                channel.basicPublish("", "batch-confirm", null, msg.getBytes());
                msgCount++;
                if (msgCount == confirmCount) {
                    channel.waitForConfirms();
                    msgCount = 0;
                }
            }
            if(msgCount>0){
                channel.waitForConfirms();
            }

            long end = System.currentTimeMillis();

            System.out.println("批量共发送1000条消息，共耗时" + (end - begin));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

运行结果：批量共发送1000条消息，共耗时139

##### 6、异步确认

> 编程逻辑比较复杂，但性价比高，通过回调函数达到消息可靠性

```java
package com.hinz.rabbitmq.msgConfirm;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.ConfirmCallback;
import java.util.concurrent.ConcurrentNavigableMap;
import java.util.concurrent.ConcurrentSkipListMap;

public class AsyncConfirm {
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            channel.queueDeclare("async-confirm",false,false,false,null);
            channel.confirmSelect();

            /**
             * 线程安全有序的一个哈希表，适用于高并发的情况下
             * 1、轻松的将序号与消息关联
             * 2、轻松批量删除信息，只要传序号
             * 3、支持并发访问
             */
            ConcurrentSkipListMap<Long,String> waitConfirmMap = new ConcurrentSkipListMap<>();

            /**
             * 确认消息的一个回调
             * 1、消息序列号
             * 2、true：可以确认小于等于当前序列号的消息
             *    false：只能确认当前消息
             */

            ConfirmCallback confirmCallback = (deliveryTag,  multiple) ->{
                if(multiple){
                    //返回的是小于或等于当前序号未确认的消息 是一个map
                    ConcurrentNavigableMap<Long, String> confirmed = waitConfirmMap.headMap(deliveryTag, true);
                    //清楚部分未确认消息
                    confirmed.clear();
                }else {
                    //只删除当前序号的消息
                    waitConfirmMap.remove(deliveryTag);
                }
            };

            ConfirmCallback nackCallback = (deliveryTag,  multiple) ->{
                String msg = waitConfirmMap.get(deliveryTag);
                System.out.println("消息：" + msg + " 未被确认");
            };
            /**
             * 添加一个异步的确认监听器
             * 1、确认消息的回调
             * 2、未收到消息的回调
             */
            channel.addConfirmListener(confirmCallback,nackCallback);
            long begin = System.currentTimeMillis();
            for (int i = 0; i < 1000; i++) {
                String msg = i+"";
                /**
                 * channel.getNextPublishSeqNo() 获取下一个消息的序列号
                 * 通过序列号与消息体进行关联
                 * 全部都是未确认的消息
                 */
                waitConfirmMap.put(channel.getNextPublishSeqNo(),msg);
                channel.basicPublish("","async-confirm",null,msg.getBytes());
            }
            long end = System.currentTimeMillis();
            System.out.println("异步发布1000条消息，共消耗" + (end - begin));
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}

```



运行结果：异步发布1000条消息，共消耗57

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/asyncConfirm.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=2nYq6k%2Bc2hTI7LLOy4jfayLyJKc%3D)





#### 六、Exchange

> ​		RabbitMQ消息传递模型的核心思想是：生产者生产的消息不会直接传递给队列。实际上，通常生产者甚至都不知道消息传递到那些队列去了。
>
> ​		生产者只能将消息发送到交换机（exchange）,exchange的工作非常简单，一方面它接受生产者的消息，另一方面将他们推入到队列。交换机必须确切知道如何处理收到的消息。是应该把收到的消息放到特定的队列还是放到多个队列或者直接丢弃，这就由交换机的类型来决定

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/exchange.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=eZTHZMYDmlyE%2BCcPI4S8tTPHN78%3D)





##### 1、Exchange的类型

+ 直接（direct）
+ 主题（topic）
+ 标题（headers）
+ 扇出（fanout）

##### 2、匿名交换机

在之前的代码中我们并没有指定具体的exchange，仍然能实现消息的推送，是因为指定空字符串的时候RabbitMQ给我们指定了默认的交换机

```java
channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
```

##### 3、临时队列

>上面我们使用的队列都都指定了特定名称，消费者到指定的队列取消费消息，若我们每次都需要一个新的队列，我们可以使用RabbitMQ提供的随机队列，这个队列一旦断开了链接，就自动被删除

```java
String queueName = channel.queueDeclare().getQueue();
```

##### 4、绑定（bindings）

> binding其实是queue和exchange的桥梁，它告诉我们exchange和哪个队列进行了绑定关系，下面这张图就是队列Q1，Q2和exchange X进行了绑定，即队列只对它绑定的交换机的消息感兴趣

![]( http://hinzzz.oss-cn-shenzhen.aliyuncs.com/binding.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=nZzeEldBj7Tf3XUxA%2Fnfh0JtXHQ%3D)



##### 5、Fanout Exchange

> **将接受到所有消息广播到它所知道所有队列**

将消息发送到exchange而不是队列，消费者绑定exchange，当生产者发送消息到exchange的时候，exchange会将消息广播到与当前exchange绑定的所有队列中

```java
package com.hinz.rabbitmq.exchanges.fanout;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;

import java.util.Scanner;

public class Sender {
    public static final String EXCHANGE_NAME = "log";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            channel.exchangeDeclare(EXCHANGE_NAME,"fanout");
            Scanner scanner = new Scanner(System.in);
            System.out.println("请输入消息");
            while (scanner.hasNext()){
                String msg = scanner.next();
                channel.basicPublish(EXCHANGE_NAME,"",null,msg.getBytes());
                System.out.println("消息发送成功：" + msg);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

两个消费者

```java
package com.hinz.rabbitmq.exchanges.fanout;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;


public class Receiver01 {
    public static final String EXCHANGE_NAME = "log";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            String queueName = channel.queueDeclare().getQueue();
            channel.exchangeDeclare(EXCHANGE_NAME,"fanout");
            channel.queueBind(queueName,EXCHANGE_NAME,"");
            System.out.println("等待接收到消息。。。。。。。。。");
            DeliverCallback deliverCallback = (consumerTag,  message)->{
                System.out.println("Receiver01接收到消息：" + new String(message.getBody()));
            };
            channel.basicConsume(queueName,deliverCallback,(x,y)->{});
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

```java
package com.hinz.rabbitmq.exchanges.fanout;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;


public class Receiver02 {
    public static final String EXCHANGE_NAME = "log";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            String queueName = channel.queueDeclare().getQueue();
            channel.exchangeDeclare(EXCHANGE_NAME,"fanout");
            channel.queueBind(queueName,EXCHANGE_NAME,"");
            System.out.println("等待接收到消息>>>>>>>>>>");
            DeliverCallback deliverCallback = (consumerTag,  message)->{
                System.out.println("Receiver02接收到消息：" + new String(message.getBody()));
            };
            channel.basicConsume(queueName,deliverCallback,(x,y)->{});
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

##### 6、Direct Exchange

> 上面的fanout交换机可以将所有的消息都广播到与它绑定的队列中去，但是某个队列如果想只接收某些特定的消息，fanout交换机就无能为力了。**在这里我们使用direct这种类型类型进行替换，这种类型的工作方式是，消息只去到它绑定的routingKey队列中去。**

```java
package com.hinz.rabbitmq.exchanges.direct;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;

public class Sender {
    public static final String EXCHANGE_NAME = "direct-log";

    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            channel.exchangeDeclare(EXCHANGE_NAME, "direct");

            channel.basicPublish(EXCHANGE_NAME, "info", null, "info".getBytes());
            channel.basicPublish(EXCHANGE_NAME, "warn", null, "warn".getBytes());
            channel.basicPublish(EXCHANGE_NAME, "error", null, "error".getBytes());
            System.out.println("消息发送成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

```java
package com.hinz.rabbitmq.exchanges.direct;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;


public class Receiver01 {
    public static final String EXCHANGE_NAME = "direct-log";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            String queueName = channel.queueDeclare().getQueue();
            channel.exchangeDeclare(EXCHANGE_NAME,"direct");
            channel.queueBind(queueName,EXCHANGE_NAME,"info");
            System.out.println("等待接收到消息。。。。。。。。。");
            DeliverCallback deliverCallback = (consumerTag,  message)->{
                System.out.println("Receiver01接收到消息：" + new String(message.getBody()));
            };
            channel.basicConsume(queueName,deliverCallback,(x,y)->{});
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

消费者2绑定两个key

```java
package com.hinz.rabbitmq.exchanges.direct;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;


public class Receiver02 {
    public static final String EXCHANGE_NAME = "direct-log";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            String queueName = channel.queueDeclare().getQueue();
            channel.exchangeDeclare(EXCHANGE_NAME,"direct");
            channel.queueBind(queueName,EXCHANGE_NAME,"warn");
            channel.queueBind(queueName,EXCHANGE_NAME,"info");
            System.out.println("等待接收到消息>>>>>>>>>>");
            DeliverCallback deliverCallback = (consumerTag,  message)->{
                System.out.println("Receiver02接收到消息：" + new String(message.getBody()));
            };
            channel.basicConsume(queueName,deliverCallback,(x,y)->{});
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

```java
package com.hinz.rabbitmq.exchanges.direct;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;


public class Receiver03 {
    public static final String EXCHANGE_NAME = "direct-log";
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            String queueName = channel.queueDeclare().getQueue();
            channel.exchangeDeclare(EXCHANGE_NAME,"direct");
            channel.queueBind(queueName,EXCHANGE_NAME,"error");
            System.out.println("等待接收到消息》》》》》》》");
            DeliverCallback deliverCallback = (consumerTag,  message)->{
                System.out.println("Receiver02接收到消息：" + new String(message.getBody()));
            };
            channel.basicConsume(queueName,deliverCallback,(x,y)->{});
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

##### 7、Topics

> 在上面direct交换机中，我们实现了有选择性的接收消息，但是这种方式还不够灵活，比如说接收的日志类型有info.a和info.b，某个队列只想接收到info.a的消息，这个是好direct交换机就做不到了，我们可以使用topic类型来解决这个问题

###### 1、要求

> 不能超过255字节，必须是一个单词列表，以点号隔开 例如"aa.bb.cc.dd"，"ee.ff.gg.hh"
>
> *（星号）可以代替任意一个单词
>
> #（井号）可以代替一个或多个单词

###### 2、匹配案例

| `*.hinzzz.*` |   中间为hinzzz的三个单词    |
| :----------: | :-------------------------: |
|  `*.*.wlq`   | 最后一个单词为wlq的三个单词 |
|  `hello.#`   | 第一个单词为hello的多个单词 |

当一个队列绑定key是#，那么它接受所有数据，类似于fanout

当个队列绑定key没有出现*或者#，那么该队列类型就是direct

```java
package com.hinz.rabbitmq.exchanges.topic;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;

import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.Map;

public class EmitLogTopic {
    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv) throws Exception {
        try (Channel channel = RabbitMQUtils.getChannel()) {
            channel.exchangeDeclare(EXCHANGE_NAME, "topic");
            /**
             * Q1-->绑定的是
             * 中间带 hinzzz 带 3 个单词的字符串(*.hinzzz.*)
             * Q2-->绑定的是
             * 最后一个单词是 wlq 的 3 个单词(*.*.wlq)
             * 第一个单词是 lazy 的多个单词(lazy.#)
             *
             */
            Map<String, String> bindingKeyMap = new HashMap<>();
            bindingKeyMap.put("quick.hinzzz.wlq", "被队列 Q1Q2 接收到");
            bindingKeyMap.put("lazy.hinzzz.elephant", "被队列 Q1Q2 接收到");
            bindingKeyMap.put("quick.hinzzz.fox", "被队列 Q1 接收到");
            bindingKeyMap.put("lazy.brown.fox", "被队列 Q2 接收到");
            bindingKeyMap.put("lazy.pink.wlq", "虽然满足两个绑定但只被队列 Q2 接收一次");
            bindingKeyMap.put("quick.brown.fox", "不匹配任何绑定不会被任何队列接收到会被丢弃");
            bindingKeyMap.put("quick.hinzzz.male.wlq", "是四个单词不匹配任何绑定会被丢弃");
            bindingKeyMap.put("lazy.hinzzz.male.wlq", "是四个单词但匹配 Q2");
            for (Map.Entry<String, String> bindingKeyEntry : bindingKeyMap.entrySet()) {
                String bindingKey = bindingKeyEntry.getKey();
                String message = bindingKeyEntry.getValue();
                channel.basicPublish(EXCHANGE_NAME, bindingKey, null,
                        message.getBytes(StandardCharsets.UTF_8));
                System.out.println("生产者发出消息" + message);
            }
        }
    }
}

```

```java
package com.hinz.rabbitmq.exchanges.topic;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;

import java.nio.charset.StandardCharsets;

public class ReceiveLogsTopic01 {
    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "topic");
        //声明 Q1 队列与绑定关系
        String queueName = "Q1";
        channel.queueDeclare(queueName, false, false, false, null);
        channel.queueBind(queueName, EXCHANGE_NAME, "*.hinzzz.*");
        System.out.println("等待接收消息.....");
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println(" 接 收 队 列 :" + queueName + " 绑 定 键:" + delivery.getEnvelope().getRoutingKey() + ",消息:" + message);
        };
        channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {
        });
    }
}

```

```java
package com.hinz.rabbitmq.exchanges.topic;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;

import java.nio.charset.StandardCharsets;

public class ReceiveLogsTopic02 {
    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "topic");
        //声明 Q2 队列与绑定关系
        String queueName = "Q2";
        channel.queueDeclare(queueName, false, false, false, null);
        channel.queueBind(queueName, EXCHANGE_NAME, "*.*.wlq");
        channel.queueBind(queueName, EXCHANGE_NAME, "lazy.#");
        System.out.println("等待接收消息.....");
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println(" 接 收 队 列 :" + queueName + " 绑 定 键:" + delivery.getEnvelope().getRoutingKey() + ",消息:" + message);
        };
        channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {
        });
    }
}

```







#### 七、死信队列

> ​	死信：无法被消费的消息，由于某些特定的原因导致队列中的消息无法被消费，这样的消息如果没有后续处理，就变成了死信，有了死信，自然就有死信队列
>
> ​	应用场景：为了保证业务订单的消息数据不丢失，需要使用到RabbitMQ的死信队列机制，当消息发生异常的时候，将消息转移到死信队列中。

##### 1、死信的来源

+ 消息TTL过期
+ 队列达到最大长度（队列满了）
+ 消息被拒绝（basic.reject或basic.nack）并且requeue=false

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/dead-queue.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=K%2BZMvrky6uaPMXpnlyMHmqwPrwc%3D)

##### 2、消息TTL

设置10秒过期时间

```java
package com.hinz.rabbitmq.deadqueue;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;

public class Producer {
    private static final String NORMAL_EXCHANGE = "normal_exchange";

    public static void main(String[] argv) throws Exception {
        try (Channel channel = RabbitMQUtils.getChannel()) {
            channel.exchangeDeclare(NORMAL_EXCHANGE, BuiltinExchangeType.DIRECT);
            //设置消息的 TTL 时间
            AMQP.BasicProperties properties = new
                    AMQP.BasicProperties().builder().expiration("10000").build();
            //该信息是用作演示队列个数限制
            for (int i = 1; i < 11; i++) {
                String message = "info" + i;
                channel.basicPublish(NORMAL_EXCHANGE, "zhangsan", properties,
                        message.getBytes());
                System.out.println("生产者发送消息:" + message);
            }
        }
    }
}
```

```java
package com.hinz.rabbitmq.deadqueue;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;

import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.Map;

public class Consumer01 {
    //普通交换机名称
    private static final String NORMAL_EXCHANGE = "normal_exchange";
    //死信交换机名称
    private static final String DEAD_EXCHANGE = "dead_exchange";

    public static void main(String[] argv) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();//声明死信和普通交换机 类型为 direct
        channel.exchangeDeclare(NORMAL_EXCHANGE, BuiltinExchangeType.DIRECT);
        channel.exchangeDeclare(DEAD_EXCHANGE, BuiltinExchangeType.DIRECT);
        //声明死信队列
        String deadQueue = "dead-queue";
        //正常队列绑定死信队列信息
        Map<String, Object> params = new HashMap<>();
        //正常队列设置死信交换机 参数 key 是固定值
        params.put("x-dead-letter-exchange", DEAD_EXCHANGE);
        //正常队列设置死信 routing-key 参数 key 是固定值
        params.put("x-dead-letter-routing-key", "lisi");
        String normalQueue = "normal-queue";
        channel.queueDeclare(normalQueue, false, false, false, params);
        channel.queueBind(normalQueue, NORMAL_EXCHANGE, "zhangsan");
        System.out.println("等待接收消息.....");
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println("Consumer01 接收到消息" + message);
        };
        channel.basicConsume(normalQueue, true, deliverCallback, consumerTag -> {
        });
    }
}
```

启动消费者1马上关掉，模拟接收不到信息

生产者发送了10条消息，此时正常队列有10条消息

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/normal-queue.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=7%2FDCY5XUszrbBZ1sksLVAtzpeDQ%3D)

10秒过去 正常队列消息由于没有被消费 消息进入死信队列

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/10dead-queue.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=DFmELQsh5LxqSBiBo9%2FTBj6i2kk%3D)



启动消费者2消费死信队列的消息

```java
package com.hinz.rabbitmq.deadqueue;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;

import java.nio.charset.StandardCharsets;

public class Consumer02 {
    private static final String DEAD_EXCHANGE = "dead_exchange";
    public static void main(String[] argv) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();
        channel.exchangeDeclare(DEAD_EXCHANGE, BuiltinExchangeType.DIRECT);
        String deadQueue = "dead-queue";
        channel.queueDeclare(deadQueue, false, false, false, null);
        channel.queueBind(deadQueue, DEAD_EXCHANGE, "lisi");
        System.out.println("等待接收死信队列消息.....");
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println("Consumer02 接收死信队列的消息" + message);
        };
        channel.basicConsume(deadQueue, true, deliverCallback, consumerTag -> {
        });
    }
}

```

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/deadqueue-consum.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=JDAViRJtbukqI3sc71Oi2X2J0PY%3D)



##### 3、队列达到最大长度

```java
package com.hinz.rabbitmq.deadqueue.maxlength;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;

public class Producer {
    private static final String NORMAL_EXCHANGE = "normal_exchange";

    public static void main(String[] argv) throws Exception {
        try (Channel channel = RabbitMQUtils.getChannel()) {
            channel.exchangeDeclare(NORMAL_EXCHANGE, BuiltinExchangeType.DIRECT);
            //该信息是用作演示队列个数限制
            for (int i = 1; i < 11; i++) {
                String message = "info" + i;
                channel.basicPublish(NORMAL_EXCHANGE, "zhangsan", null,
                        message.getBytes());
                System.out.println("生产者发送消息:" + message);
            }
        }
    }
}

```

设置最大长度为6  params.put("x-max-length",6);

```java
package com.hinz.rabbitmq.deadqueue.maxlength;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;

import java.util.HashMap;
import java.util.Map;

public class Consumer01 {
    //普通交换机名称
    private static final String NORMAL_EXCHANGE = "normal_exchange";
    //死信交换机名称
    private static final String DEAD_EXCHANGE = "dead_exchange";
    public static void main(String[] argv) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();//声明死信和普通交换机 类型为 direct
        channel.exchangeDeclare(NORMAL_EXCHANGE, BuiltinExchangeType.DIRECT);
        channel.exchangeDeclare(DEAD_EXCHANGE, BuiltinExchangeType.DIRECT);
        //声明死信队列
        String deadQueue = "dead-queue";
        channel.queueDeclare(deadQueue, false, false, false, null);
        //死信队列绑定死信交换机与 routingkey
        channel.queueBind(deadQueue, DEAD_EXCHANGE, "lisi");
        //正常队列绑定死信队列信息
        Map<String, Object> params = new HashMap<>();
        //正常队列设置死信交换机 参数 key 是固定值
        params.put("x-dead-letter-exchange", DEAD_EXCHANGE);
        //正常队列设置死信 routing-key 参数 key 是固定值
        params.put("x-dead-letter-routing-key", "lisi");
        //设置正常队列长度的限制
        params.put("x-max-length",6);
        String normalQueue = "normal-queue";
        channel.queueDeclare(normalQueue, false, false, false, params);
        channel.queueBind(normalQueue, NORMAL_EXCHANGE, "zhangsan");
        System.out.println("等待接收消息.....");
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println("Consumer01 接收到消息"+message);
        };
        channel.basicConsume(normalQueue, true, deliverCallback, consumerTag -> {
        });
    }
}
```

多余的4个消息被转入到dead-queue

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/dead-maxlength.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=9nrr0jF13txuSvAXNASboN1voYE%3D)





##### 4、消息被拒



```java
package com.hinz.rabbitmq.deadqueue.reject;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DeliverCallback;

import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.Map;

public class Consumer01 {
    //普通交换机名称
    private static final String NORMAL_EXCHANGE = "normal_exchange";
    //死信交换机名称
    private static final String DEAD_EXCHANGE = "dead_exchange";

    public static void main(String[] argv) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();
        //声明死信和普通交换机 类型为 direct
        channel.exchangeDeclare(NORMAL_EXCHANGE, BuiltinExchangeType.DIRECT);
        channel.exchangeDeclare(DEAD_EXCHANGE, BuiltinExchangeType.DIRECT);
        //声明死信队列
        String deadQueue = "dead-queue";
        channel.queueDeclare(deadQueue, false, false, false, null);
        //死信队列绑定死信交换机与 routingkey
        channel.queueBind(deadQueue, DEAD_EXCHANGE, "lisi");//正常队列绑定死信队列信息
        Map<String, Object> params = new HashMap<>();
        //正常队列设置死信交换机 参数 key 是固定值
        params.put("x-dead-letter-exchange", DEAD_EXCHANGE);
        //正常队列设置死信 routing-key 参数 key 是固定值
        params.put("x-dead-letter-routing-key", "lisi");
        String normalQueue = "normal-queue";
        channel.queueDeclare(normalQueue, false, false, false, params);
        channel.queueBind(normalQueue, NORMAL_EXCHANGE, "zhangsan");
        System.out.println("等待接收消息.....");
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            if (message.equals("info5")) {
                System.out.println("Consumer01 接收到消息" + message + "并拒绝签收该消息");
                //requeue 设置为 false 代表拒绝重新入队 该队列如果配置了死信交换机将发送到死信队列中
                channel.basicReject(delivery.getEnvelope().getDeliveryTag(), false);
            } else {
                System.out.println("Consumer01 接收到消息" + message);
                channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
            }
        };
        boolean autoAck = false;
        channel.basicConsume(normalQueue, autoAck, deliverCallback, consumerTag -> {
        });
    }
}



Consumer01 接收到消息info1
Consumer01 接收到消息info2
Consumer01 接收到消息info3
Consumer01 接收到消息info4
Consumer01 接收到消息info5并拒绝签收该消息
Consumer01 接收到消息info6
Consumer01 接收到消息info7
Consumer01 接收到消息info8
Consumer01 接收到消息info9
Consumer01 接收到消息info10
```

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/dead-reject1.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=%2FldOdgDmJXF8%2FWBBugRc2QmAXtk%3D)

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/dead_reject.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=WOWEopGFRDG8x1o0itsKFSjjNU8%3D)



#### 八、延迟队列

> 延时队列：队列内部是有序的，最重要的特性体现在它的延时属性上，延时队列中的元素是希望在指定时间之前或者之后取出处理，简单来说，延时队列就是用来存放需要在指定时间被处理的元素队列

##### 1、使用场景

+ 订单在30分钟之内不支付取消订单
+ 新创建的店铺，如果在十天内没有上传商品，则自定发送消息提醒
+ 用户注册成功后，如果三天内没有登录自动提醒
+ 用户发起退款，如果三天内没有处理则自动通知相关运营人员
+ 预订会议后，需要在会议前10分钟通知相关人员



> ​		这些场景都有一个特点，需要在某个事件发生之后或者之前的指定时间点完成某一项任务，如：
> 发生订单生成事件，在十分钟之后检查该订单支付状态，然后将未支付的订单进行关闭；看起来似乎
> 使用定时任务，一直轮询数据，每秒查一次，取出需要被处理的数据，然后处理不就完事了吗？如果
> 数据量比较少，确实可以这样做，比如：对于“如果账单一周内未支付则进行自动结算”这样的需求，
> 如果对于时间不是严格限制，而是宽松意义上的一周，那么每天晚上跑个定时任务检查一下所有未支
> 付的账单，确实也是一个可行的方案。但对于数据量比较大，并且时效性较强的场景，如：“订单十
> 分钟内未支付则关闭“，短期内未支付的订单数据可能会有很多，活动期间甚至会达到百万甚至千万
> 级别，对这么庞大的数据量仍旧使用轮询的方式显然是不可取的，很可能在一秒内无法完成所有订单
> 的检查，同时会给数据库带来很大压力，无法满足业务要求而且性能低下。  



![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/purchase_order_uml.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=Nj2fjdCETUdESl3O%2BXswPHw2VBw%3D)

##### 2、TTL

> TTL：表明一条消息或一个队列最大存活时间，若超过这个时间，消息将变成死信，或者队列中所有的消息都变成死信。如果队列和消息都设置了TTL属性，那么最小的那个值生效。
>
> **如果使用在消息属性上设置 TTL 的方式，消息可能并不会按时“死亡“，因为 RabbitMQ 只会检查第一个消息是否过期，如果过期则丢到死信队列，如果第一个消息的延时时长很长，而第二个消息的延时时长很短，第二个消息并不会优先得到执行。**  

###### 1、给消息设置ttl

```java
package com.hinz.rabbitmq.deadqueue.ttl;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;

import java.nio.charset.StandardCharsets;


public class MsgTTLProd {
    public static void main(String[] args) {
        Channel channel = RabbitMQUtils.getChannel();
        try {
            channel.exchangeDeclare("msg-ttl", BuiltinExchangeType.DIRECT);
            channel.queueDeclare("msg-ttl", true, false, false, null);
            channel.queueBind("msg-ttl", "msg-ttl", "msg-ttl");
            //给消息单独设置属性，链式编程（构建器模式，也是一种设计模式）
            AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
                    .deliveryMode(2) //持久化消息
                    .contentEncoding("UTF-8") //消息编码
                    .expiration("10000") //TTL消息，10秒过期
                    .build();

            channel.basicPublish("msg-ttl", "msg-ttl", properties, "hello message ttl".getBytes(StandardCharsets.UTF_8));
            System.out.println("消息发送完毕-->");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```



###### 2、给队列设置ttl

```java
package com.hinz.rabbitmq.deadqueue.ttl;

import com.hinz.rabbitmq.utils.RabbitMQUtils;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;

import java.util.HashMap;
import java.util.Map;

public class QueueTTLProd {
    private static final String NORMAL_EXCHANGE = "normal_exchange";
    private static final String DEAD_EXCHANGE = "dead_exchange";

    public static void main(String[] argv) throws Exception {
        try (Channel channel = RabbitMQUtils.getChannel()) {
            channel.exchangeDeclare(NORMAL_EXCHANGE, BuiltinExchangeType.DIRECT);
            //设置消息的 TTL 时间
            AMQP.BasicProperties properties = new
                    AMQP.BasicProperties().builder().expiration("10000").build();
            Map<String, Object> params = new HashMap<>();
            //正常队列设置死信交换机 参数 key 是固定值
            params.put("x-dead-letter-exchange", DEAD_EXCHANGE);
            //正常队列设置死信 routing-key 参数 key 是固定值
            params.put("x-dead-letter-routing-key", "lisi");
            channel.queueDeclare("normal_exchange",false,false,false,params);
            channel.queueBind("normal_exchange",NORMAL_EXCHANGE,"zhangsan");

            //该信息是用作演示队列个数限制
            for (int i = 1; i < 11; i++) {
                String message = "info" + i;
                channel.basicPublish(NORMAL_EXCHANGE, "zhangsan", properties,
                        message.getBytes());
                System.out.println("生产者发送消息:" + message);
            }
        }
    }
}

```



##### 3、两种方式的区别

如果设置了队列的 TTL 属性，那么一旦消息过期，就会被队列丢弃(如果配置了死信队列被丢到死信队
列中)，而第二种方式，消息即使过期，也不一定会被马上丢弃，因为消息是否过期是在即将投递到消费者
之前判定的，如果当前队列有严重的消息积压情况，则已过期的消息也许还能存活较长时间；另外，还需
要注意的一点是，如果不设置 TTL，表示消  



##### 4.队列配置参数

+ x-message-ttl：Number
  1个发布的消息在队列中存在多长时间后被取消（单位毫秒）；

+ x-expires：Number
  当Queue（队列）在指定的时间未被访问，则队列将被自动删除；

+ x-max-length：Number
  队列所能容下消息的最大长度。当超出长度后，新消息将会覆盖最前面的消息，类似于Redis的LRU算法；

+ x-max-length-bytes：Number
  限定队列的最大占用空间，当超出后也使用类似于Redis的LRU算法；

+ x-overflow：String
  设置队列溢出行为。这决定了当达到队列的最大长度时，消息会发生什么，有效值为Drop Head或Reject Publish；

+ x-dead-letter-exchange：String
  有时候我们希望当队列的消息达到上限后溢出的消息不会被删除掉，而是走到另一个队列中保存起来；

+ x-dead-letter-routing-key：String
  如果不定义，则默认为溢出队列的routing-key，因此，一般和6一起定义；

+ x-max-priority：Number
  如果将一个队列加上优先级参数，那么该队列为优先级队列；
  1）、给队列加上优先级参数使其成为优先级队列
  x-max-priority=10【0-255取值范围】
  2）、给消息加上优先级属性
  通过优先级特性，将一个队列实现插队消费；

+ x-queue-mode：String
  队列类型x-queue-mode=lazy懒队列，在磁盘上尽可能多地保留消息以减少RAM使用，如果未设置，则队列将保留内存缓存以尽可能快地传递消息；

+ x-queue-master-locator：String
  将队列设置为主位置模式，确定在节点集群上声明时队列主位置所依据的规则；









#### 九、整合SpringBoot

```java
package com.hinz.rabbitmq.config;

import org.springframework.amqp.core.*;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class RabbitMQConfig {

    public static final String X_EXCHANGE = "X";
    public static final String QUEUE_A = "QA";
    public static final String QUEUE_B = "QB";
    public static final String QUEUE_C = "QC";

    public static final String DEAD_EXCHANGE = "Y";
    public static final String DEAD_QUEUE = "QD";


    @Bean("xExchange")
    public DirectExchange getXExchange(){
        return new DirectExchange(X_EXCHANGE);
    }

    @Bean("dExchange")
    public DirectExchange getDExchange(){
        return new DirectExchange(DEAD_EXCHANGE);
    }


    //声明队列a ttl为10 并绑定死信交换机D
    @Bean("aQueue")
    public Queue getAQueue(){
        Map<String,Object> params = new HashMap<>(3);
        //声明当前队列绑定的死信交换机
        params.put("x-dead-letter-exchange",DEAD_EXCHANGE);
        //声明死信的路由key
        params.put("x-dead-letter-routing-key","DD");
        //声明队列的ttl
        params.put("x-message-ttl",10000);
        return QueueBuilder.durable(QUEUE_A).withArguments(params).build();
    }

    //声明队列A绑定交换机X
    @Bean
    public Binding getAQueueBindXExchange(@Qualifier("xExchange") DirectExchange exchange,
                                          @Qualifier("aQueue") Queue queue){
        return BindingBuilder.bind(queue).to(exchange).with("XA");
    }

    //声明队列B ttl为40 并帮顶死信交换机D
    @Bean("bQueue")
    public Queue getBQueue(){
        Map<String,Object> params = new HashMap<>(3);
        //声明当前队列绑定的死信交换机
        params.put("x-dead-letter-exchange",DEAD_EXCHANGE);
        //声明死信的路由key
        params.put("x-dead-letter-routing-key","DD");
        //声明队列的ttl
        params.put("x-message-ttl",40000);
        return QueueBuilder.durable(QUEUE_B).withArguments(params).build();
    }



    //声明队列B 绑定交换机X
    @Bean
    public Binding getBQueueBindXExchange(@Qualifier("xExchange") DirectExchange exchange,
                                          @Qualifier("bQueue") Queue queue){
        return BindingBuilder.bind(queue).to(exchange).with("XB");
    }

    //声明死信队列
    @Bean("dQueue")
    public Queue getDQueue(){
        return new Queue(DEAD_QUEUE);
    }

    //声明死信队列d 绑定死信交换机
    @Bean
    public Binding getDQueueBindDExchange(@Qualifier("dExchange") DirectExchange exchange,
                                          @Qualifier("dQueue") Queue queue){
        return BindingBuilder.bind(queue).to(exchange).with("DD");
    }



    @Bean("cQueue")
    public Queue getCQueue(){
        Map<String,Object> params = new HashMap<>(3);
        //声明当前队列绑定的死信交换机
        params.put("x-dead-letter-exchange",DEAD_EXCHANGE);
        //声明死信的路由key
        params.put("x-dead-letter-routing-key","DD");
        return QueueBuilder.durable(QUEUE_C).withArguments(params).build();
    }


    @Bean
    public Binding getCQueueBindXExchange(@Qualifier("cQueue")Queue queue,
                                          @Qualifier("xExchange") DirectExchange exchange ){
        return BindingBuilder.bind(queue).to(exchange).with("XC");
    }

}

```

```java
package com.hinz.rabbitmq.consumer;

import com.rabbitmq.client.Channel;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Date;

/**
 * @author hinzzz
 * @date 2021/7/20 16:56
 * @desc
 */
@Slf4j
@Component
public class DeadQueueConsumer {

    @RabbitListener(queues = "QD")
    public void receiveMsg(Message msg, Channel channel){
        log.info("当前时间：{}，收到死信队列的消息：{}",new Date(),new String(msg.getBody()));
    }

}
```

```java
package com.hinz.rabbitmq.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Date;

/**
 * @author hinzzz
 * @date 2021/7/20 16:23
 * @desc
 */
@Slf4j
@RequestMapping("ttl")
@RestController
public class SendMsgController {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @GetMapping("send/{msg}")
    public String send(@PathVariable String msg){
        log.info("当前时间：{},发一条消息给两个ttl队列：{}",new Date(),msg);
        rabbitTemplate.convertAndSend("X","XA",msg);
        rabbitTemplate.convertAndSend("X","XB",msg);
        return "ok";
    }

    /**
     * 消费端设置自定义ttl
     * @param msg
     * @param ttlTime
     * @return
     */
    @GetMapping("autoSend/{msg}/{ttlTime}")
    public String autoSend(@PathVariable String msg,@PathVariable String ttlTime){
        rabbitTemplate.convertAndSend("X","XC",msg, correlationData->{
            correlationData.getMessageProperties().setExpiration(ttlTime);
            return correlationData;
        });
        return "ok";
    }
}
```

> **如果使用在消息属性上设置 TTL 的方式，消息可能并不会按时“死亡“，因为 RabbitMQ 只会检查第一个消息是否过期，如果过期则丢到死信队列，如果第一个消息的延时时长很长，而第二个消息的延时时长很短，第二个消息并不会优先得到执行。**  

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/msg-ttl1.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=c8VDbRhlbPY2qNRTjGH8Oq95sA4%3D)



每增加一个新的时间需求，就要新增一个队列，无法满足个性化需求



##### 1、延迟队列插件

> 上面提到的问题：
>
> 如果使用在消息属性上设置 TTL 的方式，消
> 息可能并不会按时“死亡“，因为 RabbitMQ 只会检查第一个消息是否过期，如果过期则丢到死信队列，
> 如果第一个消息的延时时长很长，而第二个消息的延时时长很短，第二个消息并不会优先得到执行。  

##### 2、安装步骤

[插件地址]: https://www.rabbitmq.com/community-plugins.html



```powershell
# 将下载的插件拷贝到docker容器中
docker cp /mydata/rabbitmq/plugins/rabbitmq_delayed_message_exchange-3.8.0.ez 972bd874b38b:/plugins
#进入容器
docker exec -it 972 /bin/bash
#启用插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
#查看插件
rabbitmq-plugins list
#重新启动容器
docker restart 972
```





未安装之前

![]( http://hinzzz.oss-cn-shenzhen.aliyuncs.com/default_add_exchange.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=t01BDnmkyzIKiH9gWt5WSYso9rY%3D)

安装之后

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/after_add_exchange.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=419VgHbeNu2KlGDgMqF2SL23M5k%3D)



##### 3、自定义交换机

> 自定义一种新的交换类型，该类型支持延迟投递机制，消息传递后并不会立即投递到目标队列中，而是存储在mnesia（一个分布式数据系统）中，当达到投递时间时，才会投递到目标队列中



```java
package com.hinz.rabbitmq.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.CustomExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

/**
 * @author hinzzz
 * @date 2021/7/22 11:04
 * @desc 延迟队列插件
 */

@Configuration
public class DelayedQueueConfig {

    public static final String DELAYED_QUEUE = "delayed_queue";
    public static final String DELAYED_EXCHANGE = "delayed_exchange";
    public static final String ROUTING_KEY = "delayed_key";


    /**
     * 自定义交换机
     * @return
     */
    @Bean("delayedExchange")
    public CustomExchange delayedExchange(){
        Map<String,Object> map  = new HashMap<>();
        map.put("x-delayed-type","direct");
        return new CustomExchange(DELAYED_EXCHANGE,"x-delayed-message",true,false,map);
    }

    @Bean("delayedQueue")
    public Queue getDelayedQueue(){
        return new Queue(DELAYED_QUEUE);
    }

    @Bean
    public Binding getDelayedQueueBindDelayedExchange(@Qualifier("delayedExchange") CustomExchange exchange,
    @Qualifier("delayedQueue") Queue queue){
        return BindingBuilder.bind(queue).to(exchange).with(ROUTING_KEY).noargs();
    }
}

```

```java
package com.hinz.rabbitmq.consumer;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Date;

/**
 * @author hinzzz
 * @date 2021/7/22 14:18
 * @desc
 */
@Component
@Slf4j
public class DelayedMsgConsumer {

    @RabbitListener(queues = "delayed_queue")
    public void receiveDelayQueue(Message msg){
      log.info("当前时间：{}，接收到队列delayed_queue的消息：{}",new Date(),new String(msg.getBody()));
    }
}

```

```java
package com.hinz.rabbitmq.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Date;

/**
 * @author hinzzz
 * @date 2021/7/22 14:12
 * @desc
 */
@RestController
@RequestMapping("delay")
@Slf4j
public class DelayedMsgController {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @RequestMapping("sendMsg/{msg}/{ttl}")
    public Object sendMsg(@PathVariable String msg,@PathVariable Integer ttl){
        rabbitTemplate.convertAndSend("delayed_exchange","delayed_key",msg,correlationData->{
            correlationData.getMessageProperties().setDelay(ttl);
            return correlationData;
        });
        log.info("当前时间：{}，发送消息：{}，过期时间：{}",new Date(),msg,ttl);
        return "ok";
    }

}

```

**先过期的消息优先被消费**

![](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/msg-ttl2.png?Expires=32500886400&OSSAccessKeyId=LTAI4G9rkBZLb3G51wiGr2sS&Signature=T%2BXeuUA%2Fgfvz%2BGxEdBNgOrOrOfg%3D)



##### 4、发布确认

> 如果由于一些不明原因导致rabbitmq重启，在rabbitmq重启期间生产者消息投递失败，导致消息丢失，需要手动处理喝回复。那么，如何才能进行rabbitmq的消息可靠投递呢，特别是这样比较极端的情况，rabbitmq集群不可用的时候，无法投递的消息该如何处理？



![image-20210728154304317](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210728154304317.png)



```properties
##开启发布确认 spring配置文件
spring.rabbitmq.publisher-confirm-type=correlated
NONE:禁用发布确认模式。默认值
correlated：发布消息到交换机后触发回调方法
simple:发布消息成功后调用waitForConfirms	效果和correlated一致，调用waitForConfirmOrDie等待返回结果，若返回的结果为false,channel通道则会关闭 无法发送消息 相当于同步确认
    
```

```java

package com.hinz.rabbitmq.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.PostConstruct;

@Slf4j
@RestController
public class MsgProducer implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    //rabbitTemplate注入之后设置该值
    @PostConstruct
    private void init(){
        rabbitTemplate.setConfirmCallback(this::confirm);
        /**
         * true：交换机无法将消息进行路由时，会将该消息返回给生产者
         * false：如果发现消息无法路由 直接丢弃
         */
        rabbitTemplate.setMandatory(true);
        rabbitTemplate.setReturnCallback(this::returnedMessage);
    }

    @GetMapping("sendMsg/{msg}")
    public void sendMsg(@PathVariable String msg){
        //让消息绑定一个id值
        CorrelationData correlationData = new CorrelationData("111");
        rabbitTemplate.convertAndSend("confirm_exchange","key1",msg+"key1",correlationData);
        log.info("发送消息id：{},内容：{}",correlationData.getId(),msg+"key1");

        CorrelationData correlationData2 = new CorrelationData("222");
        rabbitTemplate.convertAndSend("confirm_exchange_key2","key1",msg+"key2",correlationData2);
        log.info("发送消息id：{},内容：{}",correlationData2.getId(),msg+"key2");

        CorrelationData correlationData3 = new CorrelationData("333");
        rabbitTemplate.convertAndSend("confirm_exchange","key3",msg+"key3",correlationData3);
        log.info("发送消息id：{},内容：{}",correlationData3.getId(),msg+"key3");
    }

    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("消息：{}被退回，原因是：{}，交换机是：{}，routingKey是：{}",new String(message.getBody()),replyText,exchange,routingKey);
    }

    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        String id = correlationData.getId();
        if(ack){
            log.info("交换机收到消息，id：{}",id);
        }else{
            log.info("交换机未收到消息id：{}，原因是：{}",id,cause);
        }
    }
}


```



```java
package com.hinz.rabbitmq.config;

import org.springframework.amqp.core.*;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

/**
 * @author hinzzz
 * @date 2021/7/22 15:18
 * @desc
 */
@Component
public class ConfirmQueueConfig {
    public static final String CONFIRM_QUEUE = "confirm_queue";
    public static final String CONFIRM_EXCHANGE = "confirm_exchange";

    @Bean("confirm_queue")
    public Queue getConfirmQueue() {
        return QueueBuilder.durable(CONFIRM_QUEUE).build();
    }

    @Bean("confirm_exchange")
    public DirectExchange getConfirmExchange(){
        return new DirectExchange(CONFIRM_EXCHANGE);
    }


    @Bean
    public Binding getConfirmQueueExchange(@Qualifier("confirm_queue")Queue queue,
                                           @Qualifier("confirm_exchange")DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with("key1");
    }

}

```

```java
package com.hinz.rabbitmq.consumer;

import com.rabbitmq.client.Channel;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

/**
 * @author hinzzz
 * @date 2021/7/29 16:03
 * @desc
 */
@Component
@Slf4j
public class ConfirmConsumer {

    @RabbitListener(queues = "confirm_queue")
    public void comsume(Message msg, Channel channel){
        log.info("消息>>>>>>>>>>{} 被消费 ",new String(msg.getBody()));
    }

}

```



发送三次消息，第一个消息正常发送，第二个模拟exchange不存在，第三个消息模拟队列不存在，我们来看下运行结果

![image-20210729163917177](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210729163917177.png)

![image-20210729163602428](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210729163602428.png)



第一个消息被正常消费，第二个消息因为找不到与key2绑定的队列，从控制台rabbitmq控制台可以看出消息直接丢失了，第三条消息因为找不到交换机消息也丢失了









##### 5、回退消息

###### 1、Mandatory参数

> ​		**在仅开启了生产者确认机制的情况下，交换机接收到信息后，会直接给消息生产者发送确认消息，如果发现该消息不可路由（找不到对应的队列），那么消息会被直接丢弃。**此时生产者是不知道消息被丢弃这件事的。那么如何让无法被路由的消息帮我们想办法处理一下呢？
>
> ​		通过设置mandatory参数可以在当下拍戏传递过程中不可达目的地时将消息返回给生产者



```java
 /**
         * true：交换机无法将消息进行路由时，会将该消息返回给生产者
         * false：如果发现消息无法路由 直接丢弃
         */
        rabbitTemplate.setMandatory(true);
```



##### 6、备份交换机

> 有了 mandatory 参数和回退消息，我们获得了对无法投递消息的感知能力，有机会在生产者的消息
> 无法被投递时发现并处理。但有时候，我们并不知道该如何处理这些无法路由的消息，最多打个日志，然
> 后触发报警，再来手动处理。而通过日志来处理这些无法路由的消息是很不优雅的做法，特别是当生产者
> 所在的服务有多台机器的时候，手动复制日志会更加麻烦而且容易出错。而且设置 mandatory 参数会增
> 加生产者的复杂性，需要添加处理这些被退回的消息的逻辑。如果既不想丢失消息，又不想增加生产者的
> 复杂性，该怎么做呢？前面在设置死信队列的文章中，我们提到，可以为队列设置死信交换机来存储那些
> 处理失败的消息，可是这些不可路由消息根本没有机会进入到队列，因此无法使用死信队列来保存消息。
> 在 RabbitMQ 中，有一种备份交换机的机制存在，可以很好的应对这个问题。什么是备份交换机呢？备份
> 交换机可以理解为 RabbitMQ 中交换机的“备胎”，当我们为某一个交换机声明一个对应的备份交换机时，
> 就是为它创建一个备胎，当交换机接收到一条不可路由消息时，将会把这条消息转发到备份交换机中，由
> 备份交换机来进行转发和处理，通常备份交换机的类型为 Fanout ，这样就能把所有消息都投递到与其绑
> 定的队列中，然后我们在备份交换机下绑定一个队列，这样所有那些原交换机无法被路由的消息，就会都
> 进入这个队列了。当然，我们还可以建立一个报警队列，用独立的消费者来进行监测和报警。  

配置类

```java
package com.hinz.rabbitmq.config;

import org.springframework.amqp.core.*;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

/**
 * @author hinzzz
 * @date 2021/7/22 15:18
 * @desc
 */
@Component
public class ConfirmQueueConfig {
    public static final String CONFIRM_QUEUE = "confirm_queue";
    public static final String CONFIRM_EXCHANGE = "confirm_exchange";
    public static final String BACKUP_EXCHANGE = "backup_exchange";
    public static final String BACKUP_QUEUE = "backup_queue";
    public static final String WARNING_QUEUE = "warning_queue";

    @Bean("confirm_queue")
    public Queue getConfirmQueue() {
        return QueueBuilder.durable(CONFIRM_QUEUE).build();
    }

    @Bean("backup_queue")
    public Queue getBackupQueue() {
        return QueueBuilder.durable(BACKUP_QUEUE).build();
    }

    @Bean("warning_queue")
    public Queue getWarningQueue() {
        return QueueBuilder.durable(WARNING_QUEUE).build();
    }

    @Bean("backup_exchange")
    public FanoutExchange getBackupExchange(){
        return new FanoutExchange(BACKUP_EXCHANGE);
    }

    @Bean
    public Binding getConfirmQueueExchange(@Qualifier("confirm_queue")Queue queue,
                                           @Qualifier("confirm_exchange")DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with("key1");
    }

    @Bean("confirm_exchange")
    public DirectExchange getConfirmExchange(){
        ExchangeBuilder exchangeBuilder = ExchangeBuilder.directExchange(CONFIRM_EXCHANGE)
                .durable(true)
                .withArgument("alternate-exchange", BACKUP_EXCHANGE);
        return exchangeBuilder.build();
    }



   /* @Bean
    public Binding getConfirmQueueExchange3(@Qualifier("confirm_queue")Queue queue,
                                            @Qualifier("confirm_exchange")DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with("key3");
    }*/

    @Bean
    public Binding warningBind(@Qualifier("warning_queue")Queue queue,
                                            @Qualifier("backup_exchange")FanoutExchange exchange){
        return BindingBuilder.bind(queue).to(exchange);
    }

    @Bean
    public Binding backupBind(@Qualifier("backup_queue")Queue queue,
                               @Qualifier("backup_exchange")FanoutExchange exchange){
        return BindingBuilder.bind(queue).to(exchange);
    }

}

```

消费者

```java
package com.hinz.rabbitmq.consumer;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

/**
 * @author hinzzz
 * @date 2021/8/4 10:27
 * @desc
 */
@Component
@Slf4j
public class WarningConsumer {

    @RabbitListener(queues = "warning_queue")
    public void consumer(Message msg){
        log.info(">>>>>>>>>>>>>>>>>>>>>>>发现不可路由信息：{}",new String(msg.getBody()));
    }
}

```

生产者

```java
    @GetMapping("/backupExchange/{msg}")
    public void backupExhchange(@PathVariable String msg){
        CorrelationData correlationData3 = new CorrelationData("backupExchange");
        rabbitTemplate.convertAndSend("confirm_exchange","nokey",msg+"key1",correlationData3);
        log.info("发送消息id：{},内容：{}",correlationData3.getId(),msg+"key1");
    }
```

运行结果

![image-20210804111411485](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210804111411485.png)





#### 十、常用注解

##### 1、@RabbitListener和@RabbitHandler

```java
@RabbitListener(bindings = @QueueBinding(
        exchange = @Exchange(value = "QQ", durable = "true", type = "topic"),
        value = @Queue(value = "WW", durable = "true"),
        key = "hinzzz.#"
))
```

​		用在方法上面，表明这个是个消费方法

​		用在类上面，通过 @RabbitListener 的 bindings 属性声明 Binding（若 RabbitMQ 中不存在该绑定所需要的 Queue、Exchange、RouteKey 则自动创建，若存在则抛出异常）

​		配合@RabbitHandler使用，@RabbitListener 标注在类上面表示当有收到消息的时候，就交给 @RabbitHandler 的方法处理，具体使用哪个方法处理，根据 MessageConverter 转换后的参数类型

生产者

```java
 @GetMapping("sendUser/{msg}")
    public String sendUser(@PathVariable String msg){
        rabbitTemplate.convertAndSend("QQ","hinzzz.Aaa",msg);
        rabbitTemplate.convertAndSend("QQ","hinzzz.Aaa", User.builder().userNo("1").userName(msg).build());
        log.info("当前时间：{},发送消息成功：{}",new Date());
        return "ok";
    }

```

消费者

```java
package com.hinz.rabbitmq.consumer;

import com.hinz.rabbitmq.bean.User;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@Slf4j
/**
 * payload是一种以JSON格式进行数据传输的一种方式。
 */
@RabbitListener(bindings = @QueueBinding(
        exchange = @Exchange(value = "QQ", durable = "true", type = "topic"),
        value = @Queue(value = "WW", durable = "true"),
        key = "hinzzz.#"
))
public class BindingsConsumer {


    @RabbitHandler
    public void consume(@Payload String msg) {
        log.info("接收的参数为字符串类型：{}",msg);
    }
    @RabbitHandler
    public void consume(@Payload User user) {
        log.info("接收的参数为对象：{}",user);
    }

}

```

![image-20210806141808218](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210806141808218.png)

##### 

#### 十一、消息序列化

> 涉及数据网络传输的应用序列化不可避免，发送端以某种规则将消息转成byte数据进行发送，接收端以该规则进行byte[]数组的解析，RabbitMQ的序列化是只Message的body属性，即我们需要真正传输的内容。RabbitMQ抽象出一个MessageConvert 接口，处理消息的序列化。其实现有 SimpleMessageConverter（默认）、Jackson2JsonMessageConverter 等。

##### 1、SimpleMessageConverter

对于要发送的消息体 body 为 byte[] 时不进行处理，如果是 String 则转成字节数组,如果是 Java 对象，则使用 jdk 序列化将消息转成字节数组，转出来的结果较大，含class类名，类相应方法等信息。因此性能较差

下面来看下发送的消息类型以及在控制台接收到我们发送的消息，可以看到当发送的消息是一个对象时，转出来的结果较大，浪费性能。

```java
rabbitTemplate.convertAndSend("QQ","hinzzz.Aaa",msg.getBytes(StandardCharsets.UTF_8)+" byte[]");
rabbitTemplate.convertAndSend("QQ","hinzzz.Aaa",msg+" String");
rabbitTemplate.convertAndSend("QQ","hinzzz.Aaa", User.builder().userNo("1").userName(msg).build());
```



![image-20210806143651790](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210806143651790.png)





##### 2、Jackson2JsonMessageConverter 

> 为了解决上面对象序列化产生的问题，我们需要将默认的序列化实现改为Jackson2JsonMessageConverter 
>
> 注意：被序列化对象应提供一个无参的构造函数，否则会抛出异常



```java

package com.hinz.rabbitmq.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.listener.RabbitListenerContainerFactory;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author hinzzz
 * @date 2021/8/6 10:13
 * @desc
 */
@Slf4j
@Configuration
public class GlobalConfig {

    @Bean
    public RabbitListenerContainerFactory<?> rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(new Jackson2JsonMessageConverter());
        return factory;
    }


    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(new Jackson2JsonMessageConverter());
        return template;
    }

}
```

我们再来发送一个对象看下控制台接收到的消息

```java
rabbitTemplate.convertAndSend("QQ","hinzzz.Aaa",msg.getBytes(StandardCharsets.UTF_8)+" byte[]");
rabbitTemplate.convertAndSend("QQ","hinzzz.Aaa",msg+" String");
rabbitTemplate.convertAndSend("QQ","hinzzz.Aaa", User.builder().userNo("1").userName(msg).build());
```

控制台可以看到user对象进行了json序列化，

![image-20210806150024874](http://hinzzz.oss-cn-shenzhen.aliyuncs.com/img/hinzzzimage-20210806150024874.png)









#### 十二、其他知识点

##### 1、幂等性

> 用户对于同一操作发起的一次请求或者多次请求的结果是一致的，**不会因为多次点击而产生了副作用。**
> 举个最简单的例子，那就是支付，用户购买商品后支付，**支付扣款成功，但是返回结果的时候网络异常**，
> 此时钱已经扣了，用户再次点击按钮，此时会进行第二次扣款，返回结果成功，用户查询余额发现多扣钱
> 了，流水记录也变成了两条。在以前的单应用系统中，我们只需要把数据操作放入事务中即可，发生错误
> 立即回滚，但是再响应客户端的时候也有可能出现网络中断或者异常  



##### 2、消息重复消费

> 消费者在消费 MQ 中的消息时， **MQ 已把消息发送给消费者，消费者在给 MQ 返回 ack 时网络中断**，
> 故 MQ 未收到确认信息，该条消息会重新发给其他的消费者，或者在网络重连后再次发送给该消费者，但
> 实际上该消费者已成功消费了该条消息，造成消费者消费了重复的消息。  

