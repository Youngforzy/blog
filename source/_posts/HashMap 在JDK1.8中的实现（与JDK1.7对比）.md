---
title: HashMap 在JDK1.8中的实现（与JDK1.7对比）
date: 2017-11-1 17:48:18
tags: [HashMap,JDK1.8]
categories: 技术
---

### HashMap的实现分析

#### 介绍
通过前面JDK1.7的分析，我们知道，当负载因子和Hash算法设计的很好时，可以降低hash碰撞的概率，但在数据量过大时也避免不了会出现链表变长的情况，一旦出现链表过长，查找元素变慢，则会严重影响HashMap的性能。  
于是，在JDK1.8中，对数据结构做了进一步的优化，引入了红黑树。**而当链表长度太长（默认超过8）时，链表就转换为红黑树**，利用红黑树快速增删改查的特点提高HashMap的性能，其中会用到红黑树的插入、删除、查找等算法。



#### 底层实现
HashMap的底层实现是**数组+链表+红黑树**。

![image](http://osuskkx7k.bkt.clouddn.com/1.8hash.PNG)

#### 数组索引位置

```
//第一步
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

//第二步，代码中：    
    tab[i = (n - 1) & hash]   
```
确定数组索引的位置同样是两步法：  
第一步计算hash，与JDK1.7中的计算方法不同；
计算过程如下：  

![image](http://osuskkx7k.bkt.clouddn.com/hash.png)  

第二步确定索引，与JDK1.7中的相同，只是不作为一个独立的方法；
#### put()、get()方法
**put()方法**
```
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
调用了putVal()方法：
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //如果tab为空或长度为0，则分配内存resize()
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //数组索引位置为null，直接put
        //同时这一步p赋值为tab[i]
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //判断hash值和key是否都相同，都相同则后面替换value值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //红黑书处理冲突    
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //链表处理冲突
            else {
                for (int binCount = 0; ; ++binCount) {
                    // 在链表尾部插入新结点
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //节点数 >= 7，转化为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) 
                            treeifyBin(tab, hash);
                        break;
                    }
                    //已经存在key，退出循环，后面替换value值
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    //与e = p.next组合，遍历链表
                    p = e;
                }
            }
            //已经存在key，替换value值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
put过程分析：
1. 如果数组索引位置tab[i]为null，直接put；否则进入2；
2. 与第一个节点hash值相同且key值也相同，则直接到后面替换value值，否则进入3；
3. 判断链表是否形成红黑树，并根据结果进入不同的处理。

---

**get()方法**

```
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```
调用getNode()方法：
```
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //判断目标是不是first，是直接返回first
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //已经形成链表
            if ((e = first.next) != null) {
                //第一个节点是TreeNode，说明形成了红黑树
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 还未形成红黑树，按链表处理   
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
方法分析：
1. 首先判断第一个节点first是不是要寻找的节点，如果是直接返回；不是进入2；
2. 判断第一个节点first是不是树节点，如果是说明形成红黑树，调用红黑树的查找方法；不是则进入3；
3. 说明还是链表，未形成红黑树，调用链表查找方法。

#### 扩容机制
我们知道，当往HashMap中不断地添加元素时，它就会扩大数组的长度，把小的数组用大的数组来代替。
**回忆JDK1.7中的扩容，对于链表中的每个元素都需要重新计算hash值，而在JDK1.8中，只需要看看原来的hash值新增的那个bit是1还是0就好了**，是0的话索引没变，是1的话索引变成“原索引+oldCap（原容量）”，工作过程如下图：

数组由16扩大到32的过程中，索引位置为15的元素变化：
![image](http://osuskkx7k.bkt.clouddn.com/1.8hahs%E6%89%A9%E5%AE%B9.png)

**注：JDK1.7中旧链表的元素如果刚好又在新链表中，那么元素的顺序是倒置的，而JDK1.8不会倒置。**

#### JDK1.8和JDK1.7的区别（HashMap）

**相同点**
1. **默认初始容量都是16，默认负载因子都是0.75。数组的长度length都是2的次幂，扩容时都是2倍**
2. **通过hash计算索引的方法相同（hash & length-1）**
3. **key为null的键值对都会放入table[0]中**
4. **都是懒加载，初始时表为空，在插入第一个键值对时初始**化




**不同点**
1. **结构不同，JDK1.8增加了红黑树优化结构**
2. **put方法的区别，JDK1.7中put时，添加到头节点；JDK1.8中添加到尾节点**
3. **计算hash的方法不同，JDK1.8更优化**
4. **JDK1.7新链表的顺序倒置，JDK1.8新链表顺序不倒置**