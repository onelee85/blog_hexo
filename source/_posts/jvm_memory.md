title: JVM内存分析
author: James
tags:
  - jvm
  - java
categories:
  - 语言
date: 2013-05-27 16:13:00
---
# 概要

开发人员编写Java代码(.java文件)，然后将之编译成字节码(.class文件)。最后字节码被装入内存，一旦字节码进入虚拟机，它就会被解释器解释执行，或者是被即时代码发生器有选择的转换成机器码执行。在Java平台的结构中, 可以看出，Java虚拟机(JVM) 处在核心的位置，是程序与底层操作系统和硬件无关的关键。它的下方是移植接口，移植接口由两部分组成：适配器和Java操作系统, 其中依赖于平台的部分称为适配器；JVM 通过移植接口在具体的平台和操作系统上实现；在JVM 的上方是Java的基本类库和扩展类库以及它们的API， 利用Java API编写的应用程序(application) 和小程序(Java applet) 可以在任何Java平台上运行而无需考虑底层平台, 就是因为有Java虚拟机(JVM)实现了程序与操作系统的分离，从而实现了Java 的平台无关性。

<!-- more -->

![jvm_struct2](/images/jvm/jvm_struct2.jpg)



# 详细介绍

## 程序计数器

程序计数器，可以看做是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里，字节码解释器工作就是通过改变程序计数器的值来选择下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都要依赖这个计数器来完成。

多线程中，为了让线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间互不影响、独立存储，因此这块内存是 线程私有 的。

当线程正在执行的是一个Java方法，这个计数器记录的是在正在执行的虚拟机字节码指令的地址；当执行的是Native方法，这个计数器值为空。

此内存区域是唯一一个没有规定任何OutOfMemoryError情况的区域 。

##  Java虚拟机栈

Java虚拟机栈也是线程私有的 ，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧用于存储**局部变量表**、**操作数栈**、**动态链表**、方法出口信息等。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

**局部变量表**中存放了编译器可知的各种**基本数据类型**(boolean、byte、char、short、int、float、long、double)、对象引用（reference类型，它不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。

如果扩展时无法申请到足够的内存，就会抛出OutOfMemoryError异常。

## Java堆

Java堆是所有线程共享的一块内存区域，在虚拟机启动时创建，此内存区域的唯一目的就是存放**对象实例 **。

Java堆是垃圾收集器管理的主要区域。由于现在收集器基本采用分代回收算法，所以Java堆还可细分为：新生代和老年代。从内存分配的角度来看，线程共享的Java堆中可能划分出多个线程私有的分配缓冲区(TLAB)。

Java堆可以处于物理上不连续的内存空间，只要逻辑上连续的即可。在实现上，既可以实现固定大小的，也可以是扩展的。

如果堆中没有内存完成实例分配，并且堆也无法完成扩展时，将会抛出OutOfMemoryError异常。

![jvm_heap](/images/jvm/jvm_heap.jpg)

## 方法区

方法区是各个线程共享的内存区域，它用于存储已被虚拟机加载的*类信息*、*常量*、*静态变量*、即时编译器编译后的代码等数据 。

相对而言，垃圾收集行为在这个区域比较少出现，但并非数据进了方法区就永久的存在了，这个区域的内存回收目标主要是针对常量池的回收和对类型的卸载，

运行时常量池：用于存放编译期生成的各种字面量和符号引用。

当方法区无法满足内存分配需要时，将抛出OutOfMemoryError异常。

## 直接内存

直接内存不是虚拟机运行时数据区的一部分，在NIO类中引入一种基于通道与缓冲区的IO方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。

直接内存的分配不会受到Java堆大小的限制，但是会受到本机内存大小的限制，所有也可能会抛OutOfMemoryError异常。

## 本地方法栈

本地方法栈与虚拟机的作用相似，不同之处在于虚拟机栈为虚拟机执行的Java方法服务，而本地方法栈则为虚拟机使用到的Native方法服务。有的虚拟机直接把本地方法栈和虚拟机栈合二为一。

会抛出stackOverflowError和OutOfMemoryError异常。

一个数组的在堆中的简单表示:

![array](/images/jvm/array.jpg)



# 设置参数

| 设置                       | 说明                                                         |
| -------------------------- | ------------------------------------------------------------ |
| -Xms512m                   | 表示JVM初始分配的堆内存大小为512m（JVM Heap(堆内存)最小尺寸，初始分配） |
| -Xmx1024m                  | JVM最大允许分配的堆内存大小为1024m，按需分配（JVM Heap(堆内存)最大允许的尺寸，按需分配） |
| -XX:PermSize=512M          | JVM初始分配的非堆内存                                        |
| -XX:MaxPermSize=1024M      | JVM最大允许分配的非堆内存，按需分配                          |
| -XX:NewSize/-XX:MaxNewSize | 定义YOUNG段的尺寸，NewSize为JVM启动时YOUNG的内存大小； MaxNewSize为最大可占用的YOUNG内存大小。 |
| -XX:SurvivorRatio          | 设置YOUNG代中Survivor空间和Eden空间的比例                    |

## 说明

- -Xmx不指定或者指定偏小，应用可能会导致java.lang.OutOfMemory错误
- PermSize和MaxPermSize指明虚拟机为java永久生成对象（Permanate generation）如，class对象、方法对象这些可反射（reflective）对象分配内存限制
- -XX:MaxPermSize分配过小会导致：java.lang.OutOfMemoryError: PermGen space

# 内存监控方法

jmap -heap 查看java 堆（heap）使用情况

 参数配置：内存溢出自动dump内存快照

-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/logs

jmap -dump:format=b,file=heap.bin <pid>  

format=b的含义是，dump出来的文件时二进制格式。

file-heap.bin的含义是，dump出来的文件名是heap.bin。

 <pid>就是JVM的进程号。

 （在linux下）先执行ps aux | grep java，找到JVM的pid；然后再执行jmap -dump:format=b,file=heap.bin <pid>，得到heap dump文件。

或者可以使用 jps -lv 查看java pid