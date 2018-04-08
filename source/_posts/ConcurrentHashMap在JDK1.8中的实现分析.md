---
title: ConcurrentHashMap在JDK1.8中的实现分析
date: 2017-11-10 16:42:18
tags: [ConcurrentHashMap,JDK1.8]
categories: 技术
---
### 一、ConcurrentHashMap的介绍
**ConcurrentHashMap在JDK8中进行了巨大改动，它舍弃了锁分段的技术，大量引入了CAS操作，以此来实现并发操作。**   

回忆JDK1.7中的ConcurrentHashmap，当hash碰撞频繁时，链表长度会拉长，而链表的增改删查操作都会消耗很长的时间，影响性能，因此和JDK1.8中的HashMap一样，当链表过长时，将其结构转化为红黑树，由此提高性能。
### 二、ConcurrentHashMap的实现原理

**ConcurrentHashMap的结构图**

ConcurrentHashMap在JDK1.8的实现原理是“**数组+链表+红黑树**”。（与HashMap在1.8中的实现思想一致，但是**红黑树的节点不同**，HashMap是Node节点，ConcurrentHashMap是TreeBin对象）

![image](http://osuskkx7k.bkt.clouddn.com/1.8hash.PNG)


---

下面介绍一下ConcurrentHashMap主要方法put()和get()的实现。
#### put()方法


```
final V putVal(K key, V value, boolean onlyIfAbsent) {
---第一部分
        if (key == null || value == null) throw new NullPointerException();
        //计算hash值
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //如果tab为null，则初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            //判断数组索引位置的元素是否为null
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //CAS操作设置该数组索引位置为新节点Node
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            //f节点是MOVED节点，表示有其他线程在扩容，帮助一起扩容
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
                
---第二部分-----
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        //表示是链表，还未转化成红黑树
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //如果key已存在，则替换value
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //将新的节点插入尾部
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //节点f是TreeBin对象，表示链表转为了红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    //如果大于8，转化为红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```
由于整个put()方法较长，分成两部分来分析。  
**第一部分：**
1. 遍历数组tab，如果为null，初始化数组；
2. **调用tabAt()方法查找数组索引i处的节点f，如果f为null，说明该位置还没有节点，调用casTabAt()利用CAS操作插入新的节点**
- **CAS成功**：break跳出，直到最后的addCount(1L, binCount)方法，判断插入这一个节点后是否需要扩容; 
- **CAS失败**：说明有其它线程提前插入了节点，自旋重新尝试在这个位置插入节点；  
tabAt()和casTabAt()方法源码如下：
```
 private static final sun.misc.Unsafe U;
 
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
```
可以看到，都是**调用Unsafe类的方法（原子性），Unsafe.getObjectVolatile可以直接获取指定内存的数据，保证了每次拿到数据都是最新的。**
3. **若前面的情况都不满足，判断f节点是否为MOVED节点，是则表示有其他线程在扩容，帮助一起扩容**；否则进入第二部分。  

**第二部分：**  

第二部分表示把新的节点Node插入链表或红黑树，可以看到使用了synchronized关键字实现同步。**但是注意，只在节点f上进行同步，表示只能有一个线程访问该节点。** 节点插入之前，再次利用 tabAt(tab, i) == f 判断头节点是否还是f，防止被其它线程修改。
1. 如果f.hash >= 0，说明f是链表结构的头结点，遍历链表，如果key已存在，则修改value，否则在链表尾部插入节点。
2. 如果f是TreeBin类型节点，说明链表变成红黑树，则在树结构上遍历元素，更新或增加节点。
3. 最后判断链表中的节点数binCount >= 8，则转化为红黑树。

---

#### get()方法

```
public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        //获取key的hash值
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            //遍历    
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```
1. 如果table为null或者遍历之后没找到对应的value，返回null；
2. 根据key的hash值找到table中指定的Node节点，遍历链表或红黑树找到对应的value值。
#### 总结

最后总结一下**ConcurrentHashMap从JDK1.7（1.6）到JDK1.8的变化**：
1. **底层结构改变**，从“**数组+数组+链表**”到“**数组+链表+红黑树**”
2. **锁方式改变，取消了Segment重入锁，变成CAS+Synchronized实现锁**
3. **锁粒度变小**，**由Segment数组变成table的元素**。
4. JDK1.8中size()实现更简单