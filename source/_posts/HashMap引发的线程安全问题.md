---
title: HashMap引发的线程安全问题
date: 2017-11-3 17:48:18
tags: [HashMap,线程安全]
categories: 技术
---
### 一、线程安全性
我们知道，HashMap是非线程安全的，只能在单线程的情况下使用。那么为什么不能在并发的情况下使用呢？**因为在并发时，HashMap的扩容会产生错误而形成环形链表，导致读取数据时发生死循环**。

回忆前面描述的扩容过程，调用了transfer()方法将旧链表转化为新链表：

```
void transfer(Entry[] newTable) {  
    Entry[] src = table;  
    int newCapacity = newTable.length;  
    for (int j = 0; j < src.length; j++) {  
        Entry<K,V> e = src[j];  
        if (e != null) {  
            src[j] = null;  
            do {  
                Entry<K,V> next = e.next;  
                int i = indexFor(e.hash, newCapacity);  
                e.next = newTable[i];  
                newTable[i] = e;  
                e = next;  
            } while (e != null);  
        }  
    }  
}
```
关键代码如下：

```
        do {  
        Entry<K,V> next = e.next;  
        int i = indexFor(e.hash, newCapacity);  
        e.next = newTable[i];  
        newTable[i] = e;  
        e = next;  
        } while (e != null);  
```
循环操作将旧链表中的节点放入新链表，直到下一个节点next为null。  
分别在单线程和多线程的环境下描述扩容过程。
#### 单线程扩容
假设hash数组的大小为2，负载因子为1，即超过1×2=2个元素时扩容，添加3个元素5、7、3，数组大小扩大为4，扩容过程如下：
    
![image](http://osuskkx7k.bkt.clouddn.com/h1.png?imageView2/2/w/900/h/450)

原链表中3个元素，循环3次，具体如下：
```
第一次循环
e = 3,next = 7
3.next = tab[i] = null  (此时数组tab[i]为空)
tab[i] = 3
e = 7 

第二次循环
e = 7,next = 5
7.next = tab[i] = 3
tab[i] = 7
e = 5

第三次循环
e = 5,next = null
5.next = tab[i2] = null  (此时数组tab[i2]为空)
tab[i2] = 5
e = null 

(停止循环)

```


#### 多线程扩容
为什么多线程环境下扩容会形成环形链表呢？  
还是刚刚的例子，两个线程并发执行，线程1在进入do循环的第一行挂起，线程2继续执行

```
    do {  
        Entry<K,V> next = e.next;  //线程1在此处挂起
        int i = indexFor(e.hash, newCapacity);  
        e.next = newTable[i];  
        newTable[i] = e;  
        e = next;  
        } while (e != null);
```
由前面单线程的情况可知，线程2此时成功扩容，结果如下：

![image](http://osuskkx7k.bkt.clouddn.com/h2.png)

线程1恢复执行，已知线程1的 e 指向了key(3)，而next指向了key(7)，扩容过程如下：

```
第一次循环
e = 3,next = 7
3.next = tab[i] = null  (此时数组tab[i]为空)
tab[i] = 3
e = 7 

第二次循环
e = 7,next = 3
7.next = tab[i] = 3
tab[i] = 7
e = 3

第三次循环
e = 3,next = null
3.next = tab[i] = 7
tab[i] = 3
e = null

(停止循环)
```
第一次循环图：  

![image](http://osuskkx7k.bkt.clouddn.com/h3.png)  

第二次循环图：

![image](http://osuskkx7k.bkt.clouddn.com/h22.png)

第三次循环图：

![image](http://osuskkx7k.bkt.clouddn.com/h4.png)

在停止循环后，问题就出现了，如图所示，key(3)和key(7)构成了环形链表。  
**于是，当我们调用HashMap的get方法时，由于查找链表节点时无法退出，就会产生无限循环。**
### 二、解决方法

解决方法就是采用同步的数据结构，主要有以下三种：
1. **Hashtable**
2. **Collections.synchronizedMap()**
3. **ConcurrentHashMap**

#### Hashtable
Hashtable是线程安全的。

```
public synchronized V put(K key, V value) {...}

public synchronized V get(Object key) {...}
```
可以看到，**Hashtable是通过在方法上加上synchronized关键字来实现同步功能的**。当一个线程访问时，其他线程都被阻塞住，这种方式效率很低，目前几乎不被使用。

#### Collections.synchronizedMap()
调用Collections的synchronizedMap()方法，传入一个Map，可以得到一个线程安全的SynchronizedMap。

```
private static class SynchronizedMap<K,V>
        implements Map<K,V>, Serializable {
        private final Map<K,V> m;     
        final Object      mutex;        // Object on which to synchronize

        SynchronizedMap(Map<K,V> m) {
            this.m = Objects.requireNonNull(m);
            mutex = this;
        }
        ....
        public V get(Object key) {
            synchronized (mutex) {return m.get(key);}
        }

        public V put(K key, V value) {
            synchronized (mutex) {return m.put(key, value);}
        }
```
可以看到，**它同步的原理同样也是使用了Synchronized关键字，不同的是Synchronized修饰代码块，并且将自身（this）作为了锁对象（mutex）。**


#### ConcurrentHashMap
ConcurrentHashMap是JDK1.5之后引入的，是为了替代上面提到的二者。
ConcurrentHashMap是线程安全且高效的HashMap，它使用了多个锁来控制对hash数组不同部分的修改。  

关于它的实现原理可以查看[ConcurrentHashMap](https://youngforzy.github.io/2017/11/10/ConcurrentHashMap在JDK1.8中的实现分析/)这篇文章中的分析。
