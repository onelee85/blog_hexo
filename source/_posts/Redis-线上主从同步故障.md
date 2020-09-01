title: Redis 线上主从同步故障
author: James
tags:
  - 故障
  - 'redis '
categories:
  - 数据库
date: 2019-05-15 09:49:00

---

**背景**：随着公司业务增长数据量也相应的随着一起增长了。由于我司用的缓存数据库是Redis，它的keys也大幅度的增长，内存的消耗也越来越高。近日引发了一个严重的问题，最终虽然及时解决了，没有造成服务宕机以及经济损失。但通过网上搜索发现对此问题的解决方案几乎没有故而想对此次问题的排查过程做一个总结。

<!-- more -->

# 故障简述

1. 某日，运维收到了报警，线上Redis从库节点掉出MS集群， Master-Slaver失效。 运维当时对问题未经过定位，就先觉得把节点再次加入到集群。
2. 数小时候后，从节点再次掉出，运维观察到节点日志，发现从节点链接主库timeout报错信息； 推测数据量增长主从同步时间变长，调整了repl-timeout配置参数，从60秒调整为300秒。
3. 次日凌晨，突然主节点日志也发现错误信息，从节点再次掉出集群并且一直无法sync主节点的数据，从而无法提供服务。此时线上Redis集群处于单点状态，故障升级。

# 排查和解决

我和后端研发介入排查：分析了主节点log信息，发现一段bug report信息中有个一个属性 *rdb_last_bgsave_time_sec*:305， grep了报错信息，发现一次bug report 报错中这个属性的值300上下，why？ 跟运维沟通他们刚调整了从节点的repl-timeout配置从60秒调整为300秒。 似乎发现了线索，马上启用一个丛节点实例将时间调整为600秒， 结果主节点bgsave恢复正常，从节点也顺利同步了主节点落后的keys。

```verilog
# Persistence
loading:0
rdb_changes_since_last_save:8966427
rdb_bgsave_in_progress:0
rdb_last_save_time:1557748295
rdb_last_bgsave_status:err
rdb_last_bgsave_time_sec:305
rdb_current_bgsave_time_sec:-1

```

# 官方：Replication Timeouts

Redis’ replication timeout is set to 60 seconds by default (see the repl-timeout directive in your [redis.conf file](https://download.redis.io/redis-stable/redis.conf) or do a config get repl-timeout using redis-cli).

官方的解释：此参数缺省时间是60s 有些情况下是需要调长 repl-timeout 时间的。如下几个原因：

1. 硬件情况不佳：如果主节点或从节点用硬件性能不特别好，可能会导致住节点bgsave的时候写入磁盘过长的时间； 从库同步主库的和loaddata时间过长。
2. 数据量较大：当数据量过大的情况下，需要更长的时间来同步。
3. 网络带宽：当网络带宽有限或者高延迟的情况下，需要设置更长的时间。



# 实践：如何设置合适时间

1. 首先记录下bgsave完整执行一次的时间，方法是执行[BGSAVE](https://redis.io/commands/bgsave)命令并检查相关行的日志文件。
2. 接下来，计算RDB文件从主服务器复制到从服务磁盘所需要的时间。
3. 最后计算从节点从磁盘RDB加载数据所需时间。
4. 初略统计后为了安全起见可以增加10~20%的时长。

设置完毕后，需要反复多尝试几次完全同步RDB文件所需要的时间，最好是能在不同时间段不同负载下来计算。要记住，根据数据量的增长情况来定期检查时间。

[REDIS BLOG](https://redislabs.com/blog/top-redis-headaches-for-devops-replication-timeouts/#.VQnLsWSqrzY)

