---
title: 深入理解synchronized关键字
date: 2017-10-14 17:30:15
tags: [并发,synchronized]
categories: 技术
---
## 一、synchronized的基本介绍

谈到synchronized关键字，想必大家都不陌生，对它的初次印象如果用两个字来概括，无非就是 **“同步”** 。小小的一个词，蕴含了大道理，那么就让我们来探索一下。  
### 1.简介
**synchronized实现同步的基础是：java中的任何一个对象都可以作为锁。**  

它有三种用法：  

 **1. 修饰普通同步方法，锁是当前实例对象**  
 
**2. 修饰静态同步方法，锁是当前类的class对象（唯一）**

**3. 修饰同步代码块，锁是括号中的对象**  

### 2.使用
来看以下几段代码  
（1）不使用synchronized
```
public class SynTest {
	
	public void method1(){
		System.out.println("method 1 start");
		try {
			System.out.println("method 1 execute");
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("method 1 end");
	}
	
	public void method2(){
		System.out.println("method 2 start");
		try {
			System.out.println("method 2 execute");
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("method 2 end");
	}
	
	public static void main(String[] args) {
		SynTest test = new SynTest();
		//线程1---method1
		new Thread(new Runnable() {
			@Override
			public void run() {
				test.method1();
			}
		}).start();
		//线程2---method2
		new Thread(new Runnable() {
			@Override
			public void run() {
				test.method2();
			}
		}).start();
	}
}
```
```
method 1 start
method 2 start
method 2 execute
method 1 execute
method 2 end
method 1 end
```
可以看出，在不加synchronized修饰时，两个线程同时执行，互不冲突，线程2比线程1执行的快，因此先执行完毕。  
（2）synchronized修饰普通方法

```
public class SynTest {
	
	public synchronized void  method1(){
		System.out.println("method 1 start");
		
		try {
			System.out.println("method 1 execute");
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("method 1 end");
	}
	public synchronized void method2(){
		System.out.println("method 2 start");
		
		try {
			System.out.println("method 2 execute");
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("method 2 end");
	}
	
	public static void main(String[] args) {
		SynTest test = new SynTest();
		//线程1---method1
		new Thread(new Runnable() {
			@Override
			public void run() {
				test.method1();
			}
		}).start();
		//线程2---method2
		new Thread(new Runnable() {
			@Override
			public void run() {
				test.method2();
			}
		}).start();
	}
}
```

```
method 1 start
method 1 execute
method 1 end
method 2 start
method 2 execute
method 2 end
```
可以看出，线程2在线程1执行完成后才开始执行，达到了同步的效果。**这是因为两个线程需要获取同一把锁（即test对象）**，线程1先拿到锁，线程2只能等待直到线程1释放锁，才能执行。  
（3）synchronized修饰静态方法

```
public class SynTest {
	
	public static synchronized void  method1(){
		System.out.println("method 1 start");
		
		try {
			System.out.println("method 1 execute");
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("method 1 end");
	}
	public static synchronized void method2(){
		System.out.println("method 2 start");
		
		try {
			System.out.println("method 2 execute");
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("method 2 end");
	}
	
	public static void main(String[] args) {
		SynTest test = new SynTest();
		SynTest test2 = new SynTest();
		//线程1---method1
		new Thread(new Runnable() {
			@Override
			public void run() {
				test.method1();
			}
		}).start();
		//线程2---method2
		new Thread(new Runnable() {
			@Override
			public void run() {
				test2.method2();
			}
		}).start();
	}
}
```
```
method 1 start
method 1 execute
method 1 end
method 2 start
method 2 execute
method 2 end

```
可以看出，两个线程同样获得了同步的效果。但是明明是两个不同的对象（test、test2）所调用的，这是为什么？  
**在这里synchronized修饰的是静态方法，而静态方法本质上是类的方法，因此这里的同步本质上是对类（Class对象）的同步**，test、test2都是属于类的实例对象，所以也会同步执行，不能并发执行。  
**注：每个类只有一个Class对象。**  
（4）synchronized修饰同步块

```
public class SynTest {
	
	public  void  method1(){
		synchronized(this){
		System.out.println("method 1 start");
		try {
			System.out.println("method 1 execute");
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("method 1 end");}
	}
	public void method2(){
		synchronized(this){
		System.out.println("method 2 start");
		
		try {
			System.out.println("method 2 execute");
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("method 2 end");}
	}
	
	public static void main(String[] args) {
		SynTest test = new SynTest();
		//线程1---method1
		new Thread(new Runnable() {
			@Override
			public void run() {
				test.method1();
			}
		}).start();
		//线程2---method2
		new Thread(new Runnable() {
			@Override
			public void run() {
				test.method2();
			}
		}).start();
	}
}
```

```
method 1 start
method 1 execute
method 1 end
method 2 start
method 2 execute
method 2 end

```
可以看到，两个线程获得了同步的效果。**两个线程的锁都是synchronized同步块括号中的this对象，即当前对象（test）**。

## 二、synchronized的实现原理
### 2.1 实现原理  
**JVM基于进入和退出Monitor对象来实现方法同步和代码块同步。** 但两者的实现细节不一样。
#### 1. 代码块同步  
**代码块同步是使用monitorenter和monitorexit指令（字节码指令）来完成。**   
monitorenter指令是在编译后插入到同步代码块的开始位置，monitorexit指令插入到方法结束处和异常处，JVM保证每一个monitorenter都有一个monitorexit与之相对应。任何对象都有一个monitor与之相关联，当且一个monitor被持有之后，它将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor所有权，即尝试获取对象的锁； 
#### 2. 方法同步  
**方法同步是根据方法上的ACC_SYNCHRONIZED标识符（不是字节码指令）来实现的。** 它没有通过指令monitorenter和monitorexit来完成（也可以通过它完成）。  
反编译可以发现，相比普通方法，常量池中多了ACC_SYNCHRONIZED标示符。JVM就是根据该标示符来实现方法的同步的：当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。

**总结：** 二者其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码指令来完成。
### 2.2 对象头
synchronized用的锁是存在java对象头里的。
要了解对象头，先看看对象在内存中的分布。分为三部分：对象头，实例数据，和对齐填充。（如下图）  
![image](http://osuskkx7k.bkt.clouddn.com/%E5%AF%B9%E8%B1%A1%E5%A4%B421.PNG?imageView2/2/w/500/h/600)

从图上可以看到，对象头由2个字存储（若是数组对象则为3个字，多一个存储数组长度）。  
对象头主要包括以下两部分数据（还有一个Fields）：  
![image](http://osuskkx7k.bkt.clouddn.com/%E5%AF%B9%E8%B1%A1%E5%A4%B422.PNG?imageView2/2/w/400/h/500)  
**Mark Word（标记字段）：** 用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等。  
**Klass Pointer（类型指针）：** 指向对象的类元数据，虚拟机通过这个指针来确定这个对象是哪个类的实例。  
当对象头处于无锁状态时，它的Mark Word存储结构如下：

锁状态| 25bit| 4bit| 1bit是否是偏向锁| 2bit锁标志位
--------|-------|---|---|-------
&#160;无锁状态|对象的hashCode|对象分代年龄|&#160;&#160;&#160;&#160;&#160;&#160;&#160;0|&#160;&#160;&#160;01

**注：在运行期间，Mark Word里的存储数据会随着锁标志位的变化而变化。**


## 三、锁的优化和对比
jdk1.6之后对synchronized的实现进行了优化，来减少锁操作的开销。  
因此锁出现了以下四种状态 **：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态。**  
它们并不是一成不变的状态，而是会通过互相竞争而升级，但是为了提高获得锁和释放锁的效率，它们只能升级不能降级。  升级顺序如下：  
无锁 --> 偏向锁 --> 轻量级 --> 重量级
### 3.1 偏向锁
背景：大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让获取锁的代价降低而引入了偏向锁。**偏向锁主要为了解决在没有竞争情况下锁的性能问题。**
#### 1. 偏向锁的加锁  
主要步骤如下：  
（1）检测对象头Mark Word中的状态是否为偏向锁状态：  
若是偏向锁状态（偏向锁标志为1，锁标志位01）执行步骤（2）；  
若是无锁状态（偏向锁标志关闭，锁标志位01），那么线程在执行同步块之前，JVM会现在当前线程的栈帧中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，并存储锁偏向的线程ID（当前线程）；  
（2）是偏向锁状态，则测试线程ID是否为当前线程ID，如果是则执行步骤（5），否则执行步骤（3）；  
（3）线程ID不为当前线程ID，则通过CAS操作竞争锁，竞争成功，则将Mark Word的线程ID替换为当前线程ID，否则执行步骤（4）；   
（4）通过CAS操作竞争锁失败，证明当前存在多线程竞争的情况，当到达全局安全点，获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码块；  
（5）执行同步代码块


#### 2. 偏向锁的解锁  
偏向锁的释放采用了一种只有竞争才会释放锁的机制，线程是不会主动去释放偏向锁，需要等待其他线程来竞争。偏向锁的撤销需要等待全局安全点，此时没有正在执行的字节码。步骤如下：  
（1）首先暂停持有偏向锁的线程，然后检查该线程是否活着：
没有活着：将对象头设置成无锁状态；  
（2）活着：要么重新偏向于其他线程；要么恢复到无锁或升级为轻量锁；  
（3）最后唤醒暂停的线程

#### 3. 偏向锁的关闭    
偏向锁默认开启，JVM参数关闭延迟：-XX：BiasedLockingStartupDelay=0。  
如果确定应用程序里所有的锁通常都处于竞争状态下，通过JVM参数关闭偏向锁：-XX:-UseBiasedLocking=false，那么程序会默认进入轻量级锁状态。


### 3.2 轻量级锁
背景：**“对于绝大部分的锁，在整个同步周期内都是不存在竞争的”**，这是一个经验数据，也是轻量级锁能提升程序同步性能的依据。**轻量级锁所适应的场景是线程交替执行同步块的情况。**


#### 1. 轻量级锁的加锁  
当关闭偏向锁功能或偏向锁升级为轻量级锁时，会尝试去获取轻量级锁。主要步骤如下：  
（1）检测对象头Mark Word中的状态是否为无锁状态：若是无锁状态（偏向锁标志关闭，锁标志位01）执行步骤（2）；否则步骤（4）  
（2）若是无锁状态，将对象头中的Mark Word复制到锁记录中（在执行同步块之前，JVM会先在当前线程的栈帧中创建用于存储锁记录的空间）；  
（3）通过CAS将Mark Word替换为指向锁记录的指针：如果成功表示竞争到锁，执行同步代码；如果失败执行步骤（4）；  
（4）判断当前对象的Mark Word是否指向当前线程的栈帧，如果是则表示当前线程已经持有当前对象的锁，则直接执行同步代码块；否则只能说明该锁对象已经被其他线程抢占了，这时轻量级锁需要膨胀为重量级锁，锁标志位变成10，后面等待的线程将会进入阻塞状态；

#### 2. 轻量级锁的解锁  
轻量级锁的释放也是通过CAS操作来进行的，主要步骤如下： 

（1）取出获取轻量级锁时保存在Displaced Mark Word中的数据；  
（2）用CAS操作将取出的数据替换当前对象的Mark
Word中，如果成功，则说明释放锁成功，否则执行（3）；  
（3） 如果CAS操作替换失败，说明有其他线程尝试获取该锁，存在锁竞争，锁会膨胀成重量级锁。
### 3.3 重量级锁
**重量级锁通过对象内部的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的Mutex Lock实现**，由于使用Mutex Lock需要将当前线程挂起并从用户态切换到内核态来执行，这种切换的代价非常昂贵。


### 3.4 锁的对比

锁 | 优点 | 缺点 | 适用场景
|---|------|----|--
偏向锁 | 加锁和解锁不需要额外的消耗,和执行非同步方法相比仅存在纳米级的差距|如果线程间存在锁竞争，有额外的锁撤销的消耗 | 适用于只有一个线程访问同步块的场景
轻量级锁 |竞争的线程不会阻塞，提高了程序的响应速度|如果始终得不到锁竞争的线程，使用自旋会消耗CPU | 追求响应时间，同步块执行速度非常快
重量级锁 | 线程竞争不使用自旋，不会消耗CPU|线程阻塞，响应时间缓慢 | 追求吞吐量，同步块执行速度较长


