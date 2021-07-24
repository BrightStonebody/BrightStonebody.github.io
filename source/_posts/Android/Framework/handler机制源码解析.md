---
title: handler机制源码解析
date: 2019-11-09 13:59:20
tags:
- Handler
- 源码解析
categories: 
- Android
---

# Handler机制源码解析

因为handler机制的大概源码网上都有，也很好理解，这里对关键源码做一些理解。

## 1. Looper
主要是loop()方法
```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            // 当queue.next为空时,跳出循环. 但是,实际上当queue为空时,并不会返回null,而是一直阻塞在next()方法中.
            // 只有调用quit方法时,next()方法再能真正的返回null
            return;
        }
        try {
            // msg的target为Handler, 这里是handler中handleMessage()或者callback具体调用的地方
            msg.target.dispatchMessage(msg);
        } finally {
            ... // 省略代码 
        }
        msg.recycleUnchecked();
    }
}
```

## 2. MessageQueue

### 关键成员变量

```java
// mPtr是native的MessageQueue的指针
private long mPtr; // used by native code
// mMessages是消息队列的存储, 数据结构是一个单链表. next属性指向下一个message
Message mMessages;


// 下面两个是IdleHandler的集合, IdleHandler用来在消息队列空闲时做一些操作
private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
private IdleHandler[] mPendingIdleHandlers;

private SparseArray<FileDescriptorRecord> mFileDescriptorRecords;
private boolean mQuitting;

// Indicates whether next() is blocked waiting in pollOnce() with a non-zero timeout.
private boolean mBlocked;

// The next barrier token.
// Barriers are indicated by messages with a null target whose arg1 field carries the token.
private int mNextBarrierToken;
```

### 构造方法
```java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```

### next()方法

```java
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        // 调用native层的方法,使用epoll机制,挂起当前线程. 
        // nextPollTimeoutMillis为挂起时间, 为-1时表示永久挂起
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            // 获取链表的头结点,即第一个message
            Message msg = mMessages;
            // 判断msg是否为同步栅栏
            // 表示同步栅栏的msg, 其msg.target为null
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                // 寻找队列中第一个异步message
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }

            // 处理 msg 
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }


            // msg为null, 有可能是消息队列为空, 有可能是msg的时间没有到

            // Process the quit message now that all pending messages have been handled.
            // 判断是否需要结束循环
            if (mQuitting) {
                // dispose()方法做销毁工作, 会销毁native层的队列, 并把mPtr置0
                // 只有在这里next()方法才会返回null
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            // 当获取到的message为空, 或者message的执行时间没有到
            // 当前没有msg可以处理的，开始处理 idleHandler
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                // 走到这里表示该线程需要阻塞一段时间, 阻塞时间为nextPollTimeoutMillis
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        // 在消息队列空闲时, 遍历idleHandler集合, 执行idleHandler的相关方法
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

#### 1. nativePollOnce(ptr, nextPollTimeoutMillis);
这个方法在native层调用epoll机制, 讲当前线程挂起nextPollTimeoutMillis的时间. 
**特别的nextPollTimeoutMillis = -1时, 永久挂起, 需要java层调用nativeWake()方法才能唤醒.** 
nextPollTimeoutMillis = -1 出现的时机两种情况:
* 消息队列为空
* 遇到了同步barrier, 并且之后没有遇到异步的msg

#### 2. IdleHandler:
```java
public static interface IdleHandler {
    /**
        * Called when the message queue has run out of messages and will now
        * wait for more.  Return true to keep your idle handler active, false
        * to have it removed.  This may be called if there are still messages
        * pending in the queue, but they are all scheduled to be dispatched
        * after the current time.
        */
        // queueIdle()返回值为false, 表示这个IdleHandler仅仅执行一次就自动销毁; 返回为true, 表示可以执行多次
    boolean queueIdle();
}
```
MessageQueue中有两个IdleHandler的集合, IdleHandler是一个接口, 实现了这个接口的handler可以用来在MessageQueue空闲时, 做一些操作, 即 获取message为null时(可能是队列为空, 也可能是时间没到)

#### 3. 同步栅栏 Barrier

* 定义: target为null的msg为同步栅栏
* 作用: 当出现一个Barrier, 跳过所有的同步msg, 只执行异步msg
* 意义: 相当于为msg添加了一个优先级

### quit()

```java
void quit(boolean safe) {
    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            // 这个方法会去除所有的msg.when > now 的msg
            removeAllFutureMessagesLocked();
        } else {
            // 没有任何判断, 直接去除所有的msg
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        // 因为在next()方法中才会调用dispose()方法销毁消息队
        // 并且可能当前线程是阻塞的. 所以需要先要唤醒当前线程, 继续执行next()方法
        nativeWake(mPtr);
    }
}
```

### enqueueMessage

```java
boolean enqueueMessage(Message msg, long when) {
    ... // 省略代码

    synchronized (this) {
        if (mQuitting) {
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            // 添加到队列头部, 有可能是需要wake的
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            // 插入到队列的中间位置. 通常这种情况下, 我们不需要wake线程. 除非有一种请求, 就是队列的头msg是一个同步barrier, 并且新插入的msg是一个异步消息
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}

```

#### **1. 插入顺序:**
之前说了存储msg的数据机构是一个单链表, 并且链表是按照msg.when排序的(按执行的时间顺序排序). 所以插入时要寻找合适的位置, 保证排序

#### **2. needWake**
判断是否需要唤醒线程. 
在以下情况下, needWake会为true:
* 消息队列为空, 并且当前线程是blocked的. 这种情况下nativePollOnce方法执行过, 目标线程被挂起, 并且nextPollTimeoutMillis = -1
* 消息队列不为空, 并且当前线程是blocked的, 新插入msg.when < 头结点p.when. 这种情况表示, 消息队列中头结点的时间还没到, 目标线程已经被挂起了, 但是新插入的msg执行时间小于头结点的时间, 此时当然应该唤醒线程先执行新msg
* 消息队列不为空, 但是消息队列头结点是一个同步barrier, 并且新插入的msg为异步消息. 这种情况下nativePollOnce方法同样执行过, 目标线程被挂起, 并且nextPollTimeoutMillis = -1

