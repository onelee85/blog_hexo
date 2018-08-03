title: CAS-原理
author: James
tags:
  - CAS
categories:
  - 并发
date: 2016-08-30 11:37:00
---
# 什么是 CAS

CAS （compareAndSwap），中文翻译过来叫做比较交换，一种无锁原子算法。过程是这样：它包含 3 个参数 （V，E，N），V表示要更新变量的值，E表示预期值，N表示新值。仅当 V值等于E值时，才会将V的值设为N，如果V值和E值不同，则说明已经有其他线程做两个更新，则当前线程则什么都不做。最后返回当前V的真实值。

<!-- more -->

# 为什么会出现CAS

我们平时在编发编程的时候，经常会遇到如下的情况；比如多线程对变量进行修改，同时只能有一个线程进入同步块修改变量的值

```java
void synchronized  modify(int b){
  a = a + b；
}
```

如果上面的代码不加同步关键字synchronized 的话，会出现线程安全的问题；但是加了锁之后又会消耗性能-获取锁，释放锁，等待和阻塞都会消耗性能。那么有没有办法做到不加锁？

我们来分析下上面的代码
1. 线程读取a的值
2. 然后将a和b相加
3. 最后赋值给a

当多线程的情况下，会出现问题：两个线程同时访问a，获取a的值，并同时对a加上b的值，然后同时赋值给a。
这样可能会导致 a 的值只增加了一次b，但实际上我们想加 2次b。
仔细思考，问题可能出在给a赋值操作的时候，a的值其实已经改变了，那有有没有办法在赋值的时候判断当前a变了是否已经被其他线程改变了呢，例如:

 ```java
void modify(int b) {
    int expect = a;
   	int c = a + b;
    compareAndSwap(a, expect, c)；
}

boolean compareAndSwap(int expect ,int c ){
       if (a == backup) {
           a = c;
           return true;
       }
    return false;
}
 ```

我们是不是可以先备份a的值，然后对a进行计算，最后比较a的值和期待值是否一致，如果a没有被其他线程修改过，这时候能够进行修改。这里是伪代码，上述方法中有多步操作，并不是原子操作。

# 实现原理 

当多个线程同时使用CAS 操作一个变量时，只有一个线程会胜出并成功更新，其余线程均会失败。失败的线程不会挂起，仅是被告知失败，并且允许再次尝试，当然也允许实现的线程放弃操作。基于这样的原理，CAS 操作即使没有锁，也可以发现其他线程对当前线程的干扰。

与锁相比，使用CAS会使程序看起来更加复杂一些，但由于其非阻塞的，它对死锁问题天生免疫，并且，线程间的相互影响也非常小。更为重要的是，使用无锁的方式完全没有锁竞争带来的系统开销，也没有线程间频繁调度带来的开销，因此要比基于锁的方式拥有更优越的性能。

# 底层原理

通过CPU底层指令实现的 

1.处理器自动保证基本内存操作的原子性
   处理器会自动保证基本的内存操作的原子性。处理器保证从系统内存当中读取或者写入一个字节是原子的，意思是当一个处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。 但是复杂的内存操作处理器不能自动保证其原子性，比如跨总线宽度，跨多个缓存行，跨页表的访问。但是处理器提供总线锁定和缓存锁定两个机制来保证复杂内存操作的原子性。 

2.使用总线锁保证原子性
   总线锁定其实就是处理器使用了总线锁，所谓总线锁就是使用处理器提供的一个 LOCK# 信号，当一个处理器咋总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占共享内存。 

3.使用缓存锁保证原子性
   通过缓存锁定来保证原子性。 所谓 缓存锁定 是指内存区域如果被缓存在处理器的缓存行中，并且在Lock 操作期间被锁定，那么当他执行锁操作写回到内存时，处理器不在总线上声言 LOCK# 信号，而时修改内部的内存地址，并允许他的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改两个以上处理器缓存的内存区域数据（这里和 volatile 的可见性原理相同），当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。 

   注意：有两种情况下处理器不会使用缓存锁定。
   1. 当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行时，则处理器会调用**总线锁定**。
   2. 有些处理器不支持缓存锁定，对于 Intel 486 和 Pentium 处理器，就是锁定的内存区域在处理器的缓存行也会调用总线锁定。



# Java中的CAS

Java提供了 java.util.concurrent.atomic 包，该包下所有的类都是原子操做，我们重点看下*AtomicInteger*  这个类。找到该类的*compareAndSet*方法，也就是比较并且设置。我们看看该方法实现： 

```java
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```
该方法调用了 unsafe 类的 compareAndSwapInt方法，有几个参数，一个是该变量的内存地址，一个是期望值，一个是更新值，一个是对象自身。完全符合我们之前CAS 的定义。

再往下寻找我们发现 Unsafe的`compareAndSwapInt` 是 Native 的方法： 是借助C来调用CPU底层指令实现的。 

## unsafe 是什么 
该类在 rt.jar 包中，是 sun.misc 包下。并且都是 class 文件，注释都没有，符合他的名字：不安全。
我们能构造他吗？不能，除非反射。
下载一个 OpenJdk 的源码继续向下探索，我们发现在 `/jdk9u/hotspot/src/share/vm/unsafe.cpp` 中有这样的代码

```c++
{CC "compareAndSetInt",   CC "(" OBJ "J""I""I"")Z",  FN_PTR(Unsafe_CompareAndSetInt)}
```

这个涉及到，JNI 的调用 搜索 `Unsafe_CompareAndSetInt`后发现:

```C++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSetInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)) {
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *)index_oop_from_field_offset_long(p, offset);

  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
} UNSAFE_END
```

核心**Atomic::cmpxchg **,针对不同的操作系统,JVM 对于 Atomic::cmpxchg 应该有不同的实现.
`cmpxchgl` 就是汇编版的“比较并交换”。 

**总结**:
- java 的 cas 利用的的是 unsafe 这个类提供的 cas 操作。
- unsafe 的cas 依赖了的是 jvm 针对不同的操作系统实现的 Atomic::cmpxchg
- Atomic::cmpxchg 的实现使用了汇编的 cas 操作，并使用 cpu 硬件提供的 lock信号保证其原子性

# CAS缺点

1. **ABA问题 **因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。
2. **循环时间长开销大 ** 自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。 
3. **只能保证一个共享变量的原子操作** 当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性。在java.util.concurrent.atomic 包中，就有 AtomicReference 来保证引用的原子性 它内部不仅维护了对象值，还维护了一个时间戳（我这里把它称为时间戳，实际上它可以使任何一个整数，它使用整数来表示状态值）。当AtomicStampedReference对应的数值被修改时，除了更新数据本身外，还必须要更新时间戳。当AtomicStampedReference设置对象值时，对象值以及时间戳都必须满足期望值，写入才会成功。因此，即使对象值被反复读写，写回原值，只要时间戳发生变化，就能防止不恰当的写入。 