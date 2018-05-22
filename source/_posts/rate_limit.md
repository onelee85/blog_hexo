title: 高并发限流探究
author: James
tags:
  - 接口限流
categories:
  - 架构
date: 2017-07-22 14:00:00
---

# 前言

**在开发高并发系统时，有三把利器用来保护系统：缓存、降级和限流**。那么何为限流呢？通过对并发访问/请求进行限速，或者对一个时间窗口内的请求进行限速来保护系统，一旦达到限制速率则可以拒绝服务、排队或等待、降级等处理 。通过限流，我们可以很好地控制系统的qps，防止客户端对于接口的滥用，保护服务器的资源， 从而达到保护系统的目的。本文主要介绍几种限流的方式。 

<!-- more -->

# 常见限流算法

计数器、滑动窗口、漏桶、令牌。 

## 计数器

计数器法是限流算法里最简单也是最容易实现的一种算法。比如我们规定，对于A接口来说，我们1分钟的访问次数不能超过100个。那么我们可以这么做：在一开始的时候，我们可以设置一个计数器counter，每当一个请求过来的时候，counter就加1，如果counter的值大于100并且该请求与第一个请求的间隔时间还在1分钟之内，那么说明请求数过多；如果该请求与第一个请求的间隔时间大于1分钟，且counter的值还在限流范围内，那么就重置counter。

```java
class Counter{
    public long timeStamp = System.currentTimeMillis();
    public int reqCount = 0;
    public final int limit = 10; // 时间窗口内最大请求数
    public final long interval = 10; // 时间窗口ms
    public boolean tryAcquire() {
        long now = System.currentTimeMillis();
        if (now < timeStamp + interval) {
            // 在时间窗口内
            reqCount++;
            // 判断当前时间窗口内是否超过最大请求控制数
            return reqCount <= limit;
        }
        else {
            timeStamp = now;
            // 超时后重置
            reqCount = 0;
            return true;
        }
    }
}
```

此算法很简单，很多情况下也很实用，但有个问题就是临界问题。假设时间间隔为1分钟不能超过100个请求，如果0~58秒都没有请求，而是在最后一秒钟瞬间发送100个请求。那么由开始我们设置的每分钟100请求平均每秒1.7个请求变成了每秒100个请求，超过了我们的速率限制。

## 滑动窗口

滑动窗口，又称rolling window。为了解决这个问题，我们引入了滑动窗口算法。 

滑动窗口算法对固定的时间片进行划分，并随着时间的流逝进行移动，这样就巧妙的避开了计数器的临界点问题。也就是说这些固定数量的可以移动的格子，将会进行计数判断阀值，因此格子的数量影响着滑动窗口算法的精度。 

## 漏桶

虽然滑动窗口有效避免了时间临界点的问题，但是依然有时间片的概念，而漏桶算法在这方面比滑动窗口而言，更加先进。
有一个固定的桶，进水的速率是不确定的，但是出水的速率是恒定的，当水满的时候是会溢出的。

![](/images/ratelimit/a1.png)

```java
class LeakyBucket {
    public long timeStamp = System.currentTimeMillis();
    public int capacity = 10; // 桶的容量
    public int rate = 10; // 水漏出的速度
    public long water = 0; // 当前水量(当前累积请求数)
    public boolean tryAcquire() {
        long now = System.currentTimeMillis();
        water = Math.max(0, water - (now - timeStamp) * rate); // 先执行漏水，计算剩余水量
        timeStamp = now;
        if ((water + 1) < capacity) {
            // 尝试加水,并且水还未满
            water += 1;
            return true;
        }
        else {
            // 水满，拒绝加水
            return false;
        }
    }
}
```

## 令牌桶

令牌桶算法：有一个固定容量的桶，开始的时候是空的，然后token以一个固定速率放入桶内，如果桶已经满了则丢弃，每当一个请求过来时会尝试从桶中移除一个token，如果没有令牌则无法通过。

![](/images/ratelimit/a2.png)

下面为简单的伪代码实现

```java
class TokenBucket {
    public long timeStamp = System.currentTimeMillis();
    public int capacity = 100; // 桶的容量
    public int rate = 10; // 令牌放入速度
    public long tokens = 0; // 当前令牌数量
    public boolean tryAcquire() {
        long now = System.currentTimeMillis();
        // 先添加令牌
        tokens = Math.min(capacity, tokens + (now - timeStamp) * rate);
        timeStamp = now;
        if (tokens < 1) {
            // 若不到1个令牌,则拒绝
            return false;
        }
        else {
            // 还有令牌，领取令牌
            tokens -= 1;
            return true;
        }
    }
}
```

google开源工具包guava提供了限流工具类RateLimiter，该类基于“令牌桶算法” 

我们只需要告诉RateLimiter系统限制的QPS是多少，那么RateLimiter将以这个速度往桶里面放入令牌，然后请求的时候，通过tryAcquire()方法向RateLimiter获取许可（令牌）。 

```java
 class GuavaLimiter{
     private RateLimiter rateLimiter  = RateLimiter.create(10);//qps 每秒的请求速率

     public boolean tryAcquire() {
         return rateLimiter.tryAcquire(0, TimeUnit.MILLISECONDS);//尝试获取令牌
     }
 }
```

