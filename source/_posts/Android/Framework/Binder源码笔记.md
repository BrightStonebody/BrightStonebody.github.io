---
title: Binder源码笔记
date: 2021-11-03 21:42:37
tags:
---

# Service 的启动 (独立进程的 Service )
## ProcessState
ProcessState 是一个系统 Service 进程的单例，一个进程只有一个 ProcessState 实例。 

内部会打开设备 "/dev/binder" ，调用 mmap 函数，完成自己和内核空间的内存映射。

ioctl 和 Binder 设备传递参数，设置了 binder 线程池的最大线程数

## Binder线程池
调用 ProcessState::startThreadPool() 启动 Binder 线程池。 调用之后，实际上就已经 run 了一个 Binder 主线程, 并开始 while 死循环接收 Binder 消息。 
之后会根据情况，选择在 Binder 线程池中创建新的线程，但是创建出来的就不是一个死循环的线程了

# 通信
## BpBinder 和 BBinder 
IBinder 对象是负责binder通信的实例。 BpBinder 是客户端和服务端交互的代理类，BBinder 代表服务端

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