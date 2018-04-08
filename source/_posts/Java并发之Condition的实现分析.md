---
title: Java并发之Condition的实现分析
date: 2017-12-1 15:42:18
tags: [并发,Condition]
categories: 技术
---
### 一、Condition的概念

#### 介绍
回忆synchronized关键字，它配合Object的wait()、notify()系列方法可以实现等待/通知模式。

对于Lock，通过Condition也可以实现等待/通知模式。



Condition是一个接口。  
Condition接口的实现类是Lock（AQS）中的ConditionObject。  
Lock接口中有个 newCondition()方法，通过这个方法可以获得Condition对象（其实就是ConditionObject）。  
因此，**通过Lock对象可以获得Condition对象。**
```
Lock lock  = new ReentrantLock();
Condition c1 = lock.newCondition();
Condition c2 = lock.newCondition();
```



### 二、Condition的实现分析

#### 实现

ConditionObject类是AQS的内部类，实现了Condition接口。

```
public class ConditionObject implements Condition, java.io.Serializable {
        private transient Node firstWaiter;
        private transient Node lastWaiter;
        ...
```
可以看到，等待队列和同步队列一样，使用的都是同步器AQS中的节点类Node。
同样拥有首节点和尾节点，
每个Condition对象都包含着一个FIFO队列。  
结构图：

![image](http://osuskkx7k.bkt.clouddn.com/condition.jpg)

#### 等待

调用Condition的await()方法会使线程进入等待队列，并释放锁，线程状态变为等待状态。
```
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            //释放同步状态（锁）
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //判断节点是否放入同步对列
            while (!isOnSyncQueue(node)) {
                //阻塞
                LockSupport.park(this);
                //如果已经中断了，则退出
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

分析上述方法的大概过程：
1. 将当前线程创建为节点，加入等待队列
2. 释放锁，唤醒同步队列中的后继节点
3. while循环判断节点是否放入同步队列：

- 没有放入，则阻塞，继续while循环（如果已经中断了，则退出）
- 放入，则退出while循环，执行后面的判断
4. 退出while说明节点已经在同步队列中，调用acquireQueued()方法加入同步状态竞争。
5. 竞争到锁后从await()方法返回，即退出该方法。  

![image](http://osuskkx7k.bkt.clouddn.com/enterCon.png)

**addConditionWaiter()方法：**
```
private Node addConditionWaiter() {
            Node t = lastWaiter;
            if (t != null && t.waitStatus != Node.CONDITION) {
                //清除条件队列中所有状态不为Condition的节点
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            //将该线程创建节点，放入等待队列
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

过程分析：同步队列的首节点移动到等待队列。加入尾节点之前会清除所有状态不为Condition的节点。


---
#### 通知
调用Condition的signal()方法，可以唤醒等待队列的首节点（等待时间最长），唤醒之前会将该节点移动到同步队列中。

```
public final void signal() {
            //判断是否获取了锁
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```
过程：
1. 先判断当前线程是否获取了锁
2. 然后对首节点调用doSignal()方法

```
private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```
过程：
1. 修改首节点
2. 调用transferForSignal()方法将节点移动到同步队列


```
final boolean transferForSignal(Node node) {
        //将节点状态变为0   
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        //将该节点加入同步队列
        Node p = enq(node);
        int ws = p.waitStatus;
        //如果结点p的状态为cancel 或者修改waitStatus失败，则直接唤醒
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```
调用同步器的enq方法，将节点移动到同步队列，
满足条件后使用LockSupport唤醒该线程。  

![image](http://osuskkx7k.bkt.clouddn.com/signalcon.png)


---
当Condition调用signalAll()方法：
```
public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }
private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```
可以看到doSignalAll()方法使用了do-while循环来唤醒每一个等待队列中的节点，直到first为null时，停止循环。

一句话总结signalAll()的作用：**将等待队列中的全部节点移动到同步队列中，并唤醒每个节点的线程。**


#### 总结
整个过程可以分为三步：

第一步：一个线程获取锁后，通过调用Condition的await()方法，会将当前线程先加入到等待队列中，并释放锁。然后就在await()中的一个while循环中判断节点是否已经在同步队列，是则尝试获取锁，否则一直阻塞。


第二步：当线程调用signal()方法后，程序首先检查当前线程是否获取了锁，然后通过doSignal(Node first)方法将节点移动到同步队列，并唤醒节点中的线程。


第三步：被唤醒的线程，将从await()中的while循环中退出来，然后调用acquireQueued()方法竞争同步状态。竞争成功则退出await()方法，继续执行。

