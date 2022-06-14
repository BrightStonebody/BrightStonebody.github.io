---
title: HashMap
date: 2022-04-04 14:23:14
tags:
---

## 1. 基本数据结构

Node<K, V> 的一个table数组，容量始终为2的幂，Node代表一个entry。在put一个元素时，懒加载分配数组空间。
如果实例化HashMap的时候传入一个 initialCapacity ，table的大小会给一个最接近的2的幂的大小
如果遇到hash冲突，会变为一个链表或红黑树，数组中的node作为链表or红黑树的头节点，新加入的节点插入到链表中。 在java1.8，如果新加入的链表长度超过8，会转化为红黑树

## 2. 加入的元素如何确定在数组中的 index

```java
static int indexFor(int h, int length) {  //jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的
     return h & (length-1);  //第三步 取模运算
}
```

要把新加入的元素均匀分布到数组中，我们首先想到的就是把hash值对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。但是，模运算的消耗还是比较大的. 但是jdk通过h & (table.length -1)来得到该对象的保存位，这样运算会加快。

## 3. 如何扩容

将数组的容量扩大为原来的两倍，原数组长度为oldCap，下标j的元素会被分散到 数组j 和 j+oldCap 的位置。 

```java
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                    // 数组下表位置如果是 红黑树，逻辑和下面的链表差不多
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                    // 数组下表位置如果是 链表
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 因为 oldCap 始终为2的倍数。 (e.hash & oldCap) 只会有0或1两个值，由此决定是位置 j 还是 j+oldCap
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }

```

## 4. 扩容的时机

```java
if (++size > threshold)
    resize();
```
当size大于一个阈值的时候，就会开始扩容。 `threshold = length * loadFactor` 是数组长度*负载因子。 负载因子默认是0.75 ，注意这个length是table数组的长度而不是HashMap的整体size（为了保证获取元素的时间复杂度接近1）。

loadFactor 是时间复杂度和空间复杂度的权衡。 如果内存空间很多而又对时间效率要求很高，可以降低负载因子Load factor的值；相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子loadFactor的值，这个值可以大于1。

## 5. HashMap是线程不安全的。HashTable是线程安全的。
HashTable的线程安全很粗暴，将所有的map操作加一个synchronized修饰。


参考：
https://tech.meituan.com/2016/06/24/java-hashmap.html





