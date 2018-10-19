title: 大数据-HBase
author: James
tags:
  - HBase
categories:
  - 大数据
date: 2018-07-16 13:39:00
---
# 简介

HBase系统是一个构建在HDFS分布式的、持久的、强一致性的存储系统。基于Google BigTable模型开发的，典型的key/value系统。与hadoop一样，Hbase目标主要依靠横向扩展，通过不断增加廉价的商用服务器，来增加计算和存储能力。

![](/images/hbase/arch.png)

## 架构

HBase本身从功能上可以分为三块：Zookeeper群、Master群和RegionServer群。

- Zookeeper群：HBase集群中不可缺少的重要部分，主要用于存储Master地址、协调Master和RegionServer等上下线、存储临时数据等等。
- Master群：Master主要是做一些管理操作，如：region的分配，手动管理操作下发等等，一般数据的读写操作并不需要经过Master集群，所以Master一般不需要很高的配置即可。
- RegionServer群：RegionServer群是真正数据存储的地方，每个RegionServer由若干个region组成，而一个region维护了一定区间rowkey值的数据

![](/images/hbase/a2.jpg)

# 基本概念

- RowKey：是Byte array，是表中每条记录的“主键”，方便快速查找，Rowkey的设计非常重要。
- Column Family：列族，拥有一个名称(string)，包含一个或者多个相关列。
- Column：属于某一个columnfamily，familyName:columnName，每条记录可动态添加。
- Version Number：类型为Long，默认值是系统时间戳，可由用户自定义。
- Value(Cell)：Byte array。
- HLog： HLog(WAL log)：WAL意为write ahead log（预写日志），用来做灾难恢复使用，HLog记录数据的变更，包括序列号和实际数据，所以一旦region server 宕机，就可以从log中回滚还没有持久化的数据。

![](/images/hbase/fundation.png)

# 客户端读写的流程

1. 客户端首先从zookeeper中得到META table的位置，根据META table的存储位置得到具体的RegionServer是哪台.
2. 询问具体的RegionServer

## **写流程**
1. 首先写入WAL(write ahead log)日志，以防crash。
2. 紧接着写入Memstore，即写缓存。由于是内存写入，速度较快。
3. 立马返回客户端表示写入完毕。
4. 当Memstore满时，从Memstore刷新到HFile，磁盘的顺序写速度非常快，并记录下最后一次最高的sequence号。这样系统能知道哪些记录已经持久化，哪些没有。

## 读流程
1. 首先到读缓存BlockCache中查找可能被缓存的数据
2. 如果未找到，到写缓存查找已提交但是未落HFile的数据
3. 如果还未找到， 到HFile中继续查找数据

# 恢复流程

1. HFile本身有2个备份，而且有专门的HDFS来管理其下的文件。因此对HFile来说并不需要恢复。
2. Hmaster重置region到新的regionServer。
3. 之前在MemStore中丢失的数据，通过WAL分裂先将WAL按照region切分。切分的原因是WAL并不区分region，而是所有region的log都写入同一个WAL。
4. 根据WAL回放并恢复数据。回放的过程实际上先进MemStore，再flush到HFile。

# 场景

1. 写密集型应用，每天写入量巨大，而相对读数量较小的应用
2. 不需要复杂查询条件来查询数据的应用
3. 不需要关联的场景，HBase为NoSQL无法支持join
4. 可靠性要求高
5. 数据量较大，而且增长量无法预估的应用

# 应用

- **用户画像系统**：动态列，稀疏列的特性。用于描述用户特征的维度数是不定的且可能会动态增长的(比如爱好，性别，住址等);不是每个特征维度都会有数据。
- **消息系统**：强一致性，良好的读性能
- **feed流系统存储**：账号关系数据（比如关注列表）和Feed消息内容