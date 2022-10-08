---
title: ThreadLocal
date: 2019-07-22 13:57:12
tags:
- 多线程
categories:
- java
- 多线程
---

# ThreadLocal

## 作用

**当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立的改变自己的副本，而不会影响其他线程所对应的副本。从线程的角度来看，目标变量就像是本地变量**

## 源码分析

### set()

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

createMap()方法：
```java
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
查看Thread类的代码：
```java
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```

**Thread类中有一个Thread.ThreadLocalMap成员变量。也就是说，ThreadLocal的set方法，实质是在Thread的threadlocals成员中添加value**

### get()方法
```java
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
其中的ThreadLocalMap.Entry
```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
**Entry中的key时ThreadLocal的弱引用， 所以在ThreadLocal的get方法中map.getEntry(this)传入的是自己**

### 总结
**Thread类中有一个ThreadLocalMap成员对象，这个变量在Thread类中并没有使用，只有在ThreadLocal类中才有用到。ThreadLocalMap里的Entry固定为ThreadLocal的弱引用。也就是说ThreadLocal是将value存储到Thread中，调用get和set方法是，先得到当前的Thread线程对象，获取到thread中的map，在以自己为key获取到value值**
