---
title: OkHttp3源码-发起请求的主要流程
date: 2019-11-13 21:21:04
tags:
- Okhttp3
- 源码解析
categories:
- Android
---

# OkHttp3源码解析-发起请求的主要流程

结合网上的博客和自己看的源码, 写的简单理解.

## 主要的类:
* OkHttpClient:
    用户对请求进行配置的类, 配置拦截器, cache管理, 超时时间等
* Request:
    用户对单次请求的数据进行配置, uur, 数据参数等. 
* Call:
    在框架内部表示对请求的封装
* Dispatcher:
    在框架内部对请求进行分发

## OkHttpClient:

主要是一些请求的配置操作, 这里没什么好说的. 看一下最常用的方法newCall的调用.

```java
/**
* Prepares the {@code request} to be executed at some point in the future.
*/
@Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
}
```

## Call:

### 构造方法
```java
  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    this.timeout = new AsyncTimeout() {
      @Override protected void timedOut() {
        cancel();
      }
    };
    this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
  }

  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
```

### 同步执行一个call

在RealCall中有一个execute方法, 这里发起一个同步请求
```java
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    timeout.enter();
    eventListener.callStart(this);
    try {
      client.dispatcher().executed(this);
      // getResponseWithInterceptorChain()是具体的请求的操作过程
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      e = timeoutExit(e);
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

**getResponseWithInterceptorChain()方法是请求发出的最终执行方法.** 也就是说RealCall中调用execute()就已经直接同步开始了请求操作.

但是dispatcher是请求的分发类, 如果, 直接开始了请求操作, 那么在dispatcher.executed方法和dispatcher.finished方法都干了什么呢? 

详细的内容可以看Dispatcher的源码解析.

### 异步执行一个call

RealCall中的enqueue方法, 发起一个异步请求
```java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
可以看到和同步方法不同: 异步方法没有直接调用getResponseWithInterceptorChain方法开始发起请求, 而是老老实实的交给dispatcher()去调度. 还有就是, 调用dispatcher.enqueue方法, 参数是一个AsyncCall, 这个类是RealCall的内部类

**AsyncCall中的executeOn, execute**
```java
/**
* Attempt to enqueue this async call on {@code executorService}. This will attempt to clean up
* if the executor has been shut down by reporting the call as failed.
*/
void executeOn(ExecutorService executorService) {
      assert (!Thread.holdsLock(client.dispatcher()));
      boolean success = false;
      try {
        executorService.execute(this);
        success = true;
      } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        eventListener.callFailed(RealCall.this, ioException);
        responseCallback.onFailure(RealCall.this, ioException);
      } finally {
        if (!success) {
          client.dispatcher().finished(this); // This call is no longer running!
        }
      }
    }

@Override protected void execute() {
  boolean signalledCallback = false;
  timeout.enter();
  try {
    Response response = getResponseWithInterceptorChain();
    signalledCallback = true;
    responseCallback.onResponse(RealCall.this, response);
  } catch (IOException e) {
    e = timeoutExit(e);
    if (signalledCallback) {
      // Do not signal the callback twice!
      Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
    } else {
      eventListener.callFailed(RealCall.this, e);
      responseCallback.onFailure(RealCall.this, e);
    }
  } finally {
    client.dispatcher().finished(this);
  }
}
```
再execute方法中同样看到了getResponseWithInterceptorChain, 说明, 异步call交给dispatcher调度之后, 兜兜转转, 最终还是回到了call内部的方法来最终执行.


## Dispatcher

### 重要的成员变量

```java
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;

  /** Executes calls. Created lazily. */
  // 这是一个线程池, 并且实现了懒加载
  private @Nullable ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

**为什么异步的call需要readyAsyncCalls和runningAsyncCalls两个队列:**
答案: 因为需要控制maxRequests, 当running队列的size小于maxRequests时, 才会将ready队列的call取出扔到线程池并放入running队列.

### ExcutorService

```java
public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
}
```

这个线程池其实是一个CacheThreadPool. 他的特点是: 
* 底层：返回ThreadPoolExecutor实例，corePoolSize为0；maximumPoolSize为Integer.MAX_VALUE；keepAliveTime为60L；unit为TimeUnit.SECONDS；workQueue为SynchronousQueue(同步队列)
* 通俗：当有新任务到来，则插入到SynchronousQueue中，由于SynchronousQueue是同步队列，因此会在池中寻找可用线程来执行，若有可以线程则执行，若没有可用线程则创建一个线程来执行该任务；若池中线程空闲时间超过指定时间，则该线程会被销毁。
* 适用：执行很多短期异步的小程序或者负载较轻的服务器

检查调用executorService()方法的地方, 是一个promoteAndExecute()方法

### enqueue(AsyncCall call)

```java
void enqueue(AsyncCall call) {
    synchronized (this) {
      readyAsyncCalls.add(call);
    }
    promoteAndExecute();
}
```
上面的逻辑很简单, 将call加入到准备队列中, 然后进行下一步

### promoteAndExecute()

```java

  /**
   * Promotes eligible calls from {@link #readyAsyncCalls} to {@link #runningAsyncCalls} and runs
   * them on the executor service. Must not be called with synchronization because executing calls
   * can call into user code.
   *
   * @return true if the dispatcher is currently running calls.
   */
  // 将符合条件的calls从readyAysncCalls转移到runningAsyncCalls中去, 并且在线程池中run这个call
  // 如果成功执行上面的操作, 返回true
  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      // 将readyAsyncCalls中符合条件的call转移到runningAsyncCalls去
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        // 到达了max限制, break
        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }


    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      // 在线程池中执行这个call
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }
```

promoteAndExecute()方法的解析直接看上面源码的注释. 

结合之前说的, 在enqueue(call), 还有 exectorService() (线程池初始化)时候回调用这个方法, 将准备队列中合适的call扔到线程池中执行. 
想一个问题, 就是当running队列的call都执行完了, 要怎么自动从ready队列中取call来执行呢? 
想到这个问题, 然后再检查promoteAndExecute()方法调用的时机, 发现还有一个finished方法

### finished(Deque<T> calls, T call)

```java
private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
        if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
        idleCallback = this.idleCallback;
    }

    boolean isRunning = promoteAndExecute();

    if (!isRunning && idleCallback != null) {
        idleCallback.run();
    }
}
```
在RealCall和AsyncCall代码中, 能看到在调用了责任链完成一次请求之后，调用了`despatcher.finished`方法从readyAsyncCalls集合里获取下一个call来执行

### executed(RealCall call)
```java
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
}
```
可以看到只是做了一个把 call 加入到 runningSyncCalls 里



### 同步的call

```java
/** Used by {@code Call#execute} to signal it is in-flight. */
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
}
```

同步的call在加入到了runningSyncCalls队列之后, 几乎说明也没干; 在finished方法中有被remove掉. 把同步的call放到Dispatcher中走一遭应该只是单纯的为了把同步call统一管理而已. 







