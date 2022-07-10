---
title: mmkv原理
date: 2022-07-10 17:03:32
tags:
- glide
- 源码解析
---

## SharedPreferences的痛点

### 只能全量读取和写入
sp保存数据的文件形式是xml。每次读取都去读xml文件，解析后加入内存缓存里。每次写入都更改内存缓存的值，然后全量写入到文件里

### 有anr的风险
* 在第一次载入时会产生anr
有一个加锁配合while循环，如果第一次使用xml文件还没有加载完成，就会pending。xml文件的全量读取本身就不快，如果xml文件过大，在首次使用时就可能会anr，在app启动过程中问题会比较明显。
```java
    @Override
    public boolean getBoolean(String key, boolean defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Boolean v = (Boolean)mMap.get(key);
            return v != null ? v : defValue;
        }
    }

    private void awaitLoadedLocked() {
        if (!mLoaded) {
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) {
            try {
                mLock.wait();
            } catch (InterruptedException unused) {
            }
        }
        if (mThrowable != null) {
            throw new IllegalStateException(mThrowable);
        }
    }
```
* commit写入导致anr
没啥说的，同步写入文件

* apply写入导致anr
apply会异步写入，先写入到内存缓存，然后在子线程更新xml文件。
但是sp会将更新文件的问题加入到QueuedWork中。android的系统会在Activity的onStop,onPause等生命周期中，调用QueuedWork.waitToFinish，等待落盘的任务队列执行完成，如果任务队列中的任务很多，或者待写入的数据量很大时(sp文件是全量读写的)，在一些io性能差的中低端机型上就会很容易出现anr.

具体可以参考这个文章：https://juejin.cn/post/6844904033820377096
https://www.jianshu.com/p/9ae0f6842689

## 内存映射
通过 mmap 内存映射文件，提供一段可供随时写入的内存块，App 只管往里面写数据，由操作系统负责将内存回写到文件，不必担心 crash 导致数据丢失。

* 普通io文件操作的过程
Linux系统内存分为 内核空间 和 用户空间 。文件操作属于 内核空间。应用需要通过系统调用才能完成文件读取。
普通io操作需要先将文件拷贝到页缓存里(内核空间)，然后再从页缓存拷贝到用户空间。这需要两次拷贝。
* mmap文件读写
mmap创建了 用户空间 到 内核空间 的映射，返回对应的指针；
读取文件时，文件还是一样拷贝到页缓存里(内核空间)。 因为创建了内存映射，用户空间能直接获取到文件数据。
写文件时，直接写入到指针地址对应的内核空间，Linux系统机制会保证内核空间到文件磁盘的写入，这样即使进程crash，也不会导致数据丢失

## 数据存储优化

* 数据组织
数据序列化方面我们选用 protobuf 协议，pb 在性能和空间占用上都有不错的表现。
* 写入优化
考虑到主要使用场景是频繁地进行写入更新，我们需要有增量更新的能力。我们考虑将增量 kv 对象序列化后，append 到内存末尾。
这样同一个 key 会有新旧若干份数据，最新的数据在最后。写入速度会很快
读取只需要从后往前读，读到第一个key匹配即使最新的有效数据。
* 空间增长
使用 append 实现增量更新带来了一个新的问题，就是不断 append 的话，文件大小会增长得不可控。我们需要在性能和空间上做个折中。
以内存 pagesize(内存页大小) 为单位申请空间，在空间用尽之前都是 append 模式；当 append 到文件末尾时，进行文件重整(去除重复key)、key 排重，尝试序列化保存排重结果；在程序启动第一次打开 mmkv 时，也会进行文件重整。 
排重后空间还是不够用的话，将文件扩大一倍，直到空间足够。

## 多进程读写支持
在mmap的时候，只会返回映射内存的指针。在append和重整时可以加文件锁，保证自己的写文件成功。但是一个进程并不知道自己文件的数据是否被其他进程修改过。
* 写指针的同步
因为mmkv的写入是直接append在数据末尾，mmap导致数据的大小和文件的大小并不一直，所以需要一个指针指向有效数据的末尾，称之为写指针。
mmkv 在每个进程内部缓存自己的写指针，然后在写入键值的同时，还要把最新的写指针位置也写到 mmap 内存中；这样每个进程只需要对比一下缓存的指针与 mmap 内存的写指针，如果不一样，就说明其他进程进行了写操作。
事实上 MMKV 原本就在文件头部保存了有效内存的大小，这个数值刚好就是写指针的内存偏移量，可以重用这个数值来校对写指针。
* 内存重整的感知
使用一个单调递增的序列号，每次发生内存重整，就将序列号递增。将这个序列号也放到 mmap 内存中，每个进程内部也缓存一份，只需要对比序列号是否一致，就能够知道其他进程是否触发了内存重整。
* 内存增长的感知
事实上 MMKV 在内存增长之前，会先尝试通过内存重整来腾出空间，重整后还不够空间才申请新的内存。所以内存增长可以跟内存重整一样处理。至于新的内存大小，可以通过查询文件大小来获得，无需在 mmap 内存另外存放。

可以参考文档 https://cloud.tencent.com/developer/article/1354199


## mmkv的不足

主要是mmap的缺点，需要提前确定好文件的大小，即 映射内存的大小。但是，映射内存的大小并不是有效数据的大小。 映射内存只是提供了一个快捷操作文件的场地。 会造成存储空间的浪费，并且映射内存的大小必须是页缓存(pagesize)的整数倍