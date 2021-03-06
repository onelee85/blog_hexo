title: 并发-volatile
author: James
tags:
  - volatile
categories:
  - 并发
date: 2015-08-07 10:26:00
---
# 概述
JAVA关键字volatile，经常在各种文章和博客中见到，但却感觉总是一知半解，实际研发中先少有用到，此篇博客了解清楚它的作用和原理。打算从几个方面来描述:

1. 内存模型的相关概念
2. Happen-Before 原则
3. volatile 是什么

<!-- more -->

# 内存模型

可以理解为在特定的操作协议下，对特定的内存或高速缓存进行读写访问的过程抽象。不同架构的CPU 有不同的内存模型。 

![](/images/volatile/java-memory.png)

因为多个CPU的多级缓存访问同一个内存条可能会导致数据不一致。所以需要一个协议，让这些处理器在访问内存的时候遵守这些协议保证数据的一致性。 

## 3个特性

1. **原子性（Atomicity）：**是指一个操作是不可中断的，即使是多个线程同时执行的情况下，一个操作一旦开始，就不会被其它线程干扰。对于基本类型的读写操作基本都具有原子性的（在32位操作系统中 long 和 double 类型数据的读写不是原子性的，因为它们有64位）。

   如果用户需要操作一个更到的范围保证原子性，那么，Java 内存模型提供了 lock 和 unlock （这是8种内存操操作中的2种）操作来满足这种需求，但是没有提供给程序员这两个操作，提供了更抽象的 monitorenter 和 moniterexit 两个字节码指令，也就是 synchronized 关键字。因此在 synchronized 块之间的操作都是原子性的。 

2. **可见性（Visibility）：**是指在多线程环境下，当一个线程修改了某一个共享变量的值，其它线程能够立刻知道这个修改。

3. **有序性（Ordering）：**是指程序的执行顺序是按照代码的先后顺序执行的；对于这句话如果在单线程中所有的操作都是有序的，但是在多线程环境下，一个线程的操作相对于另外一个线程的操作是无序的。

# Happen-Before 原则

1. 程序次序原则：一个线程内，按照程序代码顺序，书写在前面的操作先发生于书写在后面的操作。
2. volatile 规则：volatile 变量的写，先发生于读，这保证了 volatile 变量的可见性。
3. 锁规则：解锁（unlock） 必然发生在随后的加锁（lock）前。
4. 传递性：A先于B，B先于C，那么A必然先于C。
5. 线程的 start 方法先于他的每一个动作。
6. 线程的所有操作先于线程的终结。
7. 线程的中断（interrupt（））先于被中断的代码。
8. 对象的构造函数，结束先于 finalize 方法。

#  volatile

volatile 修饰的变量保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。因为当对普通变量进行读写的时候，每个线程先从内存拷贝变量到CPU缓存中。如果计算机有多个CPU，每个线程可能在不同的CPU上被处理，这意味着每个线程可以拷贝到不同的CPU cache中。而volatile修饰的变量，JVM保证了每次读变量都从内存中读，跳过CPU cache这一步。volatile修饰的变量禁止进行指令重排序，所以能在一定程度上保证有序性。只能保证该变量所在的语句还是原来的位置，并不能保证该语句之前或之后的语句是否被打乱。 

## 特性

1. 当一个变量被 volatile 修饰之后，能保证此变量对所有线程的可见性，即当一个线程修改了这个变量的值，新值对其它线程是立即可见的。
2. 被 volatile 修饰的变量通过查询内存屏障来禁止指令重排序，所以能在一定程度上保证有序性。
3. 对任意单个 volatile 变量的读/写具有原子性，但类似于 volatile++ 这种复合操作不具有原子性。 

## 原理

> 《深入理解Java虚拟机》： “观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令” 。

1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成 。
2. 它会强制将对缓存的修改操作立即写入主存 。
3. 如果是写操作，它会导致其他CPU中对应的缓存行无效。 

# 总结

 内存模型是抽象化了硬件的内存模型（使用了多级缓存导致出现缓存一致性协议），屏蔽了各个 CPU 和操作系统的差异。 

Java内存模型 主要围绕着原子性，可见性，有序性来设置规范。 

Happen-Before 原则规定了哪些是虚拟机不能重排序的，其中包括了锁的规定，volatile 变量的读与写规定。 

volatile 底层的实现还是 CPU 的 lock 指令，通过刷新其余的CPU 的Cache 保证可见性，通过内存栅栏保证了有序性。 

