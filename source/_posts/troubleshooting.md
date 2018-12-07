title: 在线故障排查总结
author: James
tags:
  - 故障
categories:
  - 运维
date: 2018-11-26 10:22:00

---

上周发完版本后，运维突然发现线上接口请求出现大量延时。后端和运维从排查出问题到修复一共耗费了5~6天时间（周年庆活动暂停，渠道暂停投放，给公司造成不少损失，暂时估计70~100万流水）。虽然自己已经不负责后端了，但之前负责过几年时间，故而也参与了此次排查。 这里对整个过程做个总结以及后续优化方案。

<!-- more  -->

# 问题总结：

1. **排查耗时过长**： 整个排查耗时5~6天时间（导致活动停止，渠道暂停，造成不少损失）。
   1. 在排查方向已经明确的情况下，对其他人执行的结果没有仔细确认。（我的问题）
   2. 慢查询日志的统计和收集不够完善，排查难度较大。
   3. 后端对mongodb的索引机制不够熟悉，需要临时学习和研究。
2. **引发原因**：
   1. 已经监控到慢查询没引起重视，发现后未及时优化（16号运维就发现有慢查询日志，当时已经发给到研发）。
   2. 业务代码查询条件的变更，导致索引未命中查询变慢。
   3. 接口请求响应时间变慢，oneapm 接口监控平台在平时没有定期检查习惯。

## 解决方案：

1. 加强对数据库的学习包括运维和研发人员。考虑：请专业DBA顾问定期培训和规划。
2. 数据库监控（慢查询）日常化，明确职责。 发现问题后及时指定相关人评估和处理。
3. 测试部门增加对接口和数据压测，配合后端进行数据库规划和扩容。
4. 整理好数据库故障排查手册。发生问题能对照逐步实施。



# 在线故障应对手册

整理了自己几年来针对在线故障的总结，可作为日后checklist:

## 故障发生

- 接口缓慢，无响应。
- 影响用户体验，导致收入降低。
- 不能停机。

## 召集相关人员

业务负责人、技术负责人、运维人员、核心研发人员、架构师以及相关运营人员。

## 分析思路

1. 系统最近是否进行了发布和上线工作？
2. 运营或者市场是否有新的活动?
3. 网络是否有流量的波动?
4. 最近的业务量是否上升?
5. 运营人员是否在后台做了某些变动?
6. 依赖的基础平台是否也进行了发布上线?
7. 依赖的第三方系统进行了变动?

## 可能原因

1. 代码bug：逻辑不严谨、某些资源未释放。
2. 代码性能：查库循环调用，未使用批量读取。嵌套循环。
3. 内存泄露：查询大量数据，引用未释放。
4. 异常流量/攻击: DDOS。
5. 业务量提升：容量预估失误，无法承受大量请求。
6. 数据库问题：索引、数据库数据量。
7. 系统和硬件：CPU、内存、IO指标异常、硬件损坏。

## 诊断

### 系统级：

![](/images/pro_trouble/sys.png)

### java应用

![](/images/pro_trouble/java.png)

详细可见之前的博文[JVM排查小记](http://jiaoblog.cn/Java/JVM%E6%8E%92%E6%9F%A5%E5%B0%8F%E8%AE%B0// "JVM排查小记")。 另外可以参考阿里的[java问题排查工具单 ](https://yq.aliyun.com/articles/69520?utm_content=m_10360"java问题排查工具单 ")
Alibaba Java[诊断利器Arthas](https://yq.aliyun.com/articles/69520?utm_content=m_10360"java问题排查工具单 ")
 https://alibaba.github.io/arthas/

### 数据库

#### **MongoDB**

**mongosniff**：此功能可以从底层监控到底有哪些命令发送给mongoDB去执行.

```shell
 ./mongosniff --source net lo 
```
它是实时动态监视的,需要打开另一个客户端进行命令操作.可以讲这些数据输出到一个日志文件中。
**mongostat**：快速的查看某组运行中的MongoDB实例的统计信息。
**profile**：Profiler是一种慢查询日志功能,可以作为我们优化数据库的依据。
**db.stats**：命令查看特定数据库的详细运行状态,更细粒度地分析故障。
**explain** ：观察查询器如何使用索引来加快检索,同时可以针对性优化索引。

#### Redis

**计算延迟时间**：可以用	Redis-cli工具加--latency参数运行，如:

```shell
Redis-cli --latency -h 127.0.0.1 -p 6379
```

**使用slowlog查出引发延迟的慢命令：**

```shell
config set slowlog-log-slower-than 5000
```

**监控客户端的连接：** 客户端创建连接数的数量可能超出预期的数量，也可能是客户端端没有有效的释放连接。在Redis-cli工具中输入info clients可以查看到当前实例的所有客户端连接信息。

**内存管理：**较少的内存会引起Redis延迟时间增加。


