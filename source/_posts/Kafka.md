title: 大数据-Kafka
author: James
tags:
  - 'kafka '
  - 分布式消息
categories:
  - 大数据
date: 2018-01-11 10:43:00
---
# 简介

Kafka的诞生是LinkedIn为了解决数据管道问题应运而生的。它设计的目的是提供一个高性能的消息系统，可以处理多种数据类型，并能够实时提供纯净且结构化的用户活动数据已经系统度量指标。

<!-- more -->

# 特点

- 高吞吐量：可以满足每秒百万级别消息的生产和消费
- 持久性：有一套晚上的消息存储机制，确保数据的高效安全持久化
- 分布式：基于分布式的扩展和容错机制，当某台机器故障失效时，生产者和消费者转移到其他机器。

# 组件

- Topic：主题，Kafka处理的消息的不同分类。
- Broker：消息代理，Kafka集群中的一个kafka服务节点称为一个broker，主要存储消息数据。存在硬盘中。每个topic都是有分区的。
- partition：Topic物理上的分组，一个topic在broker中被分为1个或者多个partition，分区在创建topic的时候指定，分区是kafka消息队列组织的最小单位，一个分区可以看作是一个FIFO（ First Input First Output的缩写，先入先出队列）的队列。如下图:

  ![](/images/kafka/topic.png)

- Message：消息，是通信的基本单位，每个消息都属于一个partition。

# 服务

1. Producer：消息和数据的生产者，向Kafka的一个topic发布消息。
2. Consumer：消息和数据的消费者，订阅topic并以拉取(pull)的方式获取消息。
3. Zookeeper：协调kafka的正常运行，保存集群的元数据信息和消费者的信息

   生产者生产消息、kafka集群、消费者获取消息架构，如下图：

![](/images/kafka/ac.png)

**工作图**

![](/images/kafka/workflow.png)

# 消息传输

数据传输的事务定义通常有以下三种级别：

-  最多一次: 消息不会被重复发送，最多被传输一次，但也有可能一次不传输。
-  最少一次: 消息不会被漏发送，最少被传输一次，但也有可能被重复传输.
-  精确的一次（Exactly once）: 不会漏传输也不会重复传输,每个消息都传输被一次而且仅仅被传输一次

# 客户端API

## Producer APIs

```java
class Producer {
    /* 将消息发送到指定分区 */
    publicvoid send(kafka.javaapi.producer.ProducerData<K,V> producerData);
    /* 批量发送一批消息 */
    public void send(java.util.List<kafka.javaapi.producer.ProducerData<K,V>> producerData);
    /* 关闭producer */
    publicvoid close();
}
```

## Consumer APIs

```java
class Consumer{
    /*订阅Topic*/
    public Set<String> subscription(Collection<String> topics);
    /*轮询拉取消息*/
    public ConsumerRecords<K, V> poll(long timeout) ;
}
```

# 消息和日志

每个文件夹里放着具体的数据文件，每个数据文件都是一系列的日志实体，每个日志实体有一个4个字节的整数N标注消息的长度，后边跟着N个字节的消息。每个消息都可以由一个64位的整数offset标注，offset标注了这条消息在发送到这个分区的消息流中的起始位置。每个日志文件的名称都是这个文件第一条日志的offset.所以第一个日志文件的名字就是00000000000.kafka.所以每相邻的两个文件名字的差就是一个数字S,S差不多就是配置文件中指定的日志文件的最大容量。

**写操作**:	消息被不断的追加到最后一个日志的末尾，当日志的大小达到一个指定的值时就会产生一个新的文件。对于写操作有两个参数，一个规定了消息的数量达到这个值时必须将数据刷新到硬盘上，另外一个规定了刷新到硬盘的时间间隔，这对数据的持久性是个保证，在系统崩溃的时候只会丢失一定数量的消息或者一个时间段的消息。

**读操作**:	读操作需要两个参数：一个64位的offset和一个S字节的最大读取量。S通常比单个消息的大小要大，但在一些个别消息比较大的情况下，S会小于单个消息的大小。这种情况下读操作会不断重试，每次重试都会将读取量加倍，直到读取到一个完整的消息。可以配置单个消息的最大值，这样服务器就会拒绝大小超过这个值的消息。也可以给客户端指定一个尝试读取的最大上限，避免为了读到一个完整的消息而无限次的重试。

**可靠性**:	日志文件有一个可配置的参数M，缓存超过这个数量的消息将被强行刷新到硬盘。一个日志矫正线程将循环检查最新的日志文件中的消息确认每个消息都是合法的。合法的标准为：所有文件的大小的和最大的offset小于日志文件的大小，并且消息的CRC32校验码与存储在消息实体中的校验码一致。如果在某个offset发现不合法的消息，从这个offset到下一个合法的offset之间的内容将被移除。

# 其他消息系统对比

![](/images/kafka/compare.jpg)

