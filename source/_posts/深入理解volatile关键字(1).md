---
title: 深入理解volatile关键字
date: 2017-10-12 15:12:12
tags: [并发,volatile]
categories: 技术
---
## 一、volatile的特性
###   1. volatile的可见性
 可见性的意思是：当一个线程修改一个共享变量时，另一个线程能读到这个修改的值。  
 volatile在多线程的开发中，保证了共享变量的可见性（立即）。
```
//全局变量
boolean open=true;

//线程A
resource.close();
open = false;

//线程B
while(open) {
doSomething(resource);
}
```
open是一个全局变量，用来描述一个资源的打开关闭状态，当线程A把资源关闭后，open置为false，而这个改动对线程B**不是立即可见**的，因此线程B还会运行，从而造成错误。  
当添加volatile关键字修饰之后，线程B就可以**立即**将改动后的open变量（主内存）同步到自己的工作内存中，从而正确的停止运行。


###  2. volatile的有序性（禁止指令重排序）
 

```
//线程A
context = loadContext();//初始化
init = true;
//线程B
while(!init){ //根据init变量决定是否使用context（为false时等待，为true时顺序执行）
   sleep(100);
}
doSomething(context);

```
以上程序运行没有问题，然而当线程A中发生了指令重排序：
```
init = true;
context = loadContext();
```
那么B就可能跳过等待，拿到一个正在初始化或初始化未完成的context对象，从而发生程序错误。  
当init变量用volatile修饰后，就会阻止JVM对其相关代码进行指令重排，这样就能够按照既定的顺序执行。  
 在双重判断类型的单例模式中正是应用了volatile关键字的这个特性，才不会导致单例模式失效。
 
###  3. volatile不保证操作的原子性
原子操作：不可中断的一个或一系列操作（多线程中借助于原子操作可以实现互斥锁）  
原子性：一个操作或多个操作要么全部成功执行，要么就都失败。一个操作是原子操作，那么我们称它具有原子性。  
先看示例代码
```
public class TestAtomic {
//计数器
private static volatile int count=0;
	
	public void inc(){
		count++;
	}
	private static class Countone implements  Runnable {
		public void run() {
			for(int i=0;i<100;i++){
				count++;
			}
		}
	}
	public static void main(String[] args) {
		
		for(int i=0;i<10;i++){
			new Thread(new Countone()).start();
		}
		//保证10个线程都执行完毕
		while(Thread.activeCount()>1)
			Thread.yield();
		
		System.out.println("最终count的值为："+TestAtomic.count);
	}
}
```
```
输出结果：
最终count的值为：863
最终count的值为：1000
最终count的值为：872
最终count的值为：835
```
将count变量声明为volatile int类型，保证了所有线程对变量count的可见性，上述定义了10个线程，每个线程对count执行100次自增的操作，按照理想的结果，最后的结果应该为10*100=1000，然而多次运行可以看到，并不是每次的结果都是1000，难道volatile修饰的变量的可见性特征失效了？**并不是，而是volatile只能保证共享变量对所有线程的可见性，不能保证变量操作的原子性。** count++不是一个原子操作，因此volatile不能保证这个操作的原子性。

**有以下三种方式可以保证对变量操作的原子性：**

 1. 使用synchronized关键字
 2. 使用Lock对象
 3. 使用java.util.concurrent.atomic包下提供的原子操作类

## 二、volatile的实现原理
### 1.可见性
它的实现原理与java内存模型（JMM）相关，每个线程都有自己的工作内存，并共享主内存的数据。下面是普通变量与volatile变量的异同：

 - 普通变量：读操作会优先读取工作内存的数据，如果工作内存中不存在，则从主内存中拷贝一份数据到工作内存中；写操作只会修改工作内存的副本数据。这种情况下，其它线程就无法读取变量的最新值。
 
 - volatile变量：读操作时会把工作内存中对应的值设为无效，要求线程从主内存中读取数据；写操作时会把工作内存中对应的数据刷新到主内存中。这种情况下，其它线程就可以读取变量的最新值。

那么它是如何是实现的呢？这就涉及到了CPU指令。  
如果对声明了volatile变量进行写操作，**JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。** 但是就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里。
### 2.禁止重排序
在执行程序时为了提高性能，编译器和处理器通常会对指令做重排序： 

 - 编译器重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序； 
 - 处理器重排序。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序； 
指令重排序对单线程没有影响，但是会影响多线程的正确性，因此需要对禁止指令重排序。

在上面提到过lock指令，**lock指令其实就相当于一个内存屏障。内存屏障是一组处理指令，用来实现对内存操作的顺序限制。** volatile的底层就是通过内存屏障来实现的。

## 三、volatile与synchronized的比较

 1. volatile本质是在告诉JVM当前变量在寄存器（工作内存）中的值是不确定的，需要从主存中读取； synchronized则是锁定当前变量，只有当前线程可以访问该变量，其他线程被阻塞。 
 
 2. volatile只能修饰变量，synchronized可以修饰变量、方法和类；  
 
 3. volatile保证变量的修改可见性，synchronized则可以保证变量的修改可见性和原子性。  
 4. volatile不会造成线程的阻塞，synchronized可能会造成线程的阻塞。  
 5. volatile比synchronized的使用和执行成本更低，因为它不会引起上下文的切换和调度。

## 四、volatile的扩展

### 1.volatile的常用场景

 - 状态标记量（高并发的场景）

 - 双重判断的单例模式

### 2.volatile修饰数组

**问题：volatile能否保证数组中元素的可见性？** 如果用volatile修饰一个数组，那么当一个线程对数组中的元素进行设值时，对另一个线程是否**立即**可见？ 

**答案：** 不能立即可见。因为volatile修饰的数组只针对数组的引用具有volatile的语义，而不是它的元素。
