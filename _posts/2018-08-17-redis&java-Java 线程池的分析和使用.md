---
layout:     post
title:      Java 线程池的分析和使用
subtitle:   源码分析
date:       2018-08-17
author:     薛睿
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - 多线程
    - java
---

## 一、关于线程池的几个重要接口和类
<div align=center><img src="https://s2.ax1x.com/2019/07/26/enuHgJ.png"></div>

1.Executor接口是一个顶层接口，只声明了一个方法：void execute(Runnable command)，用来执行传进去的可执行任务。

2.ExecutorService接口继承自Executor接口，声明了void shutdown()、List<Runnable> shutdownNow()、boolean isShutdown()、boolean isTerminated()、submit()、invokeAll()、invokeAny()等方法。

3.AbstractExecutorService抽象类实现了ExecutorService接口，实现了其中声明的大部分方法。

4.ThreadPoolExecutor类继承自AbstractExecutorService抽象类，实现了线程池。其中有几个重要的方法execute()、submit()、shutdown()、shutdownNow()。

## 二、 从源代码剖析线程池的实现原理
我们从ThreadPoolExecutor的构造方法开始看起，有以下四个构造方法：
```java
//构造方法一
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
 
//构造方法二
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
 
//构造方法三
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }
 
//构造方法四
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
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
我们挑选其中参数最全的一个进行参数的说明:
- **corePoolSize：** 线程池核心线程数，创建线程池后默认是空的，其中没有任何线程，等待有任务到来时才创建线程执行任务。（除非调用restartAllCoreThreads()或者prestartCoreThread()方法后会预先创建核心线程数个线程），当Threadpool中的线程数达到核心线程数后仍有任务到达时，便会把任务先放入缓存队列中。
- **maximumPoolSize：** 线程池线程的最大数，当缓存队列已满且有新的任务需要处理时，判断线程池中的线程数是否到了这个最大数，如果没有，则创建一个新的线程来执行该任务。负责提交给饱和策略来处理。
- **keepAliveTime：**  空闲线程存活时间，当线程池中的线程数量大于核心线程数时，若线程池中的某一空闲线程的存活时间超过了空闲线程存活时间就会被清理，直至线程池中的线程数目不大于corePoolSize。
- **unit：** 时间单位
TimeUnit.DAYS;               //天
TimeUnit.HOURS;             //小时
TimeUnit.MINUTES;           //分钟
TimeUnit.SECONDS;           //秒
TimeUnit.MILLISECONDS;      //毫秒
TimeUnit.MICROSECONDS;      //微妙
TimeUnit.NANOSECONDS;       //
- **workQueue：** 线程池所使用的缓冲队列,线程池的排队策略与其有关。
ArrayBlockingQueue;（有界队列FIFO）
LinkedBlockingQueue;（无界队列）
SynchronousQueue;（直接提交队列）
PriorityBlockingQueue;（优先队列，为任务实现Comparable接口，实现任务的优先调度）
- **threadFactory：** 线程池创建线程时使用的线程工厂，线程工厂的接口定义如下，只有一个创建线程的方法：
    ```java
    public interface ThreadFactory {
 
        /**
         * Constructs a new {@code Thread}.  Implementations may also initialize
         * priority, name, daemon status, {@code ThreadGroup}, etc.
         * @param r a runnable to be executed by new thread instance
         * @return constructed thread, or {@code null} if the request to
         * create a thread is rejected
         */
    Thread newThread(Runnable r);
    }
    ```
    DefaultThreadFactory实现了ThreadFactory接口， 重写了newThread方法，在创建线程时给定group,统一的线程前缀名以及优先级等。在更多的情况下，ThreadFactory需要我们自己去实现接口来满足我们的需求。
    ```java
    static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;
 
        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }
 
        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
    ```
- **handler：** 线程池对拒绝任务的处理策略
    hreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常，defaultHandler便是它。
    ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
    ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）。
    ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务。

在创建线程池后线程池处于RUNNING状态，调用shutdown方法后，线程池会进入SHUTDOWN状态，拒绝新任务的接收，等待所有任务的处理完毕。而如果在running状态调用shutdownNow方法，线程池会进入STOP状态，拒绝接受新的任务，并且去尝试关闭正在运行的线程。当处于SHUTDOWN和STOP状态的线程池中所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为TERMINATED状态。

线程池中最重要的方法是execute方法，而submit方法也是调用了execute方法，接下来看看execute是怎么实现的：
```java
public void execute(Runnable command) {
        //如果command是空对象，则抛出空指针异常
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //当前池中线程比核心数少，新建一个线程执行任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //核心池已满，但任务队列未满，添加到队列中
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))  //如果这时被关闭了，拒绝任务
                reject(command);
            else if (workerCountOf(recheck) == 0) //如果之前的线程已被销毁完，新建一个线程
                addWorker(null, false);
        }
        //核心池和队列都已满，创建新的线程
        else if (!addWorker(command, false))
            reject(command);
    }
```
AbstractExecutorService类中实现的submit方法也是调用execute方法实现，区别在于execute没有返回值，而submit方法会返回一个Future方法。

## 三、初始化线程池的工厂方法

通过构造函数初始化线程池参数众多，比较麻烦，而且对于我这样的新手来说合理的配置线程池的参数更加摸不着头脑，JDK中提供了Executors类来实现线程池的便捷初始化。
<div align=center><img src="https://s2.ax1x.com/2019/07/26/enQI4f.png"></div>
1.newFixedThreadPool:初始化一个指定线程数的线程池，其中corePoolSize和maximumPoolSize的数量相等，keepAliveTime为0,使用LinedBlockingQueue作为任务缓存队列。特点是线程池中的线程数目始终保持不变。
```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
2.newCachedThreadPool：初始化一个可以缓存线程的线程池，默认缓存60s，线程池的线程数可达到Integer.MAX_VALUE，即2147483647，内部使用SynchronousQueue作为阻塞队列。
```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
3.newScheduledThreadPool():初始化的线程池可以在指定的时间内周期性的执行所提交的任务，在实际的业务场景中可以使用该线程池定期的同步数据。
```java
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));
    }
```
## 四、线程池的工厂化方法使用演示
以newFixedThread方法为例，其他的工厂化线程池类似。
```java
class MyThread extends Thread {
    @Override
    public void run() {
 
        System.out.println(Thread.currentThread().getName() + "正在执行。。。");
 
    }
 
}
public class test {
    public static void main(String[] args)
    {
        ExecutorService pool= Executors.newFixedThreadPool(5);
        Thread[] threads=new Thread[5];
        for(int i=0;i<5;i++)
        {
            threads[i]=new MyThread();
        }
        for(int i=0;i<5;i++)
        {
            pool.execute(threads[i]);
        }
 
    }
}
```
执行结果：
```
pool-1-thread-2正在执行。。。
pool-1-thread-5正在执行。。。
pool-1-thread-4正在执行。。。
pool-1-thread-1正在执行。。。
pool-1-thread-3正在执行。。。
```









