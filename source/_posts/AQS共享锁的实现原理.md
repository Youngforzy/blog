---
title: AQS共享锁的实现原理
date: 2017-10-19 19:48:18
tags: [并发,AQS,共享锁]
categories: 技术
---
### 一、AQS共享锁的实现原理

前面的文章Lock的实现中分析了AQS独占锁的实现原理，那么接下来就分析下AQS是如何实现共享锁的。

#### 共享锁的介绍
**共享锁：同一时刻有多个线程能够获取到同步状态。**

那么它是如何做到让多个线程获取到同步状态呢？  
来看一下获取共享锁的过程：
1.  线程调用AQS的acquireShared()申请获取锁（可有多个线程获取到，根据重写的tryAcquireShared()方法决定），如果成功则进入临界区。
2.  如果失败，创建一个共享型的节点进入FIFO等待队列，阻塞然后等待唤醒。
3.  等待队列中的线程被唤醒重新尝试获取锁，**获取成功后根据state变量值决定是否继续唤醒后续节点（如果state值为0，表示没有可用的锁，不唤醒后继节点；如果state的值>0，表示有可用的锁，唤醒后继节点）**，获取失败则继续等待，直到成功。

释放共享锁的过程：
1. 线程调用releaseShared()进行锁资源释放，如果释放成功则唤醒队列中等待的节点（如果有）。

#### 共享式获取锁

线程调用acquireShared()方法获取锁：
```
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg); //获取失败进入该方法
    }
```
分析如下：
- 当tryAcquireShared(arg)返回值>=0时（可以在重写该方法时自定义锁的数量），表示获取锁成功，不会进入doAcquireShared。
- 当tryAcquireShared(arg)返回值<0时,进入doAcquireShared(arg)方法，可以猜想这里应该是构造节点放入等待队列，看如下代码：

```
private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);  //构造等待队列，和独占锁类似
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {       //自旋
                final Node p = node.predecessor(); //获取前驱节点
                if (p == head) {
                    int r = tryAcquireShared(arg); //再次尝试获取
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
可以看到，当前驱节点是头节点head时，线程尝试获取锁，此时注意返回值r，有以下三种可能：

- r<0，表示获取锁失败，继续自旋直到r>=0；
- r=0，表示获取锁成功，但刚好是最后一把锁，不会唤醒后继节点，在setHeadAndPropagate(node, r)方法中可以体现出来，后面会分析到；
- r>0，表示获取锁成功，而且还有锁资源，会唤醒后继节点，同样在setHeadAndPropagate(node, r)方法中可以体现。  

那么就来看一下setHeadAndPropagate(node, r)这个方法：

```
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // 记录原来的头节点
        setHead(node);  // 将当前节点设置为头节点
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```
注意这里propagate的值就是上一个方法中的r，首先将当前节点设置为头节点，然后if中的判断表示以下两种情况需要执行唤醒操作：

1.  根据r的值判断，r>0时，表示可以唤醒后继节点，执行doReleaseShared()方法；而当r=0时，不会直接执行doReleaseShared()方法，而是进入第二种情况继续判断；
2.  头节点后面的节点需要被唤醒（waitStatus<0），不论是老的头结点还是新的头结点

接下来看看doReleaseShared()这个方法：

```
private void doReleaseShared() {
        for (;;) { //自旋
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) { //表示后继节点需要被唤醒
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h); //唤醒
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;    //如果后继节点暂时不需要唤醒，则把当前节点状态设置为PROPAGATE确保以后可以传递下去
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
这里有两个入口可以进入该方法，一个是直接释放锁releaseShared()一个是上述setHeadAndPropagate()方法，因此在释放锁的过程中需要使用CAS操作保证线程安全。
1. 进入第一个if语句，表示后继节点需要被唤醒，采用CAS循环操作直到成功；
2. 进入else if语句，表示暂时不需要唤醒，将状态传递；
3. 最后判断头节点是否变化，没有变化则退出循环；如果头结点发生变化，比如说其他线程获取到了锁，为了使自己的唤醒动作可以传递，必须进行重试

以上就是获取共享锁的大致过程。


#### 共享式释放锁

调用releaseShared()方法主动释放锁：
```
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
可以看到，当重写的tryReleaseShared(arg)方法返回true,成功释放锁资源，进入doReleaseShared()唤醒等待的线程，这个方法上面已经分析过，这里不再赘述。



### 二、AQS共享锁与独占锁的对比
共享锁的实现稍比独占锁复杂，但大同小异。二者对比如下：
- **独占锁：**  
1. 独占锁是只有头节点获取锁，其余节点的线程继续等待，等待锁被释放后，才会唤醒下一个节点的线程；

2. 同步状态state值在0和1之间切换，保证同一时间只能有一个线程是处于活动的，其他线程都被阻塞，参考ReentranLock。
3. 独占锁是一种悲观锁。 
- **共享锁：**  
1. 共享锁是只要头节点获取锁成功，就在唤醒自身节点对应的线程的同时，继续唤醒AQS队列中的下一个节点的线程，每个节点在唤醒自身的同时还会唤醒下一个节点对应的线程，以实现共享状态的“向后传播”，从而实现共享功能。
2. 同步状态state值在整数区间内（自定义实现），如果state值<0则阻塞，否则不阻塞。参考ReadWriteLock、Semphore、CountDownLautch等。
3. 共享锁是一种乐观锁，允许多个线程同时访问共享资源。