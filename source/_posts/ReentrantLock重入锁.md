---
title: ReentrantLock重入锁
date: 2017-10-20 19:48:18
tags: [并发,ReentrantLock,重入锁]
categories: 技术
---
### 一、ReentrantLock概念
#### 重入锁概念
什么是重入锁？   顾名思义，就是支持重复进入的锁。  
**定义：支持一个线程对资源的重复加锁。（注意是一个线程）**

回忆前面有关AQS实现的文章，关于独占锁，当一个线程获取锁后，如果该线程再次调用lock()方法，那么该线程会被自己阻塞。  
原因是在实现tryAcquire()时没有考虑占有锁的线程再次获取锁的场景。因此这个锁是不支持重入的锁。  

回顾synchronized的实现原理，我们知道它也是支持重进入的锁，即可以多次获取锁。

那么Lock能不能实现这个功能呢？当然是可以的。  
ReentrantLock就是一个支持重进入的锁：**在调用lock()方法时，已经获取到锁的线程，可以再次调用lock()方法获取锁而不被阻塞。**
#### 公平与非公平
关于锁的获取，还有一个公平性的问题，于是就有了公平锁与非公平锁：  

-   **公平锁：获取锁是顺序的。先对锁请求的线程先获取；**  
-   **非公平锁：获取锁是无序的。任意线程都可以获取，无关请求先后；**

来看看ReentrantLock的构造函数：

```
public ReentrantLock() {
        sync = new NonfairSync();
    }
public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```


可以看到，ReentrantLock默认构造函数是非公平锁，另一个构造函数：  
-   传入true：公平锁  
-   传入false：非公平锁



### 二、ReentrantLock的实现原理
#### 重进入

**重进入的实现原理是定义了一个获取锁的计数器**。 涉及到以下两个问题：  

1. **线程再次获取锁。**  
锁需要识别获取锁的线程是否为当前占据锁的线程，如果是，则再次获取锁成功，计数器+1。
2. **锁的最终释放。**  
线程重复n次获取了锁，在第n次释放锁后，其他线程能够获取到锁。释放锁时，计数器-1，计数器为0时表示锁释放成功。

---

**ReentrantLock**
```
public class ReentrantLock implements Lock, java.io.Serializable {
private final Sync sync;
//AQS
abstract static class Sync extends AbstractQueuedSynchronizer {...};
//非公平锁
static final class NonfairSync extends Sync {...};
//公平锁
static final class FairSync extends Sync {...};

....(省略)
}
```
从源码可以看到，**ReentrantLock的实现依旧是依靠队列同步器AQS（Sync继承自AQS），不同的是这里有两个AQS的实现类，NonfairSync和FairSync，分别实现非公平锁和公平锁的功能。**


#### 非公平锁的实现

非公平锁获取锁的方法：
```
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {  //判断
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
该方法是Sync中的方法，NonfairSync通过重写tryAcquire()调用。  
可以看到，该方法增加了一个判断：如果当前线程是已经获取锁的线程，那么将同步状态值State增加 **（这里不需要CAS操作，因为该线程已经获取了锁，没有竞争——相当于偏向锁）**，并返回true，表示同步状态获取成功。


非公平锁释放锁的方法：

```
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```
该方法也是由Sync实现。

既然获取锁的时候增加了同步状态值，那么释放时自然要减少。
可以看到，只有当State值减为0的时候，才返回true，表示释放锁成功，并将占有线程设置为null。


#### 公平锁的实现

公平锁获取锁的方法：

```
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```
该方法是FairSync重写的tryAcquire()方法。  
对比非公平锁的获取，**唯一不同的是该方法在判断时增加了一个条件——hasQueuedPredecessors()方法，该方法判断同步队列中当前节点是否有前驱节点，如果有，表示有线程比当前线程更早请求，则需要等待前面的线程获取释放锁之后才能获取锁。**

    
公平锁释放锁的方法：  

同样也是调用Sync中的tryRelease()方法，这里不再赘述。


#### 公平锁与非公平锁的对比

**公平锁：**
- 保证了锁的获取顺序，FIFO原则
- 不足是需要进行大量的线程切换

**非公平锁**：  

- 保证了更大的吞吐量（极少的线程切换）
- 不足是可能造成线程“饥饿”（等待很久）

**频繁的线程切换对性能有很大的影响，因此ReentrantLock的默认实现是非公平锁。**