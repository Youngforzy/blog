---
title: Java并发工具类之CyclicBarrier
date: 2017-12-8 18:42:18
tags: [并发,CyclicBarrier]
categories: 技术
---
### 一、CyclicBarrier的概念

CyclicBarrier的意思是可循环使用的屏障。**它可以让一组线程到达一个屏障时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有阻塞的线程才会继续执行。**

它就好像一道关卡，只有所有的部队（线程）都到了才能放行。

![image](http://osuskkx7k.bkt.clouddn.com/cycle.png)


### 二、CyclicBarrier的实现分析

**部分源码：**
```
public class CyclicBarrier {
    
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition trip = lock.newCondition();
    private final int parties;
    private final Runnable barrierCommand;
    private Generation generation = new Generation();
    private int count;
    private static class Generation {
        boolean broken = false;
    }
    ....
}
```
可以看到，**CyclicBarrier是基于ReentrantLock和Condition实现的。**
- parties 表示拦截线程的数量
- barrierCommand 表示所有线程到达屏障后优先执行的命令
- Generation 表示屏障循环利用

#### 构造函数
CyclicBarrier有两个构造函数：

```
public CyclicBarrier(int parties) {
        this(parties, null);
    }
public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```
第一个构造函数调用的其实也是第二个构造函数，只是第二个参数barrierAction为null。这个参数其实是一个线程任务命令，用于在所有线程到达屏障时，优先执行该线程任务，方便处理更加复杂的业务场景。

#### await()方法：
每当一个线程调用await()方法表示该线程到达屏障，

```
public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); 
        }
    }
```
进入dowait()方法：

```
private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;
            
            //当前generation“已损坏”，抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
                
            //如果线程中断，终止CyclicBarrier
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            //每当线程进入，计数-1
            int index = --count;
            if (index == 0) {  //计数为0时，进入
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)   //如果有barrierCommand，则优先执行该任务
                        command.run();
                    ranAction = true;
                    nextGeneration();//唤醒所有等待线程，并更新generation
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }
            // loop until tripped, broken, interrupted, or timed out
            //自旋
            for (;;) {
                try {
                    if (!timed)//如果不是超时等待，则调用Condition.await()方法等待
                        trip.await();
                    else if (nanos > 0L)//超时等待，调用Condition.awaitNanos()方法等待
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    //如果等待过程中，线程被中断，则执行下面的函数。
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                    }
                }
                
                //当前generation“已损坏”，抛出异常
                if (g.broken)
                    throw new BrokenBarrierException();
                    
                //generation已经更新，返回index
                if (g != generation)
                    return index;

                //“超时等待”，并且时间已到,终止CyclicBarrier，并抛出异常
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```
dowait()的主要处理逻辑如下：如果该线程不是到达的最后一个线程，则它会一直处于等待状态，除非发生以下情况：
1. 最后一个线程到达，即index == 0
2. 超出了指定时间（超时等待）
3. 其他的某个线程中断当前线程
4. 其他的某个线程中断另一个等待的线程
5. 其他的某个线程在等待barrier超时
6. 其他的某个线程在此barrier调用reset()方法。reset()方法用于将屏障重置为初始状态。

Generation对象描述着CyclicBarrier的更新换代。在CyclicBarrier中，同一批线程属于同一代。当有parties个线程到达barrier，generation就会被更新换代。

### 三、CyclicBarrier与CountDownLatch的对比


**CyclicBarrier允许一系列线程相互等待对方到达屏障，先到达的线程被阻塞在屏障前，必须等到所有线程都到达了屏障，所有线程才能运行；CountDownLatch允许一个或多个线程等待一些特定的操作完成，而这些操作是在其它的线程中进行的，只有“被等的线程”的操作完成后，“等待的线程”才能执行；**

**CyclicBarrier强调的是n个线程互相等待，CountDownLatch强调的是1个线程或n个线程等待其他线程操作。**



**CyclicBarrier的计数器可以循环使用（出现错误可重置计数），CountDownLatch的计数器只能用一次；**

**CyclicBarrier可以在所有线程到达屏障后先执行一个线程任务，再运行所有线程，用于处理复杂的业务，CountDownLatch不可以。**