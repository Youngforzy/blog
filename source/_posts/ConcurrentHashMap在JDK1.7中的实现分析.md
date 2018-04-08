---
title: ConcurrentHashMap在JDK1.7中的实现分析
date: 2017-11-6 17:48:18
tags: [ConcurrentHashMap,JDK1.7]
categories: 技术
---
### 一、ConcurrentHashMap的介绍
ConcurrentHashMap是线程安全且高效的HashMap，可以在多线程的环境下使用。  
**ConcurrentHashMap允许多个线程并发访问，其关键在于使用了锁分段技术。**   
锁分段：首先将数据分成一段一段地存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个数据段的时候，其他数据段也能被其他线程访问。

### 二、ConcurrentHashMap的实现原理
**ConcurrentHashMap的结构图**  

ConcurrentHashMap在JDK1.8之前的实现原理是“**数组+数组+链表**”。（可能描述不妥）
![image](http://osuskkx7k.bkt.clouddn.com/concurrentHashmap.PNG)

  
第一个数组是Segment[ ]，每一个Segment类似于HashMap；  
第二个数组是HashEntry[ ]，每个元素可能是一个链表；  
链表是HashEntry形成的链表，HashEntry是一个节点。

---

#### HashEntry类
```
static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
        ....
        }
```
HashEntry类相当于HashMap中的Entry类（节点类），储存了key和value，并拥有指向下一个元素的引用next。  
注：**value采用volatile修饰，保证了线程之间的可见性**。
#### Segment类
```
 static final class Segment<K,V> extends ReentrantLock implements Serializable {
 
         transient volatile HashEntry<K,V>[] table;
         transient int count;
         transient int modCount;
         transient int threshold;
         final float loadFactor; 
         ....
         }
```
每个Segment都拥有一个HashEntry[]数组，还有threshold和loadFactor分别表示极限容量和负载因子，count表示元素个数，modCount表示修改的记录。如此看来，**每个Segment就好比是一个缩小版的HashMap**，从上面ConcurrentHashMap结构图也可以看出来。

**Segment继承自ReentrantLock重入锁，因此它支持一个线程重进入同一个Segment，访问其中的数据。**

---
下面介绍一下ConcurrentHashMap主要的几个方法的实现:put()方法、get()方法、size()方法
#### put()方法

```
public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject
             (segments, (j << SSHIFT) + SBASE)) == null) 
            // 扩容 
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```
首先定位到相应的Segment。
如果需要扩容则进入ensureSegment(j)方法，**注意ConcurrentHashMap不会对整个容器扩容，而只对当前的Segment进行扩容。**  扩容为原来的2倍。  
如果不需要扩容，调用Segment中的put()方法：
```
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;
                //取出头节点
                HashEntry<K,V> first = entryAt(tab, index);
                //遍历链表
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {
                        K k;
                        //key已存在，替换value
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {
                        //scanAndLockForPut()方法中已经返回了node，设置为first节点
                        if (node != null)
                            node.setNext(first);
                        else
                            //新建HashEntry节点作为头节点first
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
                        //判断是否扩容
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount; //记录修改次数
                        count = c;  //修改count值
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }

```
过程分析：  

1. 首先调用tryLock()方法尝试获取锁，如果获取失败，则进入scanAndLockForPut()方法，该方法实际上是先自旋一定的次数等待其他线程释放锁。若自旋过程中，其他线程释放了锁，导致本线程直接获得了锁，就避免了本线程进入等待锁的场景，提高了效率。若自旋一定次数后，仍未获取锁，则调用lock方法进入等待锁的场景。（这是JDK1.7的实现，如果是1.6则没有自旋，直接获取锁）
2. 获取锁成功，找到对应的链表作相应的操作。具体见代码注释。

#### get()方法
get()方法不需要锁。**因为value字段是volatile修饰，保证了线程之间的可见性，可以被多线程同时读，但只能被单线程写。一句话，get操作只需要读共享变量value，所以不用加锁。**

```
public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key); //第一次散列
        //第二次散列
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            //第三次散列（for循环中）
            //遍历链表
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }

```
（get()方法中用到了许多UNSAFE类的方法，这是在JDK1.6中没有的，主要是利用Native方法来快速的定位元素。）  

get()的过程经过了三次散列：  

**第一次：对key进行散列得到h  
第二次：对h进行散列定位到哪个Segment  
第三次：对h进行散列定位到哪个HashEntry** 

定位到HashEntry之后就对该链表遍历，查找key对应的value，若没有找到则返回null。  

#### size()方法
**size()方法需要跨Segment操作，因为要统计每个Segment中的count值。而count值是volatile变量，一般来说将所有的count变量相加就可以得到整个ConcurrentHashMap的大小。** 但可能在累加前使用的count发生了变化，那么结果就不正确。那么该如何统计呢？
1. 第一种方法，将所有Segment的put、remove、clean方法都锁住，然后统计count值。做法可行，但是低效。
2. **第二种方法，先尝试连续2次不通过锁住Segment的方式计算各个count值的和：**
-  若没有发生变化，则作为size的大小。
-  若发生变化说明有线程在操作元素，则锁住Segment统计所有的count值。


**如何判断容器大小没有发生变化？**   

**modCount变量。这个变量记录了每个Segment中put、remove、clean等操作的次数，因此在连续两次统计count的值时，比较modCount是否变化，就可得知容器大小是否变化。**