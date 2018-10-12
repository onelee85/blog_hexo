title: 大数据-flume
author: James
tags:
  - flume
categories:
  - 大数据
date: 2018-06-09 11:31:00
---
# Flume是什么？

Flume是一个海量日志采集、聚合和传输系统， 他是基于流数据的灵活框架。其具有健壮的可靠性，容错性及故障转移和恢复机制。

<!-- more -->

# 核心概念

可以把flume想象成一个水池，两头有一个进水口和一个出水口。进出水口可以配置多个管子。

进水口=Source、出水口=Sink、水池=Channel， 流动的水=event，   Source+Channel+Sink=Agent

1. Agent：Flume以agent为最小的独立运行单位。一个agent就是一个JVM。它是一个完整的数据收集工具。
2. Source：源数据收集组件，采集数据用途。
3. Channel：管道，连接源数据Source和Sink，用户传输数据。
4. Sink：发送数据，采集来自Channel中的数据。
5. Event： 一个数据单元，消息头和消息体组成。

![](/images/flume/data_flow.png)


## source

Client端操作消费数据的来源，Flume 支持 Avro，log4j，syslog 和 http  post(body为json格式)。可以让应用程序同已有的Source直接打交道，如AvroSource，SyslogTcpSource。也可以  写一个 Source，以 IPC 或 RPC 的方式接入自己的应用，Avro和 Thrift 都可以(分别有  NettyAvroRpcClient 和 ThriftRpcClient 实现了 RpcClient接口)，其中 Avro 是默认的 RPC  协议。具体代码级别的 Client 端数据接入，可以参考官方手册。

对现有程序改动最小的使用方式是使用是直接读取程序原来记录的日志文件，基本可以实现无缝接入，不需要对现有程序进行任何改动。  对于直接读取文件 Source,有两种方式：

- ExecSource: 以运行 Linux 命令的方式，持续的输出最新的数据，如 `tail -F 文件名` 指令，在这种方式下，取的文件名必须是指定的。 ExecSource 可以实现对日志的实时收集，但是存在Flume不运行或者指令执行出错时，将无法收集到日志数据，无法保证日志数据的完整性。
- SpoolSource: 监测配置的目录下新增的文件，并将文件中的数据读取出来。需要注意两点：拷贝到 spool 目录下的文件不可以再打开编辑；spool 目录下不可包含相应的子目录。

SpoolSource 虽然无法实现实时的收集数据，但是可以使用以分钟的方式分割文件，趋近于实时。

如果应用无法实现以分钟切割日志文件的话， 可以两种收集方式结合使用。 在实际使用的过程中，可以结合 log4j 使用，使用 log4j的时候，将 log4j 的文件分割机制设为1分钟一次，将文件拷贝到spool的监控目录。

log4j 有一个 TimeRolling 的插件，可以把 log4j 分割文件到 spool 目录。基本实现了实时的监控。Flume 在传完文件之后，将会修改文件的后缀，变为 .COMPLETED（后缀也可以在配置文件中灵活指定）。

Flume Source 支持的类型：

| Source类型                 | 说明                                                    |                                       |
| -------------------------- | ------------------------------------------------------- | ------------------------------------- |
| Avro Source                | 支持Avro协议（实际上是Avro RPC），内置支持              |                                       |
| Thrift Source              | 支持Thrift协议，内置支持                                |                                       |
|                            | Exec Source                                             | 基于Unix的command在标准输出上生产数据 |
| JMS Source                 | 从JMS系统（消息、主题）中读取数据，ActiveMQ已经测试过   |                                       |
| Spooling Directory Source  | 监控指定目录内数据变更                                  |                                       |
| Twitter 1% firehose Source | 通过API持续下载Twitter数据，试验性质                    |                                       |
| Netcat Source              | 监控某个端口，将流经端口的每一个文本行数据作为Event输入 |                                       |
| Sequence Generator Source  | 序列生成器数据源，生产序列数据                          |                                       |
| Syslog Sources             | 读取syslog数据，产生Event，支持UDP和TCP两种协议         |                                       |
| HTTP Source                | 基于HTTP POST或GET方式的数据源，支持JSON、BLOB表示形式  |                                       |
| Legacy Sources             | 兼容老的Flume OG中Source（0.9.x版本）                   |                                       |

## Channel

当前有几个 channel 可供选择，分别是 Memory Channel, JDBC Channel , File Channel，Psuedo Transaction Channel。比较常见的是前三种 channel。

- MemoryChannel 可以实现高速的吞吐，但是无法保证数据的完整性。
- MemoryRecoverChannel 在官方文档的建议上已经建义使用FileChannel来替换。
- FileChannel保证数据的完整性与一致性。在具体配置FileChannel时，建议FileChannel设置的目录和程序日志文件保存的目录设成不同的磁盘，以便提高效率。

File Channel 是一个持久化的隧道（channel），它持久化所有的事件，并将其存储到磁盘中。因此，即使 Java  虚拟机当掉，或者操作系统崩溃或重启，再或者事件没有在管道中成功地传递到下一个代理（agent），这一切都不会造成数据丢失。Memory  Channel 是一个不稳定的隧道，其原因是由于它在内存中存储所有事件。如果 java  进程死掉，任何存储在内存的事件将会丢失。另外，内存的空间收到 RAM大小的限制,而 File Channel  这方面是它的优势，只要磁盘空间足够，它就可以将所有事件数据存储到磁盘上。

Flume Channel 支持的类型：

| Channel类型                | 说明                                                         |
| -------------------------- | ------------------------------------------------------------ |
| Memory Channel             | Event数据存储在内存中                                        |
| JDBC Channel               | Event数据存储在持久化存储中，当前Flume Channel内置支持Derby  |
| File Channel               | Event数据存储在磁盘文件中                                    |
| Spillable Memory Channel   | Event数据存储在内存中和磁盘上，当内存队列满了，会持久化到磁盘文件（当前试验性的，不建议生产环境使用） |
| Pseudo Transaction Channel | 测试用途                                                     |
| Custom Channel             | 自定义Channel实现                                            |

##  sink

Sink在设置存储数据时，可以向文件系统、数据库、hadoop存数据，在日志数据较少时，可以将数据存储在文件系中，并且设定一定的时间间隔保存数据。在日志数据较多时，可以将相应的日志数据存储到Hadoop中，便于日后进行相应的数据分析。

Flume Sink支持的类型

| Sink类型            | 说明                                                |
| ------------------- | --------------------------------------------------- |
| HDFS Sink           | 数据写入HDFS                                        |
| Logger Sink         | 数据写入日志文件                                    |
| Avro Sink           | 数据被转换成Avro Event，然后发送到配置的RPC端口上   |
| Thrift Sink         | 数据被转换成Thrift Event，然后发送到配置的RPC端口上 |
| IRC Sink            | 数据在IRC上进行回放                                 |
| File Roll Sink      | 存储数据到本地文件系统                              |
| Null Sink           | 丢弃到所有数据                                      |
| HBase Sink          | 数据写入HBase数据库                                 |
| Morphline Solr Sink | 数据发送到Solr搜索服务器（集群）                    |
| ElasticSearch Sink  | 数据发送到Elastic Search搜索服务器（集群）          |
| Kite Dataset Sink   | 写数据到Kite Dataset，试验性质的                    |
| Custom Sink         | 自定义Sink实现                                      |

# 可靠性

Flume 的核心是把数据从数据源收集过来，再送到目的地。为了保证输送一定成功，在送到目的地之前，会先缓存数据，待数据真正到达目的地后，删除自己缓存的数据。

Flume 使用事务性的方式保证传送Event整个过程的可靠性。Sink 必须在 Event 被存入 Channel  后，或者，已经被传达到下一站agent里，又或者，已经被存入外部数据目的地之后，才能把 Event 从 Channel 中 remove  掉。这样数据流里的 event 无论是在一个 agent 里还是多个 agent 之间流转，都能保证可靠，因为以上的事务保证了 event  会被成功存储起来。而 Channel 的多种实现在可恢复性上有不同的保证。也保证了 event 不同程度的可靠性。比如 Flume  支持在本地保存一份文件 channel 作为备份，而memory channel 将 event 存在内存 queue  里，速度快，但丢失的话无法恢复。



# 实践

我们的需求是将Java 应用的log信息进行收集，达到日志采集的目的，agent目前主要有flume、Logstash，技术选型详情在此就不表了，最终选择的flume。