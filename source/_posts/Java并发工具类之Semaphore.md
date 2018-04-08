---
title: Java并发工具类之Semaphore
date: 2017-12-12 17:42:18
tags: [并发,Semaphore]
categories: 技术
---
### 一、Semaphore的概念

**Semaphore又叫信号量，用来控制同时访问特定资源的线程数量**。它通过协调各个线程，以保证合理地使用公共资源。

**Semaphore和CountDownLatch一样，也是维护了一个计数器，其本质是一个共享锁。**

**Semaphore有公平性和非公平性之分。**

**Semaphore的工作过程：**


![image](http://osuskkx7k.bkt.clouddn.com/timg.jpg)  


1. 当一个线程想要访问某个共享资源时，它必须要先获取Semaphore；
2. 当Semaphore > 0 时，获取该资源并使Semaphore – 1；
3. 当Semaphore = 0，则表示全部的共享资源已经被其他线程全部占用，线程必须要等待其他线程释放资源；
3. 当有线程释放资源时，Semaphore+1，其他线程可以争抢资源；


### 二、Semaphore的实现分析

前面分析可知，**Semaphore的实现是共享锁。**

#### 构造函数
Semaphore有两个构造函数。
```
public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```
第一个构造函数中传入的是资源许可的数量，默认是非公平锁。
第二个构造函数传入资源许可的数量和一个boolean变量，该变量可实现公平锁。



Semaphore在使用时有两个主要方法，acquire()方法表示获取一个资源许可，而 release()方法表示释放一个资源许可。

#### 资源获取：acquire()方法

调用acquire()方法获取一个资源：
```
public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```
该方法调用AQS的acquireSharedInterruptibly()方法，以共享的模式获取同步状态：
```
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```
然后调用tryAcquireShared()方法，该方法由Sync的子类来实现：
- 如果是非公平模式，调用NonfairSync的tryAcquireShared()方法；
- 如果是公平模式，调用FairSync的tryAcquireShared()方法。

在前面的文章 [ReentrantLock重入锁](http://blog.csdn.net/babylove_bale/article/details/78317204) 中有提到公平与非公平的实现。


**非公平模式**
```
final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```
remaining 表示剩余的资源许可，如果< 0，表示目前没有剩余的许可。当前线程继续等待。如果remaining >0 则执行CAS操作获取资源许可。


**公平模式**

```
protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```
**在公平模式的方法中，增加了一个判断，判断同步队列中是否有等待的线程：**
- 有，则插入作为尾节点，线程阻塞；
- 没有，则参与资源竞争；

简而言之，公平模式就是要按等待队列中的顺序获取资源许可。
#### 资源释放：release()方法

Semaphore调用release()方法释放资源许可，默认释放1个。
```
public void release() {
        sync.releaseShared(1);
    }
```
调用AQS的releaseShared()方法：
```
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
调用Sync中重写的tryReleaseShared()方法（**公平与非公平都是调用该方法**），
```
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```
**next代表如果许可释放成功，可用资源许可的数量。  
这里可能有多个线程同时释放，因此利用CAS操作将资源许可数量置为next。  
释放成功后，进入doReleaseShared()唤醒队列中等待的线程。**

**注：公平模式与非公平模式都是调用该release()方法。**


