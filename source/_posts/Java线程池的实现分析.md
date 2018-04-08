---
title: Java线程池的实现分析
date: 2017-11-18 20:42:18
tags: [并发,Java线程池]
categories: 技术
---
### 一、线程池的介绍

线程池是一种并发框架。

优势：
1. **降低资源消耗。**（重复利用线程，减少开销）
2. **提高响应速度。**（任务到达可直接执行，不需要等待创建线程）
3. **提高线程的可管理性**。（统一分配、监控、调优）

**ThreadPoolExecutor是线程池的核心实现类**。可以通过ThreadPoolExecutor来创建一个线程池。


### 二、线程池的实现分析

线程池的实现是ThreadPoolExecutor类，因此重点描述ThreadPoolExecutor类的实现。  

#### ThreadPoolExecutor的结构

ThreadPoolExecutor的构造方法
```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        ...(省略部分)
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
构造函数中省略了部分判断的代码。  
主要有7个参数：  

**corePoolSize**：核心线程池的大小  

**maximumPoolSize**：线程池的大小  

**keepAliveTime**：存活时间（超过核心数目的线程空闲后的存活时间）  

**TimeUnit**：时间单位  

**BlockingQueue<Runnable>**：任务队列（保存等待任务的阻塞队列）  

**ThreadFactory**：创建线程的工厂类   

**RejectedExecutionHandler** ：饱和策略（拒绝策略）


![image](http://osuskkx7k.bkt.clouddn.com/%E7%BA%BF%E7%A8%8B%E6%B1%A0.png)


#### 工作过程


当提交一个新任务时，线程池的工作过程：
1. **判断核心线程池（corePool）中的线程是否都在执行任务。如果不是，创建一个新的线程执行任务。核心线程池已满，进入2**；  
 
2. **判断任务队列是否已满。未满，则将新的任务存入；满了，进入3；**
3. **判断线程池（maximumPoolSize）里的线程是否都在工作。如果没有，创建一个新的线程执行任务；否则，交给饱和策略4**；
4. **根据不同的饱和策略处理这个任务**。


![image](http://osuskkx7k.bkt.clouddn.com/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B.png)

饱和策略有4种：
- **AbortPolicy（默认）**：直接抛出异常
- **CallerRunsPolicy**：只用调用者所在线程来处理
- **DiscardOldestPolicy**：丢弃任务队列中最后一个任务，执行当前任务
- **DiscardPolicy**：不处理丢弃掉

       
     
线程池回收线程时，对所谓的“核心线程”和“非核心线程”是一视同仁的，直到线程池中线程的数量等于corePoolSize参数时，回收过程才会停止。  
如果设置的corePoolSize参数和maximumPoolSize参数一致时，线程池在任何情况下都不会回收空闲线程。keepAliveTime和timeUnit也就失去了意义。  
可以调用以下方法回收核心线程。
```
threadPoolExecutor.allowCoreThreadTimeOut(true);
```





#### 线程池处理任务
线程池处理任务的方法主要有两种，execute()和submit()。

**execute()**  

**execute()方法用于提交不需要返回值的任务**，Runnable实例。所以无法判断任务是否被线程池执行成功。

**submit()**  
**submit()方法用于提交需要返回值的任务**。线程池会返回一个future类型的对象，通过这个对象可以判断任务是否执行成功。

future的get()方法会阻塞当前线程直到任务完成，返回结果。

submit()最终调用的也是execute(Runnable runable)，submit()只是将Callable或Runnable封装成一个FutureTask对象，因为FutureTask是个Runnable，所以调用的是execute()方法。

#### 线程池的关闭
线程池关闭的方法主要有两种，shutdown()和shutdownNow()。

**原理**：遍历线程池中的工作线程，逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法停止。

**区别**：  
**shutdown()只是将线程池的状态设置成SHUTDOWN状态，然后中断没有在执行任务的线程。**

**shutdownNow()首先将线程池的状态设置成STOP，然后尝试停止所有正在执行或暂停任务的线程，并返回等待执行任务的列表。**


如果任务不一定要执行完，可以调用shutdownNow()方法。