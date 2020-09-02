title: 探索 ConcurrentHashMap
author: James
tags:
  - ConcurrentHashMap
categories:
  - 语言
date: 2013-08-20 17:06:00
---

# 简介

ConcurrentHashMap 是 util.concurrent 包的重要成员。本文将结合 Java 内存模型，分析 JDK 源代码，探索 ConcurrentHashMap 高并发的具体实现机制。

<!-- more -->

# 结构分析

ConcurrentHashMap 类中包含两个静态内部类 HashEntry 和 Segment。HashEntry 用来封装映射表的键 / 值对；Segment 用来充当锁的角色，每个 Segment 对象守护整个散列映射表的若干个桶。每个桶是由若干个 HashEntry 对象链接起来的链表。一个 ConcurrentHashMap 实例中包含由若干个 Segment 对象组成的数组。*声明*: 以下源码是针对 `JDK 1.6`

## ConcurrentHashMap 类

ConcurrentHashMap 在默认并发级别会创建包含 16 个 Segment 对象的数组。每个 Segment 的成员对象 table 包含若干个散列表的桶。每个桶是由 HashEntry 链接起来的一个链表。如果键能均匀散列，每个 Segment 大约守护整个散列表中桶总数的 1/16。

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V> 
       implements ConcurrentMap<K, V>, Serializable { 
 
   /** 
    * 散列映射表的默认初始容量为 16，即初始默认为 16 个桶
    * 在构造函数中没有指定这个参数时，使用本参数
    */ 
   static final     int DEFAULT_INITIAL_CAPACITY= 16; 
 
   /** 
    * 散列映射表的默认装载因子为 0.75，该值是 table 中包含的 HashEntry 元素的个数与
* table 数组长度的比值
    * 当 table 中包含的 HashEntry 元素的个数超过了 table 数组的长度与装载因子的乘积时，
* 将触发 再散列
    * 在构造函数中没有指定这个参数时，使用本参数
    */ 
   static final float DEFAULT_LOAD_FACTOR= 0.75f; 
 
   /** 
    * 散列表的默认并发级别为 16。该值表示当前更新线程的估计数
    * 在构造函数中没有指定这个参数时，使用本参数
    */ 
   static final int DEFAULT_CONCURRENCY_LEVEL= 16; 
 
   /** 
    * segments 的掩码值
    * key 的散列码的高位用来选择具体的 segment 
    */ 
   final int segmentMask; 
 
   /** 
    * 偏移量
    */ 
   final int segmentShift; 
 
   /** 
    * 由 Segment 对象组成的数组
    */ 
   final Segment<K,V>[] segments; 
 
   /** 
    * 创建一个带有指定初始容量、加载因子和并发级别的新的空映射。
    */ 
   public ConcurrentHashMap(int initialCapacity, 
                            float loadFactor, int concurrencyLevel) { 
       if(!(loadFactor > 0) || initialCapacity < 0 || 
concurrencyLevel <= 0) 
           throw new IllegalArgumentException(); 
 
       if(concurrencyLevel > MAX_SEGMENTS) 
           concurrencyLevel = MAX_SEGMENTS; 
 
       // 寻找最佳匹配参数（不小于给定参数的最接近的 2 次幂） 
       int sshift = 0; 
       int ssize = 1; 
       while(ssize < concurrencyLevel) { 
           ++sshift; 
           ssize <<= 1; 
       } 
       segmentShift = 32 - sshift;       // 偏移量值
       segmentMask = ssize - 1;           // 掩码值 
       this.segments = Segment.newArray(ssize);   // 创建数组
 
       if (initialCapacity > MAXIMUM_CAPACITY) 
           initialCapacity = MAXIMUM_CAPACITY; 
       int c = initialCapacity / ssize; 
       if(c * ssize < initialCapacity) 
           ++c; 
       int cap = 1; 
       while(cap < c) 
           cap <<= 1; 
 
       // 依次遍历每个数组元素
       for(int i = 0; i < this.segments.length; ++i) 
           // 初始化每个数组元素引用的 Segment 对象
this.segments[i] = new Segment<K,V>(cap, loadFactor); 
   } 
 
   /** 
    * 创建一个带有默认初始容量 (16)、默认加载因子 (0.75) 和 默认并发级别 (16) 
 * 的空散列映射表。
    */ 
   public ConcurrentHashMap() { 
       // 使用三个默认参数，调用上面重载的构造函数来创建空散列映射表
this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL); 
}
```

结构示意图：

![ConcurrentHashMap](/images/concurrentHashMap/image005.jpg)

## HashEntry 类

HashEntry 用来封装散列映射表中的键值对。在 HashEntry 类中，key，hash 和 next 域都被声明为 final 型，value 域被声明为 volatile 型。

```java
static final class HashEntry<K,V> { 
       final K key;                       // 声明 key 为 final 型
       final int hash;                   // 声明 hash 值为 final 型 
       volatile V value;                 // 声明 value 为 volatile 型
       final HashEntry<K,V> next;      // 声明 next 为 final 型 
 
       HashEntry(K key, int hash, HashEntry<K,V> next, V value) { 
           this.key = key; 
           this.hash = hash; 
           this.next = next; 
           this.value = value; 
       } 
}
```

在 ConcurrentHashMap 中，在散列时如果产生“碰撞”，将采用“分离链接法”来处理“碰撞”：把“碰撞”的 HashEntry 对象链接成一个链表。由于 HashEntry 的 next 域为 final 型，所以新节点只能在链表的表头处插入。

## Segment 类

Segment 类继承于 ReentrantLock 类，从而使得 Segment 对象能充当锁的角色。每个 Segment 对象用来守护其（成员对象 table 中）包含的若干个桶。

table 是一个由 HashEntry 对象组成的数组。table 数组的每一个数组成员就是散列映射表的一个桶。

count 变量是一个计数器，它表示每个 Segment 对象管理的 table 数组（若干个 HashEntry 组成的链表）包含的 HashEntry 对象的个数。每一个 Segment 对象都有一个 count 对象来表示本 Segment 中包含的 HashEntry 对象的总数。注意，之所以在每个 Segment 对象中包含一个计数器，而不是在 `ConcurrentHashMap 中使用全局的计数器，是为了避免出现“热点域”而影响 ConcurrentHashMap 的并发性。`

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable { 
       /** 
        * 在本 segment 范围内，包含的 HashEntry 元素的个数
        * 该变量被声明为 volatile 型
        */ 
       transient volatile int count; 
 
       /** 
        * table 被更新的次数
        */ 
       transient int modCount; 
 
       /** 
        * 当 table 中包含的 HashEntry 元素的个数超过本变量值时，触发 table 的再散列
        */ 
       transient int threshold; 
 
       /** 
        * table 是由 HashEntry 对象组成的数组
        * 如果散列时发生碰撞，碰撞的 HashEntry 对象就以链表的形式链接成一个链表
        * table 数组的数组成员代表散列映射表的一个桶
        * 每个 table 守护整个 ConcurrentHashMap 包含桶总数的一部分
        * 如果并发级别为 16，table 则守护 ConcurrentHashMap 包含的桶总数的 1/16 
        */ 
       transient volatile HashEntry<K,V>[] table; 
 
       /** 
        * 装载因子
        */ 
       final float loadFactor; 
 
       Segment(int initialCapacity, float lf) { 
           loadFactor = lf; 
           setTable(HashEntry.<K,V>newArray(initialCapacity)); 
       } 
 
       /** 
        * 设置 table 引用到这个新生成的 HashEntry 数组
        * 只能在持有锁或构造函数中调用本方法
        */ 
       void setTable(HashEntry<K,V>[] newTable) { 
           // 计算临界阀值为新数组的长度与装载因子的乘积
           threshold = (int)(newTable.length * loadFactor); 
           table = newTable; 
       } 
 
       /** 
        * 根据 key 的散列值，找到 table 中对应的那个桶（table 数组的某个数组成员）
        */ 
       HashEntry<K,V> getFirst(int hash) { 
           HashEntry<K,V>[] tab = table; 
           // 把散列值与 table 数组长度减 1 的值相“与”，
// 得到散列值对应的 table 数组的下标
           // 然后返回 table 数组中此下标对应的 HashEntry 元素
           return tab[hash & (tab.length - 1)]; 
       } 
}
```

插入三个节点后 Segment 的结构示意图：

![Segment](/images/concurrentHashMap/image004.jpg)

## Put 方法的实现

1.根据 key 计算出对应的 hash 值：

```java
public V put(K key, V value) { 
       if (value == null)          //ConcurrentHashMap 中不允许用 null 作为映射值
           throw new NullPointerException(); 
       int hash = hash(key.hashCode());        // 计算键对应的散列码
       // 根据散列码找到对应的 Segment 
       return segmentFor(hash).put(key, hash, value, false); 
}
```

2.根据 hash 值找到对应的 Segment:

```java
/** 
    * 使用 key 的散列码来得到 segments 数组中对应的 Segment 
    */ 
final Segment<K,V> segmentFor(int hash) { 
   // 将散列值右移 segmentShift 个位，并在高位填充 0 
   // 然后把得到的值与 segmentMask 相“与”
// 从而得到 hash 值对应的 segments 数组的下标值
// 最后根据下标值返回散列码对应的 Segment 对象
       return segments[(hash >>> segmentShift) & segmentMask]; 
}
```

3.在 Segment 中执行具体的 put 操作

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) { 
           lock();  // 加锁，这里是锁定某个 Segment 对象而非整个 ConcurrentHashMap 
           try { 
               int c = count; 
 
               if (c++ > threshold)     // 如果超过再散列的阈值
                   rehash();              // 执行再散列，table 数组的长度将扩充一倍
 
               HashEntry<K,V>[] tab = table; 
               // 把散列码值与 table 数组的长度减 1 的值相“与”
               // 得到该散列码对应的 table 数组的下标值
               int index = hash & (tab.length - 1); 
               // 找到散列码对应的具体的那个桶
               HashEntry<K,V> first = tab[index]; 
 
               HashEntry<K,V> e = first; 
               while (e != null && (e.hash != hash || !key.equals(e.key))) 
                   e = e.next; 
 
               V oldValue; 
               if (e != null) {            // 如果键 / 值对以经存在
                   oldValue = e.value; 
                   if (!onlyIfAbsent) 
                       e.value = value;    // 设置 value 值
               } 
               else {                        // 键 / 值对不存在 
                   oldValue = null; 
                   ++modCount;         // 要添加新节点到链表中，所以 modCont 要加 1  
                   // 创建新节点，并添加到链表的头部 
                   tab[index] = new HashEntry<K,V>(key, hash, first, value); 
                   count = c;               // 写 count 变量
               } 
               return oldValue; 
           } finally { 
               unlock();                     // 解锁
           } 
       }
```

加锁操作是针对（键的 hash 值对应的）某个具体的 Segment，锁定的是该 Segment 而不是整个 ConcurrentHashMap`。因为插入键 / 值对操作只是在这个 Segment 包含的某个桶中完成，不需要锁定整个`ConcurrentHashMap。`此时，其他写线程对另外 15 个`Segment 的加锁并不会因为当前线程对这个 Segment 的加锁而阻塞。同时，所有读线程几乎不会因本线程的加锁而阻塞（除非读线程刚好读到这个 Segment 中某个 `HashEntry 的 value 域的值为 null，此时需要加锁后重新读取该值`）。

相比较于 `HashTable 和由同步包装器包装的 HashMap``每次只能有一个线程执行读或写操作，`ConcurrentHashMap 在并发访问性能上有了质的提高。*在理想状态下*，ConcurrentHashMap 可以支持 16 个线程执行并发写操作（如果并发级别设置为 16），及任意数量线程的读操作。

# 总结

ConcurrentHashMap 是一个并发散列映射表的实现，它允许完全并发的读取，并且支持给定数量的并发更新。相比于 `HashTable 和`用同步包装器包装的 HashMap（Collections.synchronizedMap(new HashMap())），ConcurrentHashMap 拥有更高的并发性。在 `HashTable 和由同步包装器包装的 HashMap 中，使用一个全局的锁来同步不同线程间的并发访问。同一时间点，只能有一个线程持有锁，也就是说在同一时间点，只能有一个线程能访问容器。这虽然保证多线程间的安全并发访问，但同时也导致对容器的访问变成``*串行化*``的了。`

在使用锁来协调多线程间并发访问的模式下，减小对锁的竞争可以有效提高并发性。有两种方式可以减小对锁的竞争：

1. 减小请求 同一个锁的 频率。
2. 减少持有锁的 时间。

ConcurrentHashMap 的高并发性主要来自于三个方面：

1. 用分离锁实现多个线程间的更深层次的共享访问。
2. 用 HashEntery 对象的不变性来降低执行读操作的线程在遍历链表期间对加锁的需求。
3. 通过对同一个 Volatile 变量的写 / 读访问，协调不同线程间读 / 写操作的内存可见性。

使用分离锁，减小了请求 *同一个锁*的频率。

通过 HashEntery 对象的不变性及对同一个 Volatile 变量的读 / 写来协调内存可见性，使得 读操作大多数时候不需要加锁就能成功获取到需要的值。由于散列映射表在实际应用中大多数操作都是成功的 读操作，所以 2 和 3 既可以减少请求同一个锁的频率，也可以有效减少持有锁的时间。

通过减小请求同一个锁的频率和尽量减少持有锁的时间 `，使得 ConcurrentHashMap 的并发性相对于 HashTable 和`用同步包装器包装的 HashMap`有了质的提高。`

