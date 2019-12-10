# rabbitMq学习笔记

1、主流的消息中间件介绍

（1）ActiveMQ

分布式集群，zookeeper    主从。（性能不是很好）

（2）KAFKA

​	分布式发布订阅消息系统（消息不可靠性）

​	追求高吞吐量（高性能），目的在于日志的收集和传输，不支持事务。内存存储，没有磁盘的IO。

（3）RocketMQ

​	纯Java 开发。具有高吞吐，高可用性。适合大规模分布式系统应用。

啥都好，就是商业版收费。

（4）RabbitMQ

使用Erlang语言开发开源下消息队列系统。什么都好。

RabbitMQ 集群架构。

![](../images/QQ截图20190402221528.png)

1、MQ的衡量指标

服务性能，数据库，集群架构



二、rabbitMQ的核心知识点

​	rabbitMQ是一个开源的消息代理和队列服务器。用来通过普通的协议在完全不同的应用之间共享数据，基于AMQP协议实现。

rabbitMQ整体架构：

![img](../images/20171227105213712)

![](../images/QQ截图20190402223714.png)

rabbitMQ高性能的原因？

核心：Erlang语言，延迟和原生socket一样的延迟。

1、应用场景

大多应用中，可通过消息服务中间件来提升系统异步通信、扩展解耦能力。

1)、异步处理

如当用户提交注册信息后,要发送邮件或短信来通知用户，传统的处理方式

![img](../images/20181024100136855.png)

用户提交后将信息写入消息队列后立马返回结果，邮件/短信服务异步读取消息队列信息再发送

![img](../images/20181024100231520.png)

2)、应用解耦

如订单系统下单的信息要在库存系统中进行计算，传统的处理方式

![img](../images/20181024100927929.png)

通过消息队列传输数据进行应用解耦，订单和库存作为2个微服务，订单系统下单将数据写入消息队列，库存系统通过订阅消息，当队列中有消息时库存系统就会进行计算。

![img](../images/20181024101132704.png)

3)、流量削峰

如10万个用户同时请求，同时请求可能导致服务挂掉，可以设置消息队列为1万，请求过来1万放入队列然后读取处理，另外9万请求返回失败信息，下次再次请求。

![img](../images/20181024101630903.png)

AMQP协议是什么？

**高级消息队列协议**

AMQP核心概念
Server
       又称作Broker，用于接受客户端的连接，实现AMQP实体服务；

Connection
       连接，应用程序与Broker的网络连接；

Channel
       网络信道，几乎所有的操作都在Channel中进行，Channel是进行消息读写的通道。客户端可建立多个Channel，每个Channel代表一个会话任务；

Message
       消息，服务器和应用程序之间传送的数据，有Properties和Body组成。Properties可以对消息进行修饰，比如消息的优先级、延迟等高级特性；Body则是消息体内容，即我们要传输的数据；

​       仅仅创建了客户端到Broker之间的连接后，客户端还是不能发送消息的。需要为每一个Connection创建Channel，AMQP协议规定只有通过Channel才能执行AMQP的命令。一个Connection可以包含多个Channel。之所以需要Channel，是因为TCP连接的建立和释放都是十分昂贵的，如果一个客户端每一个线程都需要与Broker交互，如果每一个线程都建立一个TCP连接，暂且不考虑TCP连接是否浪费，就算操作系统也无法承受每秒建立如此多的TCP连接。

RabbitMQ建议客户端线程之间不要共用Channel，至少要保证共用Channel的线程发送消息必须是串行的，

但是建议尽量共用Connection。

Virtual Host
       虚拟地址，是一个逻辑概念，用于进行逻辑隔离，是最上层的消息路由。

​	一个Virtual Host里面可以有若干个Exchange和Queue，同一个Virtual Host里面不能有相同名称的Exchange或者	Queue；
​       Virtual Host是权限控制的最小粒度；

Exchange
       交换机，用于接收消息，可根据路由键将消息转发到绑定的队列；

Binding
       Exchange和Queue之间的虚拟连接，Exchange在与多个Message Queue发生Binding后会生成一张路由表，路由表中存储着Message Queue所需消息的限制条件即Binding Key。当Exchange收到Message时会解析其Header得到Routing Key，Exchange根据Routing Key与Exchange Type将Message路由到Message Queue。Binding Key由Consumer在Binding Exchange与Message Queue时指定，而Routing Key由Producer发送Message时指定，两者的匹配方式由Exchange Type决定。

Routing Key
       一个路由规则，虚拟机可用它来确定如何路由一个特定的消息；

Queue

​       也称作Message Queue，即消息队列，用于保存消息并将他们转发给消费者；

### 路由键与交换机

- Fanout Exchange – 不处理路由键。你只需要简单的将队列绑定到交换机上。一个发送到交换机的消息都会被转发到与该交换机绑定的所有队列上。很像子网广播，每台子网内的主机都获得了一份复制的消息。Fanout交换机转发消息是最快的。

![img](../images/13510795-f4b9c5c320d8b8ce.webp)

Direct Exchange – 处理路由键。需要将一个队列绑定到交换机上，要求该消息与一个特定的路由键完全匹配。这是一个完整的匹配。如果一个队列绑定到该交换机上要求路由键 “dog”，则只有被标记为“dog”的消息才被转发，不会转发dog.puppy，也不会转发dog.guard，只会转发dog。一一对应

![img](../images/13510795-6579c0cefd90f653.png)

Topic Exchange – 将路由键和某模式进行匹配（模糊匹配）。此时队列需要绑定要一个模式上。

​	符号“#”匹配一个或多个词，

​	符号“**”匹配不多不少一个词。*

因此“audit.#”能够匹配到“audit.irs.corporate”，但是“audit.*” 只会匹配到“audit.irs”。

topic交换机是如何工作的：

![](../images/QQ截图20190404144543.png)



使用过程：

生产者：

创建ConnectionFactory并设置参数

创建Connection

创建Channel

声明exchange和路由键

发送消息



消费者：

创建ConnectionFactory并设置参数

创建Connection

创建Channel

声明交换机

声明一个队列

绑定交换机和队列（通过路由键，不同的交换机类型决定消息的到达的Channel）在生产者，消费段都可以，只是设置会冲突

创建消费者

给消费者设置对应的消息队列

接收消息

整体流程：

![](../images/QQ截图20190404153114.png)

绑定交换机和队列（通过路由键，不同的交换机类型决定消息的到达的队列）

![1554363173599](../images/1554363173599.png)

### 实际问题

![1554364352890](../images/1554364352890.png)

![1554364412268](../images/1554364412268.png)

1、消息如何保障100%投递成功？

![1554364530078](../images/1554364530078.png)

​	

![](../images/QQ截图20190404155739.png)

![1554364740377](../images/1554364740377.png)

核心是少做一次DB。提高并发量，拆分服务。

![1554365393568](../images/1554365393568.png)



幂等性

借鉴乐观锁。

解决重复消费，所有的相同的操作，得到的结果一样的。A^n = A

![1554365673832](../images/1554365673832.png)

![1554365697936](../images/1554365697936.png)

指纹码：比如时间戳或者业务规则生成的码。唯一且不同。

分库分表：比如哈希算哈。

![1554366040701](../images/1554366040701.png)



![1554366114182](../images/1554366114182.png)

![1554366148548](../images/1554366148548.png)

![1554366168018](../images/1554366168018.png)



![1554366838833](../images/1554366838833.png)

![1554366886363](../images/1554366886363.png)

![1554366913222](../images/1554366913222.png)

![1554367417993](../images/1554367417993.png)

```java
//自定义Consumer类
public class MyConsumer extends DefaultConsumer {
   public MyConsumer(Channel channel) {
      super(channel);
   }
   @Override
   public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
      System.err.println("-----------consume message----------");
      System.err.println("consumerTag: " + consumerTag);
      System.err.println("envelope: " + envelope);
      System.err.println("properties: " + properties);
      System.err.println("body: " + new String(body));
   }
}
//使用
channel.basicConsume(queueName, true, new MyConsumer(channel));
```

![1554368759747](../images/1554368759747.png)



消费端削峰。

必须要手工签收，不能自动签收。（非常重要）

![1554368909145](../images/1554368909145.png)

![1554368929329](../images/1554368929329.png)



```java
//1 限流方式  

//一般一条一条的签收（手工）
channel.basicQos(0, 1, false);
//第一件事就是 autoAck设置为 false（自动签收）
channel.basicConsume(queueName, false, new MyConsumer(channel));


public class MyConsumer extends DefaultConsumer {
	private Channel channel ;
	public MyConsumer(Channel channel) {
		super(channel);
		this.channel = channel;
	}
	@Override
	public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
		System.err.println("-----------consume message----------");
		System.err.println("consumerTag: " + consumerTag);
		System.err.println("envelope: " + envelope);
		System.err.println("properties: " + properties);
		System.err.println("body: " + new String(body));
		//不支持批量签收，一条一条处理完后发送ACK。
		channel.basicAck(envelope.getDeliveryTag(), false);
		
	}
}

```



![1554369402631](../images/1554369402631.png)

![1554369445869](../images/1554369445869.png)

```java
if((Integer)properties.getHeaders().get("num") == 1) {
   channel.basicNack(envelope.getDeliveryTag(), false, true);//requeue参数，如果是true,表示失败后重回队列
} else {
   channel.basicAck(envelope.getDeliveryTag(), false);
}
```

![1554370137018](../images/1554370137018.png)

ack:确认应答



![1554374444846](../images/1554374444846.png)

![1554374428198](../images/1554374428198.png)

![1554374483198](../images/1554374483198.png)

![1554374530130](../images/1554374530130.png)

![1554374568353](../images/1554374568353.png)



## rabbitMQ和springAMQP整合

![1554375524600](../images/1554375524600.png)

![1554375548929](../images/1554375548929.png)



5、RabbitMQ集群架构

   	rabbitMQ集群架构模式

​	主备模式

![1554450516095](../images/1554450516095.png)

主备和主从是不一样的， 备份节点不提供读写服务。主从，从节点提供只读服务。

![1554450600596](../images/1554450600596.png)

![1554450630096](../images/1554450630096.png)

![1554450769404](../images/1554450769404.png)

![1554450781074](../images/1554450781074.png)

![1554450849532](../images/1554450849532.png)

![1554450981043](../images/1554450981043.png)

![1554450992349](../images/1554450992349.png)

![1554451049496](../images/1554451049496.png)

![1554451241313](../images/1554451241313.png)

![1554451259012](../images/1554451259012.png)

![1554451215376](../images/1554451215376.png)

![1554451315759](../images/1554451315759.png)

![1554527018232](../images/1554527018232.png)

![1554526994688](../images/1554526994688.png)

![1554526967736](../images/1554526967736.png)

![1554530810828](../images/1554530810828.png)

![1554530824512](../images/1554530824512.png)

![1554530836257](../images/1554530836257.png)



镜像队列模式：

![1554534241777](../images/1554534241777.png)

既然是集群，所以每一个节点都需要安装和启用。



## 互联网大厂的SET（单元化）架构

![1554535471794](../images/1554535471794.png)



![1554535702029](../images/1554535702029.png)

![1554535748081](../images/1554535748081.png)

![1554535766383](../images/1554535766383.png)

![1554535799122](../images/1554535799122.png)

![1554535914479](../images/1554535914479.png)



![1554535957806](../images/1554535957806.png)

![1554536181483](../images/1554536181483.png)

![1554536278397](../images/1554536278397.png)

![1554536311579](../images/1554536311579.png)

![1554536371178](../images/1554536371178.png)

![1554536433315](../images/1554536433315.png)

![1554536510259](../images/1554536510259.png)

![1554536520441](../images/1554536520441.png)

![1554536566263](../images/1554536566263.png)

![1554537593449](../images/1554537593449.png)

![1554538278573](../images/1554538278573.png)

![1554538313058](../images/1554538313058.png)

![1554538451267](../images/1554538451267.png)

![1554538491112](../images/1554538491112.png)

![1554538511956](../images/1554538511956.png)

![1554539703720](../images/1554539703720.png)

![1554539743120](../images/1554539743120.png)

![1554539818716](../images/1554539818716.png)

![1554539870840](../images/1554539870840.png)

![1554539911666](../images/1554539911666.png)

![1554540083831](../images/1554540083831.png)

![1554540096148](../images/1554540096148.png)

![1554540319089](../images/1554540319089.png)

![1554540390823](../images/1554540390823.png)

![1554540531903](../images/1554540531903.png)

![1554540726507](../images/1554540726507.png)

![1554540846719](../images/1554540846719.png)

![1554540931125](../images/1554540931125.png)

![1554540959234](../images/1554540959234.png)