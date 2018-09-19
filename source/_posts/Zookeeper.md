title: 大数据-Zookeeper
author: James
tags:
  - zookeeper
  - 分布式系统
categories:
  - 大数据
date: 2018-03-19 10:15:00
---
# 概要

Zookeeper是一个分布式服务协调框架，实现同步服务，配置维护和命名服务等分布式应用.，这些功能都是分布式系统中非常底层且必不可少的基本功能，但是如果需要程序员自己实现这些功能而且要达到高吞吐、低延迟同时还要保持一致性和可用性，实际上非常困难。因此zookeeper提供了这些功能，开发者在zookeeper之上构建自己的各种分布式系统。

<!-- more -->

# 架构简图

![](/images/zookeeper/zk-framework.png)



# 节点

Zookeeper提供一个多层级的节点命名空间（节点称为znode），每个节点都用一个以斜杠（/）分隔的路径表示，而且每个节点都有父节点（根节点除外），非常类似于文件系统。

下图所示:

![](/images/zookeeper/zookeeper-tree.png)

znode节点可以含有数据，也可以没有。如果一个znode节点包含数据的话，那么数据是以字节数组的形式来存储。字节数组的具体格式依赖于应用本身的实现，Zookeeper不直接提供解析的支持，应用可以使用如Protocol
Buffer、Thrift、Avro或MessagePack等序列化包来处理保存于znode节点的数据。

## znode类型

1. **PERSISTENT**：持久化ZNode节点，一旦创建这个ZNode点存储的数据不会主动消失，除非是客户端主动的delete。
2. **EPHEMERAL**：临时ZNode节点，Client连接到Zookeeper Service的时候会建立一个Session，之后用这个Zookeeper连接实例创建该类型的znode，一旦Client关闭了Zookeeper的连接，服务器就会清除Session，然后这个Session建立的ZNode节点都会从命名空间消失。总结就是，这个类型的znode的生命周期是和Client建立的连接一样的。
3. **PERSISTENT_SEQUENTIAL**：顺序自动编号的ZNode节点，这种znoe节点会根据当前已近存在的ZNode节点编号自动加 1，而且不会随Session断开而消失。
4. **EPEMERAL_SEQUENTIAL**：临时自动编号节点，ZNode节点编号会自动增加，但是会随Session消失而消失。


## 监视与通知

Zookeeper提供了基于通知（notification）的机制：客户端向Zookeeper对某个znode设置监视点（watch），当该znode发生变更时会触发一个通知。

# Zookeeper和CAP的关系

作为一个分布式系统，分区容错性是一个必须要考虑的关键点。一个分布式系统一旦丧失了分区容错性，也就表示放弃了扩展性。因为在分布式系统中，网络故障是经常出现的，一旦出现在这种问题就会导致整个系统不可用是绝对不能容忍的。所以，大部分分布式系统都会在保证分区容错性的前提下在一致性和可用性之间做权衡。

ZooKeeper是个CP（一致性+分区容错性）的，即任何时刻对ZooKeeper的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性；但是它不能保证每次服务请求的可用性。也就是在极端环境下，ZooKeeper可能会丢弃一些请求，消费者程序需要重新请求才能获得结果。

ZooKeeper是分布式协调服务，它的职责是保证数据在其管辖下的所有服务之间保持同步、一致；所以就不难理解为什么ZooKeeper被设计成CP而不是AP特性的了。而且，  作为ZooKeeper的核心实现算法**Zab**，就是解决了分布式系统下数据如何在多个服务之间保持同步问题的。

## Zab协议

**Zab**（ZooKeeper Atomic Broadcast）原子消息广播协议作为数据一致性的核心算法，Zab协议是专为Zookeeper设计的支持崩溃恢复原子消息广播算法。

Zab协议核心如下：

所有的事务请求必须一个全局唯一的服务器（Leader）来协调处理，集群其余的服务器称为follower服务器。Leader服务器负责将一个客户端请求转化为事务提议（Proposal），并将该proposal分发给集群所有的follower服务器。之后Leader服务器需要等待所有的follower服务器的反馈，一旦超过了半数的follower服务器进行了正确反馈后，那么Leader服务器就会再次向所有的follower服务器分发commit消息，要求其将前一个proposal进行提交。

# 实践

## 排它锁

写锁或独占锁，若事务T1对数据对象O1加上了排它锁，那么在整个加锁期间，只允许事务T1对O1进行读取和更新操作，其他任何事务都不能再对这个数据对象进行任何类型的操作，直到T1释放了排它锁。

## 共享锁

若事务T1对数据对象O1加上共享锁，那么当前事务只能对O1进行读取操作，其他事务也只能对这个数据对象加共享锁，直到该数据对象上的所有共享锁都被释放。

简单伪代码:

```java
private ZkClient zkClient;
 
private String path;
private final String LOCK;

//上锁
public boolean lock() throws Exception {
    if (zkClient.exists(path))
        return true;
    else//创建临时节点
        return zkClient.create(path, LOCK.getBytes(), CreateMode.EPHEMERAL) == null ?true : false;
}
//解锁 
public boolean unlock() throws Exception {
    return zkClient.delete(path);
}
 
public boolean islock() throws Exception {
    return zkClient.exists(path);
}
```

