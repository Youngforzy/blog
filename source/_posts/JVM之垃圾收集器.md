---
title: JVM之垃圾收集器
date: 2017-10-23 19:28:13
tags: [JVM,垃圾收集器]
categories: 技术
---
### 一、垃圾收集器概述

**垃圾收集器是内存垃圾回收算法的具体实现。**  
Java虚拟机规范中对垃圾收集器应该如何实现并没有任何规定，因此出现了7种收集器：Serial、ParNew、Parallel Scavenge、Serial Old、Parallel Old、CMS、G1。  
它们以组合的形式配合工作来完成不同分代的垃圾收集工作。

**常用组合**：
1. Serial/Serial Old
2. ParNew/Serial Old
3. ParNew/CMS
4. Parallel Scavenge/Parallel Old
5. G1

![image](http://osuskkx7k.bkt.clouddn.com/JVMtool2.png)

#### Minor GC和Full GC
Minor GC（新生代GC）：发生在新生代的GC动作，频率高，回收速度块。

Full GC（老年代GC）：发生在老年代的GC动作，老年代满了后才进行，一般伴随至少一次Minor GC，频率低，回收速度慢。

#### 垃圾收集器种类
目前的收集器主要有以下三种：  

**串行收集器：** 只有一条垃圾收集线程工作  

**并行收集器：** 多条垃圾收集线程并行工作，用户线程等待


**并发收集器：** 垃圾收集线程与用户线程同时执行（不一定并行，可能交替执行）

### 二、垃圾收集器详述

#### 新生代收集器（3种）
##### Serial 收集器（串行）
Serial收集器是一个单线程的收集器。  
“单线程”：不仅只是有一条收集线程，而且必须暂停用户工作线程。 

工作过程：  

![image](http://osuskkx7k.bkt.clouddn.com/seria.png)

**特点：** 单线程，无线程切换，简单高效，（管理内存小，停顿可以接受）  
**缺点：**  暂停工作线程    
**应用场景：** 适用于运行在Client模式下的虚拟机


##### ParNew 收集器（并行）
ParNew收集器是Serial收集器的多线程版本。

工作过程：  

![image](http://osuskkx7k.bkt.clouddn.com/ParNew.png)

**特点：** 多线程，除Serial外唯一能和CMS收集器配合  
**缺点：**  暂停工作线程，单线程下不如Serial    
**应用场景：** 适用于运行在Server模式下的虚拟机

##### Parallel Scavenge 收集器（并行）

Parallel Scavenge收集器被称为“吞吐量优先”收集器。

工作过程：  

![image](http://osuskkx7k.bkt.clouddn.com/parallel%20scavenge.png)

**特点：** 
- **可控制吞吐量。**  
吞吐量 = 用户代码时间/（用户代码时间+垃圾收集时间）  
吞吐量越高，表示越高效地利用CPU，适合后台运算任务。  
- **GC 自适应调节策略**  
  JVM可以根据当前系统的运行情况自适应调节参数，以提供最合适的停顿时间和最大的吞吐量。

**缺点：** 相比停顿时间更注重吞吐量    
**应用场景：** 主要用于后台计算，不需要与用户进行太多交互，对暂停时间没有特别高的要求等场景，如批量处理；

#### 老年代收集器（3种）

##### Serial Old 收集器（串行）
Serial Old是Serial的老年代版本，同样是一个单线程收集器。  

工作过程：  

![image](http://osuskkx7k.bkt.clouddn.com/seria.png)

**特点：** 单线程，“标记-整理”算法  
**缺点：**  暂停工作线程    
**应用场景：** 适用于运行在Client模式下的虚拟机

##### Parallel Old 收集器（并行）

Parallel Old是Parallel Scavenge收集器的老年代版本，多线程收集器。（JDK1.6之后出现）

工作过程：  

![image](http://osuskkx7k.bkt.clouddn.com/parallel%20old.png)

**特点：** 多线程，“标记-整理”算法，唯一能和Parallel Scavenge收集器配合  
**缺点：**  暂停工作线程   
**应用场景：** 适用于注重吞吐量及CPU资源敏感的场合

##### CMS 收集器（并发）

**CMS（Concurrent Mark Sweep）是一种以获取最短回收停顿时间为目标的收集器。  
第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。**  

工作过程分为**4个步骤**：  
1. **初始标记**：标记“GC-Roots”关联的对象（Stop the World）
2. **并发标记**： 进行GC-Roots Tracing的过程，在刚才产生的集合中标记出存活对象；
3. **重新标记**：修正并发标记期间因用户程序继续运作而导致标记变动的那一部分对象的标记记录（Stop the World）
4. **并发清除**：回收所有垃圾对象

所以，**在初始标记和重新标记阶段，还是只有垃圾收集线程工作；并发标记和并发清除阶段是和用户线程并发执行的**。

![image](http://osuskkx7k.bkt.clouddn.com/CMS.png)

**特点：** 多线程并发执行，停顿时间短  
**缺点：**  
- 对CPU资源十分敏感
- 无法处理浮动垃圾
- 产生大量空间碎片（由于是“标记-清除”算法）  

**应用场景：** 适用于大型网站或B/S的服务端，注重响应速度和用户体验。

#### 通用收集器
##### G1 收集器（并发）
G1是目前最前沿的收集器，可处理整个GC堆，JDK1.7之后出现。  

**G1是如何处理整个堆**？  
G1将整个堆划分为多个大小相等的独立区域Region（不再是新生代老年代），然后跟踪各个Region获得其垃圾收集价值大小，并在后台维护一个优先列表，根据允许的收集时间，优先回收价值最大的Region（名称Garbage-First的由来）。这个过程保证了在有限的时间内可以回收更多的垃圾。


工作过程分为**4个步骤**： 
1. **初始标记**：标记“GC-Roots”关联的对象，并修改TAMS的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象
2. **并发标记**：根据GC-Roots进行可达性分析，找出存活对象
3. **最终标记**：为了修正并发标记期间由于用户程序继续运作而导致标记产生变动的那一部分对象的标示记录
4. **筛选回收**：根据各个Region的价值回收


![image](http://osuskkx7k.bkt.clouddn.com/G1.png)

**特点：** 
- 并行且并发  
- 独立处理整个GC堆，不需要配合其他收集器
- 可预测停顿时间
- 不产生空间碎片


**应用场景：** 面向服务器，适用于多CPU及大容量内存的机器。


### 三、常见的参数配置

- **Xms**：堆的最小值（初始）
- **Xmx**：堆的最大值  
（Xms、Xmx二者一样时，可避免自动扩展）
- **Xmn**：堆中新生代的大小
- **Xss**：每个线程的堆栈大小
- **XX:PermSize**：永久代的大小（初始）
- **XX:MaxPermSize**：永久代的最大值
- **XX:NewRatio**：年轻代（包括Eden和两个Survivor区）与年老代的比值，设置为3，则年轻代与年老代所占比值为1：3，年轻代占整个堆栈的1/4

- **XX:SurvivorRatio**：年轻代中Eden区与Survivor区的大小比值。设置为4，则两个Survivor区与一个Eden区的比值为2:4，一个Survivor区占整个年轻代的1/6





