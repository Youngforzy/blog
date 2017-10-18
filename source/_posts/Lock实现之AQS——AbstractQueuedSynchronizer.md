---
title: Lock实现之AQS——AbstractQueuedSynchronizer
date: 2017-10-18 17:48:18
tags: [并发,AQS,Lock]
categories: 技术
---
### 一、AQS的介绍
**队列同步器AbstractQueuedSynchronizer（AQS）是构建锁或者其他同步组件的基础框架，是实现Lock的基础。它使用了一个volatile修饰的int变量来表示同步状态，并维护了一个FIFO队列来完成资源获取线程的排队。**


```
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer 
implements java.io.Serializable {
 
 private transient volatile Node head;//头节点
 private transient volatile Node tail;//尾节点
 private volatile int state;          //同步状态
 protected final int getState() {
      return state;
 }
protected final void setState(int newState) {
    state = newState;
}
protected final boolean compareAndSetState(int expect, int update) {
   return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
...
```
从上面AQS的部分代码可以看到，AQS是一个类，它包含了表示同步状态的state变量（volatile修饰）；维护队列的两个引用头节点head和尾节点tail（volatile修饰）；以及提供了三个主要方法，用来保证同步状态的改变是线程安全的；省略了其他方法。

那么AQS是如何实现锁的呢？  
**当我们需要实现锁的时候，首先继承AQS并重写指定的方法，然后将AQS子类组合在自定义组件（锁）的实现中，并调用AQS的模板方法，而这些模板方法将会调用我们重写的方法（模板方法模式）**，这就可以达到我们想要的效果。  
**注：重写指定的方法时需要用到AQS中的三个主要方法来对同步状态进行访问或修改。**

AQS中可重写的方法如下：
```
protected boolean tryAcquire(int arg) {} //独占式获取
protected boolean tryRelease(int arg) {} //独占式释放
protected int tryAcquireShared(int arg) {} //共享式获取
protected boolean tryReleaseShared(int arg) {} //共享式释放
protected boolean isHeldExclusively() {} //判断AQS是否被该线程独占
```
来看一个独占锁的示例。

```
class Mutex implements Lock, java.io.Serializable {
   // 内部类，自定义同步器，继承AQS
   private static class Sync extends AbstractQueuedSynchronizer {
     // 重写方法——是否处于占用状态
     protected boolean isHeldExclusively() {
       return getState() == 1;
     }
     // 重写方法——当状态为0的时候获取锁
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }
     // 重写方法——释放锁，将状态设置为0
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (getState() == 0) throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }
     // 返回一个Condition，每个condition都包含了一个condition队列
     Condition newCondition() { return new ConditionObject(); }
   }
   // 仅需要将操作代理到Sync上即可
   private final Sync sync = new Sync();
   public void lock()                { sync.acquire(1); }
   public boolean tryLock()          { return sync.tryAcquire(1); }
   public void unlock()              { sync.release(1); }
   public Condition newCondition()   { return sync.newCondition(); }
   public boolean isLocked()         { return sync.isHeldExclusively(); }
   public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
   public void lockInterruptibly() throws InterruptedException {
     sync.acquireInterruptibly(1);
   }
   public boolean tryLock(long timeout, TimeUnit unit)
       throws InterruptedException {
     return sync.tryAcquireNanos(1, unit.toNanos(timeout));
   }
 }
```
以上就是利用AQS来实现一个独占锁的示例。  
Mutex是一个自定义的Lock，它在同一时刻只允许一个线程占有锁。**它定义了一个静态内部类继承自AQS，并重写了相应的方法，实现了独占式的获取释放锁。**  
在重写的tryAcquire方法中，**调用CAS方法改变同步状态，因为是原子操作只有一个线程能完成**；在重写的tryRelease方法中将同步状态设为0。  
在使用这个Lock时，我们只要调用Mutex的方法，有关同步的细节都由同步器完成。大大降低了自定义并发组件的门槛。

### 二、AQS的实现原理分析
知道了AQS的用法，那么就来分析下它的实现原理。
**同步器可分为独占式和共享式。** 一般只实现其中一种。这里主要分析独占锁的实现。

#### 同步队列
AQS是依靠内部的同步队列来完成同步状态的管理，当前线程获取同步状态失败时，会将当前线程以及等待状态等信息构造成一个节点（node）加入同步队列，并阻塞当前线程。当同步状态释放时，会把首节点中的线程唤醒，使其尝试获取同步状态。

##### Node类

```
static final class Node {
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
        volatile int waitStatus; //线程的等待状态（上述）
        volatile Node prev; //前驱节点
        volatile Node next; //后继节点
        volatile Thread thread; //线程引用
        Node nextWaiter; //等待队列中的后继节点
        ...
        }
```
Node是AQS维护的静态内部类。用来保存线程引用（失败）、等待状态和前后节点。  
节点是构成同步队列的基础，同步器拥有首节点（head）和尾节点（tail），获取同步状态失败的线程会成为节点加入队列的尾部。同步器结构如下： 
![image](http://osuskkx7k.bkt.clouddn.com/AQS2.png?imageView2/2/w/500/h/300)


**注：构造节点的过程必须保证线程安全，因为会有多个线程失败。那么它是如何做到的？AQS提供了一个基于CAS的构造尾节点的方法compareAndSetTail，它可以保证节点被正确地加入到队列中。**

#### 独占式获取锁
来看一看获取锁的流程。  
调用AQS的acquire（int args）方法获取同步状态。

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
**&&：** 短路与，当第一个为false时不再判断后面条件；第一个为true时还会判断第二个条件；  
**&:**     当第一个为false时，还会判断后面的条件；
- 
  当成功获取锁，即tryAcquire(arg)为true时，!tryAcquire(arg) 为false，跳出if，此时执行selfInterrupt()；
-   当没有成功获取锁，即tryAcquire(arg)为false时，!tryAcquire(arg) 为true时，接着判断第二个条件，两个步骤： 

步骤一：  addWaiter(Node.EXCLUSIVE)：将该节点加入同步队列的尾部,返回该节点；  
步骤二： acquireQueued(Node node, arg))：使该节点以"死循环"的方式获取同步状态；若获取不到则阻塞节点中的线程，被阻塞的线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。

**分析步骤一：addWaiter方法**

```
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        if (pred != null) { //如果有尾节点，快速尝试在尾部添加，减少开销
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;                  //如果已经有尾节点，利用CAS将自己添加为尾节点之后返回
            }
        }
        enq(node);          //如果没有尾节点，那么进入enq方法
        return node;
    }
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) {        
                if (compareAndSetHead(new Node()))    //初始化头节点
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {   //CAS添加node为尾节点
                    t.next = node;
                    return t;
                }
            }
        }
```

可以看到，若队列还未生成即没有尾节点，则进入enq方法中，先创造一个头节点，然后通过死循环**for(;;)** 来保证节点的正确添加，再通过**compareAndSetTail（CAS）** 这个方法确保节点能够被线程安全地添加（可以想象多个线程获取同步失败后，如果不保证线程安全添加，将导致顺序混乱，可能丢失线程），只有从CAS返回后，线程才能返回，否则将不断尝试。
这个enq方法将并发的添加节点的请求通过CAS变得串行化了。

**分析步骤二：acquireQueued方法**  
节点进入同步队列后，就进入了一个自旋的过程，每个节点（或线程）都在自省的观察，当获取到同步状态就可以从自旋中退出，否则依旧自旋。

```
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();     //获得前驱节点
                if (p == head && tryAcquire(arg)) {  //只有前驱节点是头节点才尝试获取锁，false直接跳出if
                    setHead(node);        //获取成功，将自己设置为头节点
                    p.next = null;           // help GC
                    failed = false;
                    return interrupted;       //返回false，回到acquire方法中，不执行selfInterrupt();
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

```
private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }
```
这段代码主要做了2件事：

  1. 判断当前节点的前驱节点是否为头节点并尝试tryAcquire，只有当前驱节点是head的节点才会尝试tryAcquire，如果节点尝试tryAcquire成功，执行setHead方法将当前节点作为head、将当前节点中的thread设置为null、将当前节点的prev设置为null，这保证了链表中头结点永远是一个不带Thread的空节点；

  2. 如果当前节点的前驱节点不是头节点或者tryAcquire失败，那么执行第13行~第15行的代码，做了两步操作，首先判断在acquie失败后是否应该park（阻塞），其次park并检查中断状态；  
  
分析下第2件事：
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)     //ws=SIGNAL= -1
            return true;
        if (ws > 0) {             //ws=CANCELLED= 1
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {                 //ws=CONDITION= -2 or PROPAGATE= -3
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
­这个方法做了以下工作，
­每个节点判断它前驱节点的状态：  
1. 它的前驱节点是SIGNAL状态的，返回true，表示当前节点应当park(阻塞)，执行parkAndCheckInterrupt()，­该方法利用LockSupport的park方法让当前线程阻塞，如下。 

```
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

2. 它的前驱节点的waitStatus>0，即CANCELLED，那么CANCELLED的节点作废，当前节点不断向前找并重新连接为双向队列，直到找到一个前驱节点的waitStats不是CANCELLED的为止。
3. 它的前驱节点不是SIGNAL状态且waitStatus<=0，即CONDITION或PROPAGATE，此时执行第11行代码，利用CAS机制，将前驱节点的状态更新为SIGNAL状态。
****



#### 独占式释放锁
调用AQS的release方法可以释放同步状态，唤醒后继节点。

```
public final boolean release(int arg) {
        if (tryRelease(arg)) {       
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);  //调用LockSupport来唤醒处于等待状态的线程
            return true;
        }
        return false;
    }
```
tryRelease释放成功，获取到head节点，如果head节点的waitStatus不为0的话，执行unparkSuccessor方法。

```
private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
这个方法做了以下工作：  
1. 头节点的waitStatus<0，将头节点的waitStatus设置为0；
2. 拿到头节点的下一个节点s，如果s==null或者s的waitStatus>0（被取消了），那么从队列尾巴开始向前寻找一个waitStatus<=0的节点作为后继要唤醒的节点；
3. 如果拿到了一个不等于null的节点s，就利用LockSupport的unpark方法让它取消阻塞。
#### 总结： 
获取同步状态时，AQS维护一个同步队列，获取状态失败的线程都会加入到队列中并在队列中进行自旋；移出队列的条件是前驱节点为头节点且获取同步状态成功。  
释放同步状态时，头节点唤醒它的后继节点。