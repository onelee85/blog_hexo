title: 线程池探究
author: James
tags:
  - threadpool
categories:
  - 实战
date: 2013-07-26 14:07:00
---

# 基本概念

在多线程编程的时候，为每一个任务临时分配一个线程开销很大，所以线程池应运而生，成为我们管理线程的工具。Java中提供了`Executor`接口，提供了一种标准的方法将任务的提交过程和执行过程解耦开来，并用`Runnable`表示任务。

下面我们将基于JDK1.7分析下线程池框架的实现`ThreadPoolExecutor`以及用途。

# 源码分析

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

## 核心参数

- **corePoolSize** ： 线程池中所保存的核心线程数，包括空闲线程。
- **maximumPoolSize** ：允许的最大线程数。
- **workQueue** ：阻塞队列，存储待执行的队列。
  - JDK中提供如下阻塞队列：
    1. `ArrayBlockingQueue`：基于数组结构的有界阻塞队列，按FIFO排序任务；
    2. `LinkedBlockingQuene`：基于链表结构的阻塞队列，按FIFO排序任务
    3. `SynchronousQuene`：一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态
    4. `priorityBlockingQuene`：具有优先级的无界阻塞队列
- **keepAliveTime** ：当无任务执行时，线程空闲的存活时间，时间单位由`TimeUnit`指定。
- **threadFactory** ： 是构造Thread的方法，你可以自己去包装和传递，主要实现newThread方法。
- **handler** ：达到`maximumPoolSize`后丢弃处理的方法，提供了几种丢弃处理的方法，当然你也可以自己实现接口：`RejectedExecutionHandler`中的方法
  - 线程池提供了4种策略：
    1. AbortPolicy：直接抛出异常，默认策略；
    2. CallerRunsPolicy：用调用者所在的线程来执行任务；
    3. DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
    4. DiscardPolicy：直接丢弃任务；

## 执行流程

当使用execute执行Runnable方法时候：

```java
	public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
    	//当前工作线程小于corePoolSize 则启用新线程开始当前任务。
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
    	//如果线程池处于RUNNING状态，且把提交的任务成功放入阻塞队列中
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //再次检查线程池的状态，如果线程池没有RUNNING，且成功从阻塞队列中删除任务
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //则执行reject方法处理任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
    	//创建新的线程执行任务，如果执行失败，则执行reject方法处理任务
        else if (!addWorker(command, false))
            reject(command);
	}

	private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    //判断当前需要创建的线程是否为核心线程
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
		//开始创建新的线程
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            final ReentrantLock mainLock = this.mainLock;
            //work内会通过factory创建一个线程   Work是个关键地方后面会分析。
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                mainLock.lock();
                try {
                    int c = ctl.get();
                    int rs = runStateOf(c);

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //把Woker实例插入到HashSet
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();//并启动Woker中的线程
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
 	}
```



1. 如果线程池中的线程数量**小于**corePoolSize，即使线程池中有空闲线程，也会创建一个新的线程来执行新添加的任务。
2. 如果线程池中的线程数量**大于等于** corePoolSize
   1. 缓冲队列workQueue未满，则将新添加的任务放到workQueue中，按照FIFO的原则依次等待执行（线程池中有线程空闲出来后依次将缓冲队列中的任务交付给空闲的线程执行）。
   2. 缓冲队列workQueue已满
      1. 但线程池中的线程数量小于maximumPoolSize，则会创建新的线程来处理被添加的任务。
      2. 如果线程池中的线程数量等于了maximumPoolSize，执行丢弃处理。

**小结**:当有新的任务要处理时，先判断线程池中的线程数量是否大于`corePoolSize`，再看缓冲队列`workQueue`是否满，最后看线程池中的线程数量是否大于`maximumPoolSize`。

![工作流程](/images/threadpool/工作流程.jpg)

下面一起看下核心方法**runWorker **具体的执行流程是怎么样的。 

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        //获取第一个任务firstTask
        Runnable task = w.firstTask;
        w.firstTask = null;
    	//通过unlock方法释放锁
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            //如果第一个任务firstTask为空，则从阻塞队列中获取等待的任务
            while (task != null || (task = getTask()) != null) {
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        //执行任务的run方法
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

从阻塞队列中获取等待的任务**getTask**方法

```java
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
            // 队列为空则直接返回null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
            boolean timed;      // Are workers subject to culling?
            for (;;) {
                int wc = workerCountOf(c);
                timed = allowCoreThreadTimeOut || wc > corePoolSize;
                if (wc <= maximumPoolSize && ! (timedOut && timed))
                    break;
                if (compareAndDecrementWorkerCount(c))
                    return null;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
            try {
                //根据timed在阻塞队列上超时等待或者阻塞等待任务
                Runnable r = timed ?
                    //如果在keepAliveTime时间内，阻塞队列还是没有任务，则返回null
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```



# 使用方法

## 创建

```java
threadsPool = new ThreadPoolExecutor(corePoolSize, maximumPoolSize,
	keepAliveTime, milliseconds,runnableTaskQueue, threadFactory,handler);
```

`Exectors`工厂类提供了线程池的初始化接口，主要有如下几种：

-  `newFixedThreadPool`：初始化一个指定线程数的线程池，其中corePoolSize == maximumPoolSize
-  `newCachedThreadPool`：初始化一个可以缓存线程的线程池，默认缓存60s，线程池的线程数可达到Integer.MAX_VALUE，即2147483647，内部使用SynchronousQueue作为阻塞队列
-  `newSingleThreadExecutor`：初始化的线程池中只有一个线程，如果该线程异常结束，会重新创建一个新的线程继续执行任务，唯一的线程可以保证所提交任务的顺序执行
-  `newScheduledThreadPool`：初始化的线程池可以在指定的时间内周期性的执行所提交的任务，在实际的业务场景中可以使用该线程池定期的同步数据。

## 提交任务

通过`execute`提交任务

```JAVA
threadsPool.execute(new Runnable() {
	public void run() {
		// TODO Auto-generated method stub
	}
});
```

使用`submit` 方法来提交任务，它会返回一个`future`,那么我们可以通过这个future来判断任务是否执行成功

```java
try {
	Object s = future.get();
} catch (InterruptedException e) {
    
} catch (ExecutionException e) {
    
} finally {
	// 关闭线程池
    executor.shutdown();
}
```

## 线程池的关闭

我们可以通过调用线程池的`shutdown`或者`shutdownNow`方法来关闭线程池。他们之间的不同

`shutdown`：只是将线程池的状态设置成 *SHUTDOWN* 状态，然后中断所有没有正在执行任务的线程。

`shutdownNow`：遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。

# 实战相关问题

## 线程池大小的估算

合理的配置线程池，就必须首先分析任务特性，可以从以下几个角度来进行分析：

1. 任务的性质: CPU密集型任务，IO密集型任务和混合型任务。
2. 任务执行时间的长短：长，中和短。
3. 任务的优先级：高，中和低。

CPU密集型任务配置尽可能少的线程数量，如配置Ncpu+1个线程的线程池。

IO密集型任务则由于需要等待IO操作，线程并不是一直在执行任务，则配置尽可能多的线程，如2*Ncpu。

**线程等待时间所占比例越高，需要越多线程。线程CPU时间所占比例越高，需要越少线程。**



