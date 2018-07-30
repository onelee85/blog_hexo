title: JVM排查小记
author: James
tags:
  - JVM
categories:
  - Java
date: 2016-07-25 10:46:00
---
在生产环境中，我们经常遇到突发问题，无法用代码调试或者其他可视化工具去立马查看当前的运行状态和拿到错误信息，此时，借助Java自带的命令行工具以及相关dump分析工具以及一些小技巧，可以大大提升我们排查问题的效率。

此篇博客主要是总结相关线上调试的一些经验
<!-- more -->

# CPU 突然飚高

思路：首先找出 CPU 飚高的 Java 进程，可能服务器会有多个 JVM 进程。然后找到那个进程中的 “问题线程”，最后根据线程堆栈信息找到问题代码。最后对代码进行排查。

1. 通过 top 命令找到 CPU 消耗最高的进程，记录进程 ID。

   ![](/images/jvm_trouble/top.png)

2. 再次通过 top -Hp [进程 ID] 找到 CPU 消耗最高的线程 ID.

   ![](/images/jvm_trouble/thread.png)

3. 通过 JDK 提供的 jstack 工具 dump 线程堆栈信息到指定文件中。具体命令：jstack -l [进程 ID] >jstack.log。
4. 由于刚刚的线程 ID 是十进制的，而堆栈信息中的线程 ID 是16进制的，因此我们需要将10进制的转换成16进制的，并用这个线程 ID 在堆栈中查找。使用 printf “%x\n” [十进制数字] ，可以将10进制转换成16进制。
   通过刚刚转换的16进制数字从堆栈信息里找到对应的线程堆栈。就可以从该堆栈中看出端倪。

# 内存问题
内存管理一般容易发生两种情况: 一种是内存溢出了;一种是GC频繁

1. java.lang.OutOfMemoryError: PermGen space ，说明是Java虚拟机对永久代Perm内存设置不够。
2. java.lang.OutOfMemoryError: Java heap space，说明Java虚拟机的堆内存不足。
   1. 可以通过参数-Xms、-Xmx来调整。 
   2. 代码中创建了大量大对象，并且长时间不能被垃圾收集器收集（存在被引用）。 

## 排查方法:

1. top命令：Linux命令。可以查看实时的内存使用情况。   
2. jmap -histo:live [pid]，然后分析具体的对象数目和占用内存大小，从而定位代码。 
3. jmap -dump:live,format=b,file=xxx.dump [pid]，然后利用可视化工具(MAT，Jprofile，jvisualvm 等分析是否存在内存泄漏。 
4. 提前对Java程序加上参数打印dump文件 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./`
5. GC 的优化有2个维度，一是频率，二是时长。 首先看频率，如果 YGC 超过5秒一次，甚至更长，说明系统内存过大，应该缩小容量，如果频率很高，说明 Eden 区过小，可以将 Eden 区增大，但整个新生代的容量应该在堆的 30% - 40%之间，eden，from 和 to 的比例应该在 8：1：1左右，这个比例可根据对象晋升的大小进行调整。

# 常用命令

- **jps**: 显示系统内所有的JVM进程 

- **jstat**: 用于监控虚拟机运行状态信息。它可以显示本地或者远程虚拟机进程的内存、垃圾收集、JIT编译 
  option：选项
  - `-class`
    监视类的装载/卸载数量、总空间以及类装载所耗时间；
  - `-gc`
    监视java heap情况，包括eden区和两个survivor区、old区、永久区等的容量，已用空间和GC时间等信息；
  - `-gccapacity`
    监视内容与`-gc`基本是一致的，`-gccapacity`的输出包括heap各个区域使用到的最大最小空间；
  - `gcutil`
    监视内容同样与`-gc`基本一致，`-gcutil`的输出主要是heap各个区域使用空间占总空间百分比；
  - `gccause`
    与`-gcutil`功能一致，但是会额外输出导致上一次gc的原因；
  - `gcnew`
    监视young区gc情况；
  - `gcnewcapacity`
    监视内容与`-gcnew`基本相同，`-gcnewcapacity`的输出包括使用到的最大最小空间；
  - `-gcold`
    监视old区gc情况；
  - `-gcoldcapacity`
    监视内容与`-gcold`基本相同，`-gcoldcapacity`的输出包括使用到的最大最小空间；
  - `-gcpermcapacity`
    输出永久代使用到的最大最小空间；

- **jmap**: 用于生成堆内存快照（heapdump或者dump文件） 

- **jstack**: 生成虚拟机当前时刻的线程快照 

