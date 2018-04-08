---
title: Java并发工具类之CountDownLatch
date: 2017-12-3 18:42:18
tags: [并发,CountDownLatch]
categories: 技术
---
### 一、概念

CountDownLatch也叫闭锁，是并发包的工具类之一，允许一个或多个线程等待其他线程完成操作后再执行。

回忆之前，我们知道Join方法，用于让当前线程等待join的线程执行结束。

```
while(isAlive()){
    wait(0);
}
```


其原理就是不停地检查join线程是否存活，如果存活则一直等待。
CountDownLatch也可以实现join的功能，且功能更多。  

![image](http://osuskkx7k.bkt.clouddn.com/CountDownLatch.png)

CountDownLatch是通过一个计数器来实现的，当new 一个CountDownLatch对象的时候需要传入该计数器值。  
CountDownLatch有两个最主要的方法await()和countDown()。

当一个线程调用await()时会阻塞；每当一个其他线程完成自己的任务调用countDown()后，计数器的值就会减1。当计数器的值为0时，就表示所有的线程均已经完成了任务，然后阻塞的线程就可以继续执行了。

### 二、实现分析

**部分源码**：
```
public class CountDownLatch {
    private final Sync sync;
    //内部类Sync（继承AQS）
    private static final class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) {
            setState(count);
        }
        //重写
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
    //构造方法
     public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    ....
}
```
可以看到，CountDownLatch的实现同样依赖AQS，可见AQS作用之大。
观察Sync重写的方法（tryAcquireShared、tryReleaseShared），我们可以断定：
**CountDownLatch使用的是共享锁。**

#### await()的实现

调用await()方法会阻塞当前线程，直到计数器为0或被中断。

```
public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```
可以看到，其实调用了AQS的acquireSharedInterruptibly()方法：
```
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```
该方法中，如果线程被中断则抛出异常；否则尝试获取锁。

```
protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

```
尝试获取锁时调用了Sync重写的tryAcquireShared()方法：
很简单只有一行代码，但却是CountDownLatch的原理：**如果同步状态为0（计数器值为0）那么返回1，表示获取锁成功；否则返回-1，获取锁失败**，并进入doAcquireSharedInterruptibly()方法：

```
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    //注意r只会等于 0 or -1；
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
该方法在前面共享锁的文章中有提到，它是一个自旋尝试获取锁的方法，这里不再赘述。**注意18行的代码：int r = tryAcquireShared(arg)，在获取同步状态时只会返回两个值0和-1。**


#### countDown()的实现
每次调用countDown()时，计数器的数量就会减1。

```
public void countDown() {
        sync.releaseShared(1);
    }
```
调用的是AQS的releaseShared()方法，释放同步状态：
```
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
进入releaseShared()方法，调用Sync重写的tryReleaseShared()方法：
```
protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```
可以看到，**该方法自旋CAS操作释放同步状态（可能多个线程同时调用countDown()方法，所以用CAS保证原子性），计数器每次-1，但是直到同步状态为0（计数器为0）时，才返回true**。然后进入doReleaseShared()方法，唤醒阻塞的线程，使其从await()方法退出。

#### 总结

**CountDownLatch的内部实现是共享锁。**


**创建CountDownLatch时，需要传入计数器的初始值，可以理解为共享锁的总次数。**  
**当一个线程调用await()方法，会检查计数器的值，不为0则阻塞直到为0。  
当其他线程调用countDown()方法时（可以一个线程调用多次），会释放共享状态，计数器-1。**  
**当计数器减为0时，阻塞的线程才会运行。**


### 三、应用场景
CountDownLatch的应用场景：**主线程等到N个子线程执行完毕之后，再继续往下执行。** 如跑步比赛统计排名、启动程序等。

```
public class CountDownLatchDemo {
	
	private CountDownLatch cd = new CountDownLatch(5);
	/*
	 * 飞船类
	 */
	 class Plane extends Thread{
		 @Override
		public void run() {
			System.out.println("飞船准备就绪，倒计时5s：");
			System.out.println(cd.getCount());//计数器的值
			try {
				cd.await();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println("飞船起飞啦！！！！");
		}
	 }
	 /*
	 * 倒计时类
	 */
	 class Time extends Thread{
		 @Override
		public void run() {
			 for(int i=5;i>0;i--){
				 System.out.println("起飞倒计时："+i+"s");
				 cd.countDown();
			 }
		}
	 }
	
	public static void main(String[] args) {
		CountDownLatchDemo cdemo = new CountDownLatchDemo();
		Plane p = cdemo.new Plane();
		p.start();
		Time t = cdemo.new Time();
		t.start();
	}
}
```
输出结果：
```
飞船准备就绪，倒计时5s：
5
起飞倒计时：5s
起飞倒计时：4s
起飞倒计时：3s
起飞倒计时：2s
起飞倒计时：1s
飞船起飞啦！！！！
```
**当调用CountDownLatch的countDown方法时，计数器N就会-1，无论是在多个线程调用，还是一个线程调用多次（上面的例子就是在一个线程中多次调用）。**
