---
title: JVM之对象创建过程
date: 2017-10-26 18:48:19
tags: [JVM,对象创建过程]
categories: 技术
---

### 一、Java对象的创建过程

在Java程序中，通常都是通过 new 关键字来创建对象，那么在虚拟机中对象是如何创建的？  
（普通Java对象，不包含数组和Class对象）

虚拟机创建对象主要经历5个步骤：**类加载检查、为对象分配内存、内存空间初始化、对象设置、执行对象<init>方法。**
#### 1 类加载检查
当虚拟机遇到 new 指令时，首先会去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且**检查这个符号引用代表的类是否已被加载、解析和初始化过**。如果没有，先执行类加载过程。
#### 2 为对象分配内存
对象所需的内存大小在类加载完成时已经确定，因此**为对象分配内存等同于在Java堆中划分一块大小确定的内存空间**。  
##### 两种分配方式：
- **指针碰撞**：Java堆中的内存是完整的，将指针往空闲空间挪动一段与对象大小相等的距离
- **空闲列表**：Java堆中的内存不是完整的，JVM维护了一个记录可用内存的列表，分配时将列表中足够大的空间划分给对象，并更新列表

因此选择何种分配方式由Java堆是否完整决定，而这又由垃圾收集器是否带有压缩整理功能决定。如：

- **Serial、ParNew等采用指针碰撞**
- **CMS基于“标记-清除”采用空闲列表**

##### 线程安全问题
当处于并发情况下时，还要考虑线程安全问题。
两种解决方案：
- **对分配内存的动作进行同步处理**。JVM采用CAS加失败重试的方式保证原子性
- **把内存分配的动作按照线程划分在不同的空间进行**。每个线程在Java堆中预先分配一小块内存（本地线程分配缓冲TLAB），只有TLAB用完重新分配时才同步锁定。
#### 3 内存空间初始化
JVM将分配到的内存空间都初始化为零值（不包括对象头）；若使用TLAB，则提前至TLAB分配时执行。

这一步**保证了对象实例字段在Java代码中可以不赋初值就直接使用**，程序能访问到这些字段的数据类型所对应的零值。

#### 4 对象设置
JVM设置对象头信息，如类元数据信息、对象的哈希码、对象的GC分代年龄信息等。还有是否启用偏向锁。
#### 5 执行对象<init>方法
此时，对于JVM来说，对象已经产生；  
对于Java程序来说，才刚刚开始，执行<init>方法进行初始化，一个对象才算真正创建完成。


### 二、Java对象的初始化

在Java对象初始化过程中，主要涉及三种执行对象初始化的结构，分别是 **实例变量初始化、实例代码块初始化** 以及 **构造函数初始化**。 

#### 实例变量初始化与实例代码块初始化
在定义（声明）实例变量的同时，还可以直接对实例变量进行赋值或者使用实例代码块对其进行赋值。  
如果我们**以这两种方式为实例变量进行初始化，那么它们将在构造函数执行之前完成这些初始化操作。** 实际上，如果我们对实例变量直接赋值或者使用实例代码块赋值，那么编译器会将其中的代码放到类的构造函数中去，并且这些代码会被放在对超类构造函数的调用语句之后(Super())，构造函数本身的代码之前。
```
public class Tdemo2 {
	//成员变量
	private int i = 1;
	private int j = 1;
	public  Tdemo2(int c){
		System.out.println(i);
		System.out.println(j);
		this.i = c;
		System.out.println(i);
	}
	//代码块
	{
		j = j+1;
	}
	//静态代码块
	static{
		int a = 5;
		System.out.println(a);
	}
	public static void main(String[] args) {
		new Tdemo2(3);
	}
}
```
输出：
```
5
1
2
3
```
可见执行顺序是static代码块、成员变量赋值、代码块、构造函数。
#### 构造函数初始化

**Java要求在实例化类之前，必须先实例化其超类，以保证所创建实例的完整性。**
Java强制要求Object对象(Object是Java的顶层对象，没有超类)之外的所有对象构造函数的第一条语句必须是超类构造函数的调用语句或者是类中定义的其他的构造函数，如果我们既没有调用其他的构造函数，也没有显式调用超类的构造函数，那么编译器会为我们自动生成一个对超类构造函数的调用。

实际上，实例化一个类的对象的过程是一个典型的递归过程。

![image](http://osuskkx7k.bkt.clouddn.com/%E5%AE%9E%E4%BE%8B%E5%8C%96%E9%80%92%E5%BD%92%E8%BF%87%E7%A8%8B.png)


在准备实例化一个类的对象前，首先准备实例化该类的父类，如果该类的父类还有父类，那么准备实例化该类的父类的父类，依次递归直到递归到Object类。

**注意：实例初始化不一定要在类初始化结束之后才开始初始化。**


回忆一下Java中赋值顺序： 
1. 父类的静态变量赋值 
2. 自身的静态变量赋值 
3. 父类成员变量赋值和父类代码块赋值 
4. 父类构造函数赋值 
5. 自身成员变量赋值和自身块代码赋值 
6. 自身构造函数赋值


### 三、Java对象的创建方式

Java对象的创建方式有 5 种：
1. new 关键字
```
Person p = new Person();
```
2. Class类的newInstance()（反射）
```
Person p2 = Person.class.newInstance();
```
3. Constructor类的newInstance方法（反射）
```
Constructor c = Person.class.getConstructor();
Person p3 = (Person) c.newInstance();
```
4. clone方法（实现Cloneable接口）
```
Person p4 = (Person) p3.clone();
```
5. 反序列化（实现Serializable接口）
```
//写对象
ObjectOutputStream output = new ObjectOutputStream(new FileOutputStream("person.txt"));
output.writeObject(p);
output.close();

//读对象
ObjectInputStream input = new ObjectInputStream(new FileInputStream("person.txt"));
Person p5 = (Person) input.readObject();
```



