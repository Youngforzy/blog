---
title: Java并发之Executor框架
date: 2017-11-22 16:42:18
tags: [并发,Executor框架]
categories: 技术
---
### 一、Executor框架的介绍

Java中的线程既是工作单元又是执行机制。JDK1.5开始，把工作单元与执行机制分离开来。**工作单元为Runnable（Thread实现该接口）和Callable，执行机制就是Executor框架。** 使用Executor框架时不用显式的创建线程Thread。

Executor框架由三部分组成：  
- **任务**：Runnable或Callable  
- **任务的执行**：ExecutorService接口及其实现。  
- **异步计算的结果**：Future接口或其实现类FutureTask


#### Executor的结构

![image](http://osuskkx7k.bkt.clouddn.com/Executor1.png)



#### **Executor接口**
```
public interface Executor {
        void execute(Runnable command);
}
```
**Executor接口中只有一个execute()方法，用来执行已经提交的Runnable实例，可见即使是Callable实例，最后也会被封装成Runnable来执行。**




#### **ExecutorService接口**


```
public interface ExecutorService extends Executor {
        void shutdown();
        List<Runnable> shutdownNow();
        <T> Future<T> submit(Callable<T> task);
        <T> Future<T> submit(Runnable task, T result);
        Future<?> submit(Runnable task);
        ....
    }
```
**ExecutorService扩展了Executor接口，添加了许多方法用于服务、管理和关闭线程池。**
submit()方法最终执行时也是调用了execute()方法。

ExecutorService接口有两个实现类，ThreadPoolExecutor（核心）和ScheduledThreadPoolExecutor（定时执行）。


#### **Executors工具类**

**Executors工具类中包含了许多静态工厂方法。采用了多方法静态工厂模式。** 本质是根据不同的输入创建出不同类型的对象。



通过Executors工具类可以创建3种类型的线程池，即3种ThreadPoolExecutor对象。实质是创建ThreadPoolExecutor时传入的参数不同。


### 二、3种常用线程池

#### FixedThreadPool
**FixedThreadPool是固定大小的线程池。内部线程可重用。**  
Executors工具类中的静态方法：
```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

**特点**：
1. 参数corePoolSize和maximumPoolSize大小都是nThreads，说明最大线程数就是核心线程数，所以线程大小固定
2. 参数keepAliveTime为0L，说明不会有空闲的线程
3. 参数BlockingQueue是无界队列LinkedBlockingQueue，说明任务会一直放入，不会采用饱和策略。

**应用场景**：需要限制线程数量，适用于负载较重的服务器

![image](http://osuskkx7k.bkt.clouddn.com/fixed.png)

**工作过程**：
1. 当前线程数小于corePoolSize，创建新线程执行任务
2. 当前线程数等于corePoolSize，任务加入阻塞队列
3. 线程反复执行阻塞队列中的任务

#### SingleThreadExecutor

**SingleThreadExecutor是只有一个线程的线程池。**


```
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
**特点**：
1. 参数corePoolSize和maximumPoolSize大小都是1，说明线程数固定为1
2. 参数keepAliveTime为0L，说明不会有空闲的线程
3. 参数BlockingQueue是无界队列LinkedBlockingQueue，说明任务会一直放入，不会采用饱和策略。

**应用场景**：**适用于执行的任务需要保证顺序；并且在任意时间点，不会有多个线程是活动的场景。**

![image](http://osuskkx7k.bkt.clouddn.com/single1.png)

**工作过程**：
1. 当前线程数小于1，创建一个唯一的线程执行任务
2. 当前线程数等于1，任务加入阻塞队列
3. 这个唯一的线程反复执行阻塞队列中的任务

#### CachedThreadPool
**CachedThreadPool是一个根据需要创建线程的线程池。**


```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
**特点**：
1. 参数corePoolSize为0，maximumPoolSize为Integer.MAX_VALUE（2147483647），说明可创建的线程数巨大，且都是可销毁的
2. 参数keepAliveTime为60L，说明空闲的线程等待时间最长60s
3. 参数BlockingQueue是一个没有容量的阻塞队列SynchronousQueue，说明任务会一直被线程执行。

**应用场景**：**大小无界的线程池，适用于执行很多的短期异步任务的小程序，或者负载较轻的服务器。  
注：使用时需控制并发的任务数，否则创建大量的线程可能导致严重的性能问题。**

![image](http://osuskkx7k.bkt.clouddn.com/cached1.png)

**工作过程**：
1. 首先执行SynchronousQueue.offer()，如果有空闲的线程在执行SynchronousQueue.poll()，表示配对成功，任务交给空闲线程执行。
2. 初始化时，如果没有空闲的线程，那么创建一个新的线程执行任务。
3. 步骤2中的线程任务完成后，会执行SynchronousQueue.poll()等待60s，若没有任务提交，则该空闲线程销毁。
 
SynchronousQueue队列的每个插入操作都要等待一个移除操作，因此是没有容量的队列。


---

除上述3种常用线程池外，Executors还可以创建以下几种线程池。


**newScheduledThreadPool**：可以定时或周期性执行任务的线程池（线程数目指定）

**newSingleThreadScheduledExecutor：** 可以定时或周期性执行任务的线程池。只有一个线程。
