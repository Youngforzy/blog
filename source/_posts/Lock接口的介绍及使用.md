---
title: Lock接口的介绍及使用
date: 2017-10-17 21:40:19
tags: [并发,Lock]
categories: 技术
---
### 一、Lock的介绍

我们知道，在Java中锁的实现可以由synchronized关键字来完成，但在Java5之后，出现了一种新的方式来实现——Lock接口。

那么为什么提出这种新的方式呢？  
在多线程的情况下，当一段代码被synchronized修饰之后，同一时刻只能被一个线程访问，其他线程都必须等到该线程释放锁之后才能有机会获取锁访问这段代码，占用锁的线程只有在两种情况下才能释放锁：  

1. 线程执行完了这段代码，释放锁；
2. 线程执行发生异常，释放锁；  

考虑一下，如果该线程由于IO操作或者其他原因（调用Sleep方法）被阻塞了，那么其他线程就会一直无期限地等待下去，后果可想而知。  
那么能否用一种方式来防止等待的线程无限等待呢？（等待一段时间或者响应中断）通过Lock就可以实现。

再如：当用多线程对文件进行读写时，读与写是互斥的，写与写是互斥的，但读与读却不是互斥的。如果用synchronized来实现同步，就会有这样的问题：多个线程都只需要读操作，但只能有一个线程进行读操作，其他线程只能等待。  
那么能不能让线程不用等待，多线程都能进行读操作呢？通过Lock就可以实现。




### 二、Lock的用法

Lock是一个接口，包含以下方法：
```
public interface Lock {
void lock();      //获取锁
void lockInterruptibly() throws InterruptedException; //可中断的获取锁
boolean tryLock(); //尝试非阻塞的获取锁
boolean tryLock(long time, TimeUnit unit) throws InterruptedException; //超时获取锁
void unlock(); //释放锁
Condition newCondition(); //获取等待通知组件，和当前的锁绑定
}
```
可以看到当使用Lock时，获取锁和释放锁都是主动调用执行的，而synchronized则是系统自动释放锁的。  
前四个方法都是用来获取锁的，但各有区别：  
- **lock()**：是最常用的获取锁的方法，若锁被其他线程获取，则等待（阻塞）。
- **tryLock()**：尝试非阻塞地获取锁，立即返回。获取成功返回true；获取失败返回false，但不会阻塞。
- **tryLock(long time, TimeUnit unit)**：与tryLock()相似，但是会超时等待一段时间，如果未获取到返回false。
- **lockInterruptibly()**：可中断地获取锁，该方法会响应中断，在锁的获取过程中可以中断当前线程。  

**注：当使用synchronized关键字时，一个线程在等待获取锁的过程中是无法中断的。而使用lockInterruptibly()方法获取某个锁时，如果不能获取到，在进行等待的情况下是可以响应中断的。**




Lock的使用：

```
Lock lock = new ReentrantLock(); //可重入锁（Lock的一种实现）
lock.lock();
try{
    dosomething();
}finally{
    lock.unlock();
}
```
**在finally块中释放锁的目的是保证获取锁之后，最终能被释放。**  
不要将获取锁——lock()放在try中，如果在获取锁时发生了异常，异常抛出的同时，也会导致锁的无故释放（需要主动释放）。





### 三、与synchronized的区别
Lock与synchronized的区别：  

1. Lock是一个接口，是代码层面的实现；synchronized是关键字，是内置的语言实现（JVM层面）。
2. Lock是显示地获取释放锁，扩展性更强；synchronized是隐式地获取释放锁，更简捷。
2. Lock在发生异常时，如果没有主动通过unlock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁；synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生。
3. Lock可以让等待锁的线程响应中断；而使用synchronized时等待锁的线程会一直等待下去，不能响应中断；
4. Lock可以尝试非阻塞、可中断、超时地获取锁；synchronized不可以。
4. Lock可以知道是否成功获取锁；synchronized无法知道。  



总结：在资源竞争不是很激烈的情况下，Synchronized的性能要优于Lock；但是在资源竞争很激烈的情况下，Synchronized的性能会下降几十倍，Lock的性能更优；