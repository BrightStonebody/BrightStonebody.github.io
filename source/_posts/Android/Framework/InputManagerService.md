---
title: InputManagerService
date: 2021-10-18 21:31:03
tags:
---

## native层概述

### InputReader 

输入事件的读取者。 运行在单独的 InputReaderThread 线程中，死循环，通过 EventHub.getEvents 来获取输入事件。

InputReader 持有 InputMapper 对应不同的输入设备的 事件加工器。 

事件加工后，InputReader 会 调用 InputDispatcher.notifyXXX 函数 唤醒 InputDispatcher 进行事件分发

### InputDispatcher

输入事件的分发者。 分发输入事件到目标窗口。 运行在 InputDispatcherThread 中， 这个线程平时是休眠状态，等待 InputReader 的唤醒

被唤醒后，调用 dispatchOnceInnerLocked 将输入事件分发到合适的窗口中

