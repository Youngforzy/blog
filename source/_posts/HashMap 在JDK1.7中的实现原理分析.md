---
title: HashMap 在JDK1.7中的实现原理分析
date: 2017-10-30 18:48:18
tags: [HashMap,JDK1.7]
categories: 技术
---
### 一、HashMap的介绍
HashMap是存储键值对（key，value）的一种数据结构。  
每一个元素都是一个key-value。  
HashMap最多只允许一个key为null，允许多个key的value值为null。  
HashMap是非线程安全的，只适用于单线程环境。  
HashMap实现了Serializable、Cloneable接口，因此它支持序列化和克隆。


### 二、HashMap的实现原理


从**底层结构、put和get方法、hash数组索引、扩容机制**等几个方面来分析HashMap的实现原理：
#### 1.底层结构
HashMap的底层结构是由**数组+链表**构成的。  

![image](http://osuskkx7k.bkt.clouddn.com/hash2.PNG)


数组（紫色）：hash数组（桶），数组元素是每个链表的头节点  
链表（绿色）：解决hash冲突，不同的key映射到了数组的同一索引处，则形成链表。

**构成链表的节点类Node：** （jdk 1.7 中的源码）
```
static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
        ....
}
```
可以看到，key和value都储存于节点之中，next表示下一个节点。

#### 2.put、get方法
**put()方法：**

```
public V put(K key, V value) {  
    // 若key为null，则将该键值对添加到table[0]中。  
    if (key == null)  
        return putForNullKey(value);  
    // 若key不为null，则计算该key的hash值，然后将其添加到该哈希值对应的数组索引处的链表中。
    int hash = hash(key.hashCode());  
    int i = indexFor(hash, table.length);  
    //遍历该数组索引位置处的链表
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {  
        Object k;  
        // 若该key对应的键值对已经存在，则用新的value替换旧的value，退出  
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {  
            V oldValue = e.value;  
            e.value = value;  
            e.recordAccess(this);  
            return oldValue;  
        }  
    }  
    modCount++;
    // 若数组索引位置处table[i]没有链表，即没有元素
    // 则将key-value添加到数组索引table[i]处，成为头节点
    addEntry(hash, key, value, i);  
    return null;  
}
```
put()方法大概过程如下：
1. **如果添加的key值为null，那么将该键值对添加到数组索引为0的链表中，不一定是链表的首节点。**
2. **如果添加的key不为null，则根据key计算数组索引的位置**：  
- **数组索引处存在链表**，则遍历该链表，如果发现key已经存在，那么将新的value值替换旧的value值
- **数组索引处不存在链表**，将该key-value添加到此处，成为头节点

**addEntry()方法如下：**

```
void addEntry(int hash, K key, V value, int bucketIndex) {  
    // bucketIndex是数组位置索引，保存“bucketIndex”位置的值到“e”中  
    Entry<K,V> e = table[bucketIndex];  
    // 设置“bucketIndex”位置的元素为“新Entry”，设置“e”为“新Entry的下一个节点”  
    table[bucketIndex] = new Entry<K,V>(hash, key, value, e);  
    // size超过阈值，则调整HashMap的大小  
    if (size++ >= threshold)  
        resize(2 * table.length);  
}
```
**将新的节点（假设为节点n）添加到数组索引位置处，将原来的节点e作为n的next节点，即下一个节点。**

由此可知：**每一次添加的新节点总是作为头节点。**

---

**get()方法：**

```
public V get(Object key) {  
    if (key == null)  
        return getForNullKey();  
    // 获取key的hash值  
    int hash = hash(key.hashCode());  
    // 在“该hash值对应的链表”上查找“键值等于key”的元素  
    for (Entry<K,V> e = table[indexFor(hash, table.length)];  
         e != null;  
         e = e.next) {  
        Object k;  
        //判断key是否相同
        if (e.hash == hash && ((k = e.key) == key || key.equals(k)))  
            return e.value;  
    }
    //没找到则返回null
    return null;  
}  
 
// 获取“key为null”的元素的值  
// HashMap将“key为null”的元素存储在table[0]位置，但不一定是该链表的第一个位置！  
private V getForNullKey() {  
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {  
        if (e.key == null)  
            return e.value;  
    }  
    return null;  
}
```
get()方法的大概过程：
1. 如果key为null，那么在数组索引table[0]处的链表中遍历查找key为null的value
2. 如果key不为null，根据key找到数组索引位置处的链表，遍历查找key的value，找到返回value，若没找到则返回null



#### 3.hash数组索引位置

前面多次提到了数组索引位置，那么这个位置该如何确定呢？  
两步：
1. 确定key的hash值
2. 根据hash计算索引

```
static int hash(int h) {
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
    
static int indexFor(int h, int length) {
        return h & (length-1);
    }
```
第一步：采用位操作计算hash值，这么做的**目的是为了下一步的索引值分布均匀，减少碰撞，提高效率。**  
第二步：根据hash值计算索引的值，把hash值和数组长度-1做一个"与"操作，保留的值作为索引。

**为什么是length-1？**  
**这也是HashMap的数组长度要取2的整次幂的原因之一。  因为length为2的整数次幂时，（length-1）正好相当于一个“低位掩码”****。“与”操作的结果就可以保留低位作为数组索引。**  
例如：length = 16和某个hash值进行 & 操作


```
某个hash           10110110 11100010 01000101
length-1=15    &   00000000 00000000 00001111
           ---------------------------------
                   00000000 00000000 00000101   //只保留末四位，索引值=5
```
**另一个原因**：HashMap的数组长度length为2的整次幂时，length-1为奇数（偶数-1），此时进行 & 操作时可保证最后一位可能是 0 或 1 ，保证了数组索引的均匀性；而如果length-1为偶数，那么 & 操作时最后一位只能是 0，也就是数组索引只能是偶数索引位置，这样就浪费了一半的空间，所以length为2的整次幂。  
**综上，数组长度取2的整次幂，是为了减少hash碰撞的概率，使元素散列更加均匀。**
#### 4.扩容机制

先看一个例子，创建一个HashMap，初始容量默认为16，负载因子默认为0.75，那么什么时候它会扩容呢？
来看以下公式：
```
实际容量 = 初始容量 × 负载因子
```
计算可知，16×0.75=12，也就是当实际容量超过12时，这个HashMap就会扩容。
  
**初始容量**  

当构造一个hashmap时，初始容量设为不小于指定容量的2的次方的一个数（new HashMap(5)， 指定容量为5，那么实际初始容量为8，2^3=8>5），且最大值不能超过2的30次方。  

**负载因子**  

**负载因子是哈希数组在其容量自动增加之前可以达到多满的一种尺度。（时间与空间的折衷）** 当哈希数组中的条目数超出了加载因子与初始容量的乘积时，则要对该哈希数组进行扩容操作（即resize）。  
**特点：**

- **负载因子越小，容易扩容，浪费空间，但查找效率高**
- **负载因子越大，不易扩容，对空间的利用更加充分，查找效率低（链表拉长）**

**扩容过程**  

 HashMap在扩容时，**新数组的容量将是原来的2倍**，由于容量发生变化，原有的每个元素需要重新计算数组索引Index，再存放到新数组中去，这就是所谓的rehash。

扩容代码：
```
void resize(int newCapacity) {  
    Entry[] oldTable = table;  
    int oldCapacity = oldTable.length;  
    if (oldCapacity == MAXIMUM_CAPACITY) {  
        threshold = Integer.MAX_VALUE;  
        return;  
    }  
    // 新建一个HashMap，将“旧HashMap”的全部元素添加到“新HashMap”中
    // 然后，将“新HashMap”赋值给“旧HashMap”。  
    Entry[] newTable = new Entry[newCapacity];  
    transfer(newTable);  
    table = newTable;  
    threshold = (int)(newCapacity * loadFactor);  
}
```
调用transfer()方法
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
第一步for循环遍历每一个数组元素（桶）；  
第二步遍历每一个数组元素中的链表，将链表中的节点存入新数组指定的位置构成链表，注意此时新链表顺序反转。（原链表的头节点将变成新链表的尾节点）
**注：原链表中的节点可能存到不同的新链表中，因为rehash重新计算了索引位置。**

由此可知，**扩容操作是一个耗时耗性能的操作，因为它需要重新计算元素的位置，并进行复制操作。因此，在使用时提前预估HashMap的大小有助于提高性能。**

HashMap未初始容量和初始容量的对比：

```
long start1 = System.currentTimeMillis();
		HashMap map = new HashMap<>(); //未初始容量
		for(int i=0;i<20000;i++){
			map.put(i, "I am zy");
		}
		long end1 = System.currentTimeMillis();
		System.out.println("不初始化时耗时："+(end1-start1)+ "ms");
		
		long start2 = System.currentTimeMillis();
		HashMap map2 = new HashMap<>(32768); //初始容量
		for(int i=0;i<20000;i++){
			map2.put(i, "I am zy");
		}
		long end2 = System.currentTimeMillis();
		System.out.println("初始化时耗时："+(end2-start2)+ "ms");
```
```
不初始化时耗时：5ms
初始化时耗时：1ms
```
可见初始化容量有助于提高性能，对于数据量大则越明显。

