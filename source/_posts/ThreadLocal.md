title: 深入ThreadLocal
author: James
tags:
  - ThreadLocal
categories:
  - Java
date: 2014-06-05 09:54:00
---

# 前言

Threadlocal是什么？它一个线程的本地变量副本，当工作在多线程的环境中每个线程中的对象使用Threadlocal维护变量。Threadlocal为每个线程分配一个独立的变量副本，所以线程都是独立访问和操作自己的副本不会影响到其他线程

<!-- more -->

# 原理

## 看看源码

在使用`ThreadLocal`时，首先创建`ThreadLocal`对象，然后再调用其`set(T)`、`T get()`方法 

```java
public void set(T value) { 
    Thread t = Thread.currentThread(); 
    ThreadLocalMap map = getMap(t); 
    if (map != null) 
        map.set(this, value); 
    else createMap(t, value); 
}

//返回Thread实例的threadLocals属性
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

先拿到保存键值对的`ThreadLocalMap`对象实例`map`，如果`map`为空（第一次调用的时候`map`值为`null`），则去创建一个`ThreadLocalMap`对象并赋值给`map`，并把键值对保存到`map`中。 

```java
public class Thread{
    ...
	ThreadLocal.ThreadLocalMap threadLocals = null
}
```

每个线程引用的`ThreadLocal`副本值都是保存在当前线程`Thread`对象里面的。存储结构为`ThreadLocalMap`类型，`ThreadLocalMap`保存的键类型为`ThreadLocal`，值为`副本值`。 

```java
public T get() { 
    Thread t = Thread.currentThread(); 
    ThreadLocalMap map = getMap(t); 
    if (map != null) { 
        ThreadLocalMap.Entry e = map.getEntry(this); 
        if (e != null) return (T)e.value; 
    } 
    return setInitialValue(); 
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

当前线程`Thread`对象实例中保存的`ThreadLocalMap`对象`map`，然后从`map`中读取键为`this`（即`ThreadLocal`类实例）对应的值。 

### ThreadLocalMap 

```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal> {
        Object value; // 实际保存的值
        Entry(ThreadLocal k, Object v) {
            super(k);
            value = v;
        }
    }
    /**
     * 哈希表初始大小，但是这个值无论怎么变化都必须是2的N次方
     */
    private static final int INITIAL_CAPACITY = 16;
    /**
     * 哈希表中实际存放对象的容器，该容器的大小也必须是2的幂数倍
     */
    private Entry[] table;
    /**
     * 表中Entry元素的数量
     */
    private int size = 0;

    /**
     * 哈希表的扩容阈值
     */
    private int threshold; // 默认值为0

    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }
    /**
    * 并不是Thread被创建后就一定会创建一个新的ThreadLocalMap，
    * 除非当前Thread真的用了ThreadLocal
    * 并且赋值到ThreadLocal后才会创建一个ThreadLocalMap
    */
    ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode 
              & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
```

1. 存放对象信息的表是一个数组。这类方式和HashMap有点像 
2. 数组元素是一个**WeakReference（弱引用）**的实现;因为线程的执行时间可能很长，但是对应的ThreadLocal对象生成时间未必有线程的执行寿命那般长，在对应ThreadLocal对象由该线程作为根节点出发，逻辑上不可达时，就应该可以被GC，如果使用了强引用，该对象无法被成功GC，因此会带来内存泄露的问题 



## 这样设计 why?

我们的第一想法使用**全局ConcurrentMap结构**。在对应的`ThreadLocal`对象内维持一个本地变量表，以当前线程（使用Thread.currentThread()方法）作为key，查找对应的的本地变量（value值），那么这么设计存在什么问题呢？ 

第一，全局的ConcurrentMap表，这类数据结构虽然是一类分段式且线程安全的容器，但是这类容器仍然会有线程同步的的额外开销。

第二，随着线程的销毁，原有的ConcurrentMap没有被回收，因此导致了内存泄露。



# 总结

1. ThreadLocal 通过隐式的在不同线程内创建独立实例副本避免了实例线程安全的问题 
2. 每个线程持有一个 Map 并维护了 ThreadLocal 对象与具体实例的映射，该 Map 由于只被持有它的线程访问，故不存在线程安全以及锁的问题 
3. ThreadLocalMap 的 Entry 对 ThreadLocal 的引用为弱引用，避免了 ThreadLocal 对象无法被回收的问题 
4. ThreadLocal 适用于变量在线程间隔离且在方法间共享的场景 