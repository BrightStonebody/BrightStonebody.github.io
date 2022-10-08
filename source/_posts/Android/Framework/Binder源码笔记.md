---
title: Binder源码笔记
date: 2021-11-03 21:42:37
tags:
---

## ProcessState
ProcessState 是一个系统 Service 进程的单例，一个进程只有一个 ProcessState 实例。 zygote 在fork出应用进程之后会在c层完成初始化启动 binder

内部会打开设备 "/dev/binder"（binder驱动）；调用 mmap 函数，完成自己和内核空间的内存映射，分配缓冲区；启动binder线程，不断循环和binder驱动交互

ioctl 和 Binder 设备传递参数，设置了 binder 线程池的最大线程数

## Binder线程池
调用 ProcessState::startThreadPool() 启动 Binder 线程池。 调用之后，实际上就已经 run 了一个 Binder 主线程, 并开始 while 死循环接收 Binder 消息。 
之后会根据情况，选择在 Binder 线程池中创建新的线程，但是创建出来的就不是一个死循环的线程了

# 通信
## BpBinder 和 BBinder 
IBinder 对象是负责binder通信的实例。 BpBinder 是客户端和服务端交互的代理类，BBinder 代表服务端
在BpBinder类的内部有一个成员变量mHandle，它是一个int型变量。
对于每一个经过binder driver传输的BBinder对象，binder driver都会在驱动层为它构建一个binder_node数据结构；同时为这个binder_node生成一个“引用”：binder_ref
BpBinder的成员变量mHandle的值就是bidner_ref中的int型描述符，这样就建立起了用户层的Bpbinder和一个驱动空间的binder_ref数据结构的对应关系。通过 “BpBinder handle→binder_ref→binder_node→BBinder”这样的匹配关系，就可以建立一个Bpbinder对应一个BBinder的关系。 

## BpBinder::transact
发送数据的方法

## IPCThreadState::self()->transact(...)
真正数据通信的方法。 一个线程只会有一个实例， 使用 pthread_setspecific 实现，类似于 java 的 ThreadLocal
IPCThreadState 有两个缓存 mIn 和 mOut 分别负责数据的接收和发射

```shell
-> transact(..)
    -> writeTransactionData(..)
        -> 将 cmd 和 data 写入到 mOut 中
    -> waitForResponse(..)
        -> talkWithDriver(doReceive)
            -> 将 bwr(一个和Binder驱动的结构体) bwr.write_buffer 指向 mOut.data()
            -> 将 bwr.read_buffer 指向 mIn.data()
            -> do-while 循环，调用 ioctl 函数 和 Binder 驱动通信，直到结束
```



# 问题：
## 1. 只有系统Service 才能记录到 ServiceManager 中，那么 应用的 Service 是如何查找到的
应用 Service 是发布到 AMS 上的，客户端从 AMS 上去查找 Service 的 Binder 对象

## 2. Serializable 和 Parcelable
Serializable 序列化时使用了大量的反射， Parcelable 序列化过程是用户自定义的， Parcelable 更优越。
Serializable 更适合用于网络传输和本地存储， 因为 Parcelable 可能由于安卓系统    。