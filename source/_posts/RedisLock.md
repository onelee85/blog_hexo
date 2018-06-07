title: 基于Redis分布式锁
author: James
tags:
  - redis
  - lock
categories:
  - 架构
date: 2017-05-08 10:21:00
---

# 关于分布式锁

分布式环境下，数据一致性问题一直是一个比较重要的话题，而又不同于单进程的情况。分布式与单机情况下最大的不同在于其不是多线程而是多进程。多线程由于可以共享堆内存，因此可以简单的采取内存作为标记存储位置。而进程之间甚至可能都不在同一台物理机上，因此需要将标记存储在一个所有进程都能看到的地方。

<!-- more -->

## 常见分布式锁方案

- 基于数据库实现分布式锁  
- 基于缓存的锁实现方案，如 redis
- 基于Zookeeper实现分布式锁 

本文将讨论第二种方式，基于Redis实现分布式锁。 

# 基于Redis的实现

## 可靠性

1. **排他性**：任意时刻只能一个客户端自有其锁
2. **避免死锁** ：如果某个持有锁的客户端发生异常，而不能主动释放锁，其他客户端也能获得锁
3. **容错性**：大部分节点能够正常运行，就能保证加锁和释放 



## 思路

使用redis的SETNX实现分布式锁，多个进程执行以下Redis命令： 

```bash
SETNX lock.id <current Unix time + lock timeout + 1>
```

SETNX是将 key 的值设为 value，当且仅当 key 不存在。若给定的 key 已经存在，则 SETNX 不做任何动作。   

- 返回1，说明该进程获得锁，SETNX将键 lock.id 的值设置为锁的超时时间，当前时间 +加上锁的有效时间。 
- 返回0，说明其他进程已经获得了锁，进程不能进入临界区。进程可以在一个循环中不断地尝试 SETNX 操作，以获得锁。



## 实现方式1

获得锁

```java
    public void lock(Integer sec) throws Exception{
        Boolean lock = Boolean.FALSE;
        Long curr = System.currentTimeMillis();//TODO 需要每个实例自己生成过期时间，要求多个客户端之间需要时间同步
        Long timeout = sec * LOCK_TIME_MILLIS;
        Long timestamp = curr + timeout;
        Long blockTime = timeout;
        while(!lock){
            // SETNX方式尝试获得锁
            lock = mainRedis.opsForValue().setIfAbsent(lock_key, timestamp.toString());
            if(lock){
                //设置失效时间
                mainRedis.expire(lock_key, timeout, TimeUnit.MILLISECONDS);//设置超时
                lock_value = timestamp.toString();
                return;
            }
            //超过重试次数unlock 此步骤避免超时设置失败导致异常
            blockTime -= WAIT_TIME_MILLIS;
            if( blockTime <= 0 ){
                String lock_value = mainRedis.opsForValue().get(lock_key);
                if(lock_value != null && Long.valueOf(lock_value) < curr)//TODO 时间不同步，可能导致误解锁
                    mainRedis.delete(lock_key);//TODO 非原子操作，可能导致误解锁.需借鉴lua脚本执行的方式
            }
            Thread.sleep(WAIT_TIME_MILLIS);"+blockTime);
        }
    }
```

- lock_key来当锁。
- timestamp作为value,在解锁的时候能有依据。
- SET IF NOT EXIST，即当key不存在时，我们进行set操作；若key已经存在，则不做任何操作。
-  timeout 为过期时间，超时判断
- blockTime  避免超时设置失败导致异常,当重试获得锁次数达到时候解锁

释放锁

```java
  public void unlock(){
        mainRedis.execute(new RedisCallback<Boolean>() {
            @Override
            public Boolean doInRedis(RedisConnection connection) throws DataAccessException {
                connection.watch(lock_key_bytes);//redis事务,防止此时key超时，其他实例获得新锁被误删除
                String alive_lock_value = KeyUtils.decode(connection.get(lock_key_bytes));
                connection.multi();
                //对比value是否为当前线程
                if (alive_lock_value != null && lock_value.equals(alive_lock_value)){
                    connection.del(lock_key_bytes);
                }
                return connection.exec() != null;
            }
        });
    }
```

## 实现方式2

最新版本jedis 解决获取锁设置失效时间原子问题，下面代码仅供参考:

```java
public boolean lock(String key, String request, int blockTime) throws InterruptedException {

        while (blockTime >= 0) {

            String result = this.jedis.set(lockPrefix + key, request, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, 10 * TIME);
            if (LOCK_MSG.equals(result)) {
                return true;
            }
            blockTime -= sleepTime;

            Thread.sleep(sleepTime);
        }
        return false;
}
```



## 实现方式3

设置Reids key超时时候可以用lua脚本达到原子操作。Redis如何实现的原子操作的呢?

Redis 使用单个 Lua 解释器去运行所有脚本，并且 Redis 也保证脚本会以原子性(atomic)的方式执行：当某个脚本正在运行的时候，不会有其他脚本或 Redis 命令被执行。 

# 总结

至此一个基于 Redis 的分布式锁完成，但是依然有些问题。

- 需要每个实例自己生成过期时间，要求多个客户端之间需要时间同步。
- 设置key超时时候可以用lua脚本达到原子操作。或者用新版本jedis包。
- 就算 Redis 是集群部署的，如果每个节点都只是 master 没有 slave，那么 master 宕机时该节点上的所有 key  在那一时刻都相当于是释放锁了，这样也会出现并发问题。就算是有 slave 节点，但如果在数据同步到 salve 之前 master  宕机也是会出现上面的问题。

感兴趣的朋友还可以参考 [Redisson](https://github.com/redisson/redisson) 的实现。