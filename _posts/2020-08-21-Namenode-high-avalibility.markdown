---
layout: HDFS NameNode
title: hdfs-namenode
date: 2020-08-21 19:20:23 +0900
category: wenjian
---

# HDFS NameNode 

## 组件

```
<!--[架构](https://github.com/aerfalwl/aerfalwl.github.io/blob/master/image/2020/08/20200821-HDFS-Arc.png)->
```





### NameNode

1. Activate NameNode
   - 主NameNode，该节点对外提供读写服务。
2. Standby NameNode
   - 备NameNode，该节点在主节点故障时将切换至主节点身份。

###  FailoverController 

主备切换控制器，作为独立的进程运行，借助Zookeeper实现具体功能。

1. 主FailoverController
   - 检测NameNode的健康状况。
   - NameNode故障时借助Zookeeper实现自动的主备选举和切换。

### Zookeeper

主要为主备切换控制器提供主备选举支持。

### 共享存储系统

1. 共享存储是实现NameNode高可用最关键的部分
2. 该系统保存了HDFS的所有元数据信息
3. 在主备切换时，备节点必须要确认元数据完全同步之后，才能继续对外提供服务。

### DataNode

1. DataNode会同时向主NameNode和备NameNode上报数据块的位置信息。
2. 主备NameNode需要共享**HDFS的数据块**和**DataNode之间的映射关系**。



## 主备切换实现

主备切换主要涉及到的组件有三个，ZKFailoverController，HealthMonitor和ActiveStandbyElector。



### ZKFailoverController

1. 在NameNode启动时，也作为一个独立的进程启动，同时创建HealthMonitor和ActiveStandbyElector两个主要的内部控件。
2. 在启动之后，也会向HealthMonitor和ActiveStandbyElector注册相应的回调方法。

### HealthMonitor

1. 负责检测NameNode的健康状态。
2. 若发生异常，回调ZKFailoverController的相关方法进行自动的主备选举。

### ActiveStandbyElector

1. 主要负责完成自动的主备选举。
2. 内部封装了Zookeeper的处理逻辑。
3. 在Zookeeper主备选举完成时，会回调ZKFailoverController的相应方法来进行NameNode的主备状态切换。



## NameNode主备切换的流程

1. HealthMonitor会启动内部线程定时调用对应**NameNode的HAServiceProtocal RPC方法**，对其健康状态进行检测。
2. 若检测到异常，则回调FailoverController的响应方法进行处理。
3. 如果ZKFailoverController判断需要进行主备切换，会首先使用ActiveStandbyElector来进行自动的主备选举。
4. ActiveStandbyElector与Zookeeper交互进行自动的主备选举。
5. ActiveStandbyElector在主备选举完成后，回调ZKFailoverController的响应方法来通知当前的NameNode成为主NameNode或者备NameNode。
6. ZKFailoverController调用的NameNode的RPC方法将NameNode转换为Active状态或者Standby状态。

### Health Monitor实现分析

### ActiveStandbyElector实现分析

1. 主要利用Zookeeper的写一致性和临时节点机制
2. Zookeeper能保证多个节点写一个目录时，只有一个节点写成功，因此写成功的节点就是下一届的主NameNode节点。
3. 不管节点是否为主namenode节点，ActiveStandbyElector随后都会向zookeeper注册一个Watcher来监听这个节点的状态变化事件，并主要关注其NodeDeleted事件。
4. 如果主NameNode上对应的HealthMonitor检测到Namenode的状态异常时，ZKFC会主动删除Lock文件，这样别的节点就监听到了NodeDeleted事件，收到这个事件之后，再次尝试创建Lock节点的流程，创建成功的节点成功成为下一任的主节点。如果主NameNode宕机，那么根据zookeeper的临时特性，Lock节点文件会自动删除，从而触发下一次的主备切换。

### 防止脑裂

### NameNode元数据存储

1. 内存中的文件系统镜像用于NameNode向客户端提供读服务，而EditLog仅仅是在数据恢复的时候起作用。
2. Eitdlog中每个操作称为一个事务，每个事务有一个事务Id.
3. Editlog有很多段，每一个段称为一个Segment, 正在写入的Editlog会处于inprogress状态。
4. NameNode会定期对内存中的文件系统镜像进行checkpoint操作，在磁盘上生成FSImage文件。

### 基于QJM的数据同步机制分析

1. Active NameNode提交EditLog到QournalNode集群
2. 若大多数JournalNode写成功，则认为本次log同步成功，否则，NameNode认为这次提交Editlog失败。提交EditLog失败会导致Active NameNode关闭JournalSet之后退出进程，留待Standby状态的NameNode接管之后进行数据恢复。
3. 因为In-process的Editlog并没有被同步至JournalNode集群上，因此，Standby状态的NameNode内存镜像是落后于Active NameNode的，所以Standby NameNode在转换为Active NameNode时需要将落后的Editlog补上来。

## NameNode在进行状态转换时对共享存储的处理

[原文地址](https://developer.ibm.com/zh/articles/os-cn-hadoop-name-node/)





