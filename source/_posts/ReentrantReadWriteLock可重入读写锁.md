---
title: ReentrantReadWriteLock可重入读写锁
date: 2017-10-28 17:48:18
tags: [并发,ReentrantReadWriteLock,可重入读写锁]
categories: 技术
---
### 一、ReentrantReadWriteLock的概念
#### 介绍
前面提到的锁（独占锁、ReentrantLock）等都是排他锁，这些锁在同一时刻只允许一个线程访问。  
而**读写锁在同一时刻可以允许多个读线程访问，但在写线程访问时，所有读线程和其他写线程都阻塞。**


ReadWriteLock并不是继承自Lock接口，而是一个单独的接口。

```
public interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
}
```
ReentrantReadWriteLock则是这个接口的实现。通过readLock()和writeLock()方法可分别获得一个ReadLock实例和一个WriteLock实例，这两个实例实现了Lock接口。
因此，我们可以调用Lock接口的相关方法来完成锁的语义。

```
ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
Lock r = rw.readLock();
Lock w = rw.writeLock();
...
```
#### 特性：
1. **公平性：同样有公平锁和非公平锁；**  

2. **重入性：读锁和写锁都支持重入（最大65535）；**
3. **锁降级：获取写锁之后，获取读锁，释放写锁，保留读锁；（按顺序）**

### 二、ReentrantReadWriteLock的实现原理

ReentrantReadWriteLock与ReentrantLock一样，锁的语义的实现依旧是依靠Sync（继承自AQS），它的读锁、写锁的实现原理如下：
- **读锁：AQS共享锁**
- **写锁：AQS独占锁**


#### 读写状态
读写锁的实现同样是依赖AQS来实现同步功能。
那么它的读写状态如何表示呢？
同样是**使用一个int型的变量**表示，将这个变量“按位切割”成两部分，**高16位表示读，低16位表示写**。这样我们就能通过位运算确定它的读写状态。（如下图）  

![（图）](http://osuskkx7k.bkt.clouddn.com/duxiesuo.jpg)  

如果已知整体同步状态为S，那么：
- **写状态：S & 0x0000FFFF**（将高16位变0，抹去）
- **读状态：S>>>16** （无符号补0右移16位）  

**注：当写状态为0，S不为0时，表示读状态不为0，读锁被获取。**



#### 写锁的获取和释放

写锁是独占锁，获取时调用Sync中的tryAcquire()方法：
```
protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            //获取状态
            int c = getState();
            //获取写状态
            int w = exclusiveCount(c);
            if (c != 0) {
                //写状态为0表示存在读线程，获取失败
                //或当前线程不是获取写锁的线程，获取失败
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //超过写锁总数量
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //当前线程是获取写锁的线程，重进入，获取成功
                setState(c + acquires);
                return true;
            }
            //是否需要阻塞
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```
写锁的获取过程如代码中注释所示。
只有在以下情况才能获取写锁：
- **不存在读锁或当前线程是已经获取写锁的线程（可重入）**

---
写锁的释放调用的是Sync的tryRelease()方法：

```
protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```
写锁的释放与重入锁的释放过程类似，每次释放时将写状态减少，直到写锁状态为0时，表示写锁释放。

---

#### 读锁的获取和释放
读锁是共享锁，调用的是Sync的tryAcquireShared()方法：

```
protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState(); //获取状态
            //写锁不为0  && 且获取写锁的线程不是当前线程
            //直接失败
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current) 
                return -1;
            //获取读锁
            int r = sharedCount(c);
            //readerShouldBlock()：读锁是否要等待（公平or非公平）
            // r < MAX_COUNT：读锁小于最大值（65535）
            //compareAndSetState(c, c + SHARED_UNIT))：CAS操作成功
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                //r == 0：只有一个读锁（A），计数+1
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                //持有读锁的线程（A）重进入，计数++
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                //另一个线程（B）进入，此时找到缓存的rh，将计数++；
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            //循环尝试
            return fullTryAcquireShared(current);
        }
```
获取锁的过程如注释所示。  
如果不满足第二个if语句中的判断，比如读锁需要阻塞，则会进入fullTryAcquireShared（current）方法，**该方法循环不断尝试修改状态直到成功或被写入锁占有。**
    

```
final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                //写锁存在但不是当前线程，直接失败
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                //读锁是否要阻塞（公平 or 非公平）
                } else if (readerShouldBlock()) {
                    if (firstReader == current) {
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                //读锁达到最大值，不能再获取
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //CAS操作
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; 
                    }
                    return 1;
                }
            }
        }
```
以上的代码中多次出现了一个rh变量（HoldCounter），我们知道重入锁的原理就是计数器，同理这个rh变量也相当于一个计数器，记录线程获取读锁的次数。来看它的定义：

```
//HoldCounter类
static final class HoldCounter {
            int count = 0;
            final long tid = getThreadId(Thread.currentThread());
        }
//继承ThreadLocal类        
static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
```
HoldCounter的定义只包含一个计数器和当前线程的Id，它的作用就是记录该线程获取读锁的次数，那么它是如何与线程绑定的呢？我们知道ThreadLocal类是线程维护的私有变量，利用它就可以和线程绑定。  

注：（需要说明的是这样**HoldCounter绑定线程id而不绑定线程对象的原因是****避免HoldCounter和ThreadLocal互相绑定而GC难以释放它们**，所以其实这样做只是为了帮助GC快速回收对象而已。）

---

当读锁释放时，调用的是Sync的tryReleaseShared()方法：

```
protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```
循环CAS操作直到读锁的状态为0。
#### 锁降级
前面提到，读写锁有个特性是锁降级。  
锁降级指的是：**写锁降级为读锁**。

具体过程：**获取写锁的线程把持住写锁，然后获取读锁，再释放写锁。**  

**目的：保证写锁修改的数据可以被其他线程看见，保证了数据的可见性。** 


锁降级中读锁的获取是否为必要？肯定是必要的。试想，假如当前线程A不获取读锁而是直接释放了写锁，这个时候另外一个线程B获取了写锁，那么这个线程B对数据的修改是不会对当前线程A可见的。   如果获取了读锁，则线程B在获取写锁过程中判断如果有读锁还没有释放则会被阻塞，只有当前线程A释放读锁后，线程B才会获取写锁成功。
