---
title: rocketmq learning notes
date: 2021-02-01 16:14:12
categories: distribute file system
tags:
	- message queue
	- learning notes
---
# 《RokcetMQ实战与原理解析》学习笔记

阿里消息中间件有很长的历史，其模型也经过了相关的演变。

1. Notify主要使用推(Push)模型，解决了事务消息。
2. MetaQ主要使用拉(Pull)模型，解决了顺序消息和海量堆积的问题。
3. RocketMQ基于**长轮询的拉取方式**，兼有两者的优点。

> Push方式是Server端接收到消息后，主动把消息推送给Client端，实时性高。对于一个提供队列服务的Server端来说，用Push方式主动推动有很多弊端：首先是加大了Server端的工作量，进而影响Server的性能；其次，Client的处理能力各不相同，Client的状态不受Server的控制，如果Client不能及时处理Server推送过来的消息，会造成潜在问题。
>
> 
>
> Pull方式是Client端循环地从Server端拉取消息，主动权在Client手里，自己拉取到一定消息后，处理妥当载接着取。Pull方式的问题是循环拉取消息的间隔不好设定，间隔太短就处在一个忙等的状态，浪费资源；每个Pull的时间间隔太长，Server端消息到来时，有可能没被及时处理。
>
>
> “长轮询”方式通过Client端和Server端的配合，达到了即拥有Pull的优点，又能达到保证实时性的目的。长轮询的主动权还是掌握在Consumer手里，Broker即使有大量消息积压，也不会主动推送消息给Consumer。但是它的局限性在于，Hold住Consumer请求的时候需要占用资源，它适合用在消息队列这种客户端连接数可控的场景中。

## Broker

消息顺序存储，由CousumerQueue和CommitLog配合完成。CommitLog是真正的物理存储文件。ConsumerQueue是消息的逻辑队列，类似数据库的索引文件，存储的是指向物理存储的地址。

### 数据结构简介

1. **CommitLog**：物理存储文件，存储真实数据的文件。默认存储文件大小为1G，当文件超过指定大小时，新建文件，且文件名标示该文件第一个消息的全局Offset。该数据结构与ClickQ中的FileHandle类似。
2. **ConsumerQueue**: 定长结构，每个记录20个字节: 8字节（CommitLog Offset）+ 4字节（Size）+ 8字节(Message Tag HashCode)。即消费消息的时候，需要读2次，先读ConsumerQueue得到Offset，再读CommitLog得到消息内容。

### 对比

| 对比项        | 自研MQ       | RocketMQ |
| ------------- | ---------------- | -------- |
| 数据存储      | 顺序存储，KV存储 | 顺序存储 |
|  数据读取 | 在顺序存储模式下，顺序读取文件；在KV存储模式下，若数据在内存中，查找速度很快，若数据在文件里，则是加上Cache的随机读取。 |随机读，利用操作系统的PageCache机制，可以批量从磁盘读取文件，作为cache存在内存中，加速后续的读取速度。|
| 读写请求处理 | 读写分离，即有专门的读端口和写端口来处理不同类型的请求。读写服务提供长连接和短连接两种方式进行数据的传输；写服务有多个线程以提高系统并发性。 | 有专门的读写队列，正常情况下，应设置读写队列个数相等。在需要扩容或者缩容的时候，可以调整读写队列个数做到快速扩容或者缩容。 |
| 物理节点与Broker的关系 | 一台节点上可以部署一个Master类型的Broker节点和多个Slave类型的Broker节点。（一台节点也可以部署多个Master节点，但是没必要，部署多个Master节点时，注意设置不同的端口号以防止端口冲突） | 一台节点上可以部署一个Master类型的Broker节点和多个Slave类型的Broker节点。（一台节点也可以部署多个Master节点，但是没必要，部署多个Master节点时，注意设置不同的端口号以防止端口冲突） |
| 高可用 | **待完善** | 通过Master与Slave节点之间的数据同步，从而提高系统的高可用特性。其中元数据信息是采用基于Netty的command方式来同步消息，而commitlog信息的同步方式，是直接基于Java NIO的tcp通信实现的。 |
| 文件删除策略 | **待完善** | 消息在磁盘上保存的时间有限制，超时自动删除。 |
| features |                  | 1. 有专门的tools（形式类似于cmd和console）来维护broker的topic元数据信息，以根据需求进行相关的信息调整。<br />2. 根据时间查询消息；<br />3. 根据消息id查询消息； |



## Nameserver

1. 没有采用zookeeper，因为zookeeper功能很强大，包括自动Master选举等，而RocketMQ的架构设计不需要进行Master选举，用不到相关复杂的功能，只需要一个轻量级的元数据服务器就足够了。
2. Nameserver的相关信息保存在内存中，并不需要进行持久化操作。
3. 所有的角色会定期向Namserver发送心跳信息，nameserver会有专门的线程来检测是否有节点不可用，若一段时间都没收到某个节点的心跳便认为该节点已经处于不可用状态。

## Consumer

### 消费者类型

1. **DefaultMQPushConsumer**：由系统控制读取操作，收到消息后自动调用传入的处理方法来处理。
2. **DefaultMQPullConsumer**: 由使用者自己控制消息的拉取。

Push和Pull模式都是采用消费端主动拉取的方式，即consumer轮询从broker拉取消息。

**区别：**

- Push方式里，consumer把轮询过程封装了，并注册MessageListener监听器，取到消息后，唤醒MessageListener的consumeMessage()来消费，对用户而言，感觉消息是被推送过来的。
- Pull方式里，取消息的过程需要用户自己写，首先通过打算消费的Topic拿到MessageQueue的集合，遍历MessageQueue集合，然后针对每个MessageQueue批量取消息，一次取完后，记录该队列下一次要取的开始offset，直到取完了，再换另一个MessageQueue

#### **DefaultMQPushConsumer**

1. 流量控制：PushConsumer会判断获取但是还未处理的消息个数、消息总大小、offset的跨度，任何一个值超过设定的大小就隔一段时间再拉取消息，从而达到流量控制的目的。

#### **DefaultMQPullConsumer**

工作流程是：做个读取某个Topic下所有Message Queue的内容，读完一遍后退出，主要处理额外的三件事情。

1. 获取Message Queue并遍历；
2. 维护Offsetstore；
3. 根据不同的消息状态做不同的处理。

### 消息模式

1. **Clustering模式**：在一个ConsumerGroup中的每个Consumer只消费订阅消息的一部分内容，同一个ConsumerGroup里所有的Consumer消费的内容结合起来才是订阅Topic内容的整体，从而达到负载均衡的目的。
2. **BroadCasting模式**：在一个ConsumerGroup中的每个Consumer都能消费Topic的全部信息，也就是一个信息会被分发多次，被多个Consumer消费。



消息重复消费的两种方法：第一种方法是保证消费逻辑的幂等性，另一种是维护一个已消费消息的记录，消费前查询这个消息是否被消费过。这两种方式都要使用者自己实现。

消息优先级也需用户自己编码实现优先级分配。

## Producer

消息发送需要五个步骤：

1. 设置GroupName；
2. 设置InstanceName；当一个Jvm需要启动多个Producer的时候，通过设置不同的InstanceName来区分，不设置的话系统使用默认名称“DEFAULT”;
3. 设置消息失败重试步骤；
4. 设置Nameserver地址；
5. 组装消息并发送；

## Offset存储

两种存储方式

1. **LocalFileOffsetStore**，即用户自己将offset存储在本地；场景：PushConsumer的BroadCasting模式；PullConsumer
2. **RemoteBrokerOffset**，即Broker端存储offset；场景：PushConsumer在Clustering模式；



