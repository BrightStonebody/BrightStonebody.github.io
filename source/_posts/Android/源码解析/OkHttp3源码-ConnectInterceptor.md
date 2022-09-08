---
title: OkHttp3源码-ConnectInterceptor
date: 2019-11-15 23:45:15
tags:
- okhttp3
- 源码解析
categories:
- Android
---

ConnectInterceptor的主要工作是创建一个连接. 由于建立连接涉及到tcp握手之类的操作, 所以开销是很大的, okhttp的一个特色在创建连接时使用到了ConnectionPool, 实现了连接的复用.

## 参考链接:
[okhttp源码分析（四）-ConnectInterceptor过滤器](https://www.jianshu.com/p/4bf4c796db6f)
[OkHttp3源码解析(三)——连接池复用](https://juejin.im/post/5b73abe55188256142142d89)

## intercept

```java
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

ConnectInterceptor中的逻辑很简单, 只有那么一点点代码, 主要的功能都在StreamAllocation这个类里面. StreamAllocation这个类真的是一点都不简单.

## StreamAllocation

### 流, 连接, 请求
HTTP通信执行网络"请求"需要在"连接"上建立一个新的"流". StreamAllocation称之流的桥梁，它负责为一次"请求"寻找"连接"并建立"流"
来看看StreamAllocation源代码上的官方注释:
```java
/**
 * <ul>
 *     <li><strong>Connections:</strong> physical socket connections to remote servers. These are
 *         potentially slow to establish so it is necessary to be able to cancel a connection
 *         currently being connected.
 *     <li><strong>Streams:</strong> logical HTTP request/response pairs that are layered on
 *         connections. Each connection has its own allocation limit, which defines how many
 *         concurrent streams that connection can carry. HTTP/1.x connections can carry 1 stream
 *         at a time, HTTP/2 typically carry multiple.
 *     <li><strong>Calls:</strong> a logical sequence of streams, typically an initial request and
 *         its follow up requests. We prefer to keep all streams of a single call on the same
 *         connection for better behavior and locality.
 * </ul>
 */
```
**翻译:**
* Connection:
    到远端服务器的物理连接. Socket连接的具体工作者
* Stream:
    在连接上建立的http请求和返回的逻辑流. 关于同一时刻能够携带的流的数量, 不同的连接有不同的限制. http/1.x连接能够同时建立一个流, http/2能够同时建立多个.
    在okhttp3的流是HttpCodec表示
* Call: 
    对一系列逻辑流的封装. 有可能是一个单独的请求, 有可能包含多个重定向. 为了更好的性能表现, 我们更希望将一个call中的多个流包含在一个连接中

### newStream, findHealthyConnection

#### newStream

获取合适的连接, 从连接中获取流
```java
  public HttpCodec newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {
    int connectTimeout = client.connectTimeoutMillis();
    int readTimeout = client.readTimeoutMillis();
    int writeTimeout = client.writeTimeoutMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      //获取一个连接
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);
      //实例化HttpCodec,如果是HTTP/2则是Http2Codec否则是Http1Codec
      HttpCodec resultCodec = resultConnection.newCodec(client, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```

#### findHealthyConnecton

不断循环, 直到获取一个healthy?的连接
健康的连接, 大概意思是socket能正常使用的意思吧

```java
  /**
   * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
   * until a healthy connection is found.
   */
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        // 如果这个连接不健康, 
        // 禁用这条连接, 重复寻找可用的连接
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
```

### findConnection-重点

```java
 /**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
 /**
   * 返回一个连接. 优先使用已存在的连接, 其次是从连接池中获取, 最后新建一个连接
   */
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      ... // 省略代码

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new streams.
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {
        //如果当前connection不为空可以直接使用
        // We had an already-allocated connection and it's good.
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }

      //当前这个connection不能使用，尝试从连接池里面获取一个请求
      if (result == null) {
        // Attempt to get a connection from the pool.
        // Internal是一个抽象类，instance是在OkHttpClient中实现的，get方法实现的时候从pool的get方法
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    ... // 省略代码

    if (result != null) {
      // 找到一条可复用的连接
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // 到达这里表示没有找到
    // 切换路由再在连接池里面找下，如果有则返回

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    // 检查是否有其他路由
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        // 有其他路由, 遍历RooteSelector
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      if (!foundPooledConnection) {
        //没找到则创建一条
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        //往连接中增加流
        acquire(result, false);
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    //如果第二次找到了可以复用的，则返回
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    // 建立连接,开始握手
    result.connect(
        connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled, call, eventListener);
    // 将这条路由从错误缓存中清除
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // Pool the connection.
      //将这个请求加入连接池
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      // 如果是多路复用，则合并
      if (result.isMultiplexed()) {
        // 返回的是一个重复的socket
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    // 关闭重复的socket
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```

## ConnectionPool

### 主要成员变量

```java
/**
* Background threads are used to cleanup expired connections. There will be at most a single
* thread running per connection pool. The thread pool executor permits the pool itself to be
* garbage collected.
*/
/**
被用来清理超时连接的后台线程. 在每个连接池中最多只会有一个这样的线程(虽然它是个线程池). 
*/
private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
    Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
    new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));

/** The maximum number of idle connections for each address. */
// 允许的每个host地址可以维持的最大空闲连接数量, 默认为5
private final int maxIdleConnections;
// 允许的线程空闲的最大时间, 默认为5分钟
private final long keepAliveDurationNs;
// 清理的task
private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
        while (true) {
            long waitNanos = cleanup(System.nanoTime());
            if (waitNanos == -1) return;
            if (waitNanos > 0) {
                long waitMillis = waitNanos / 1000000L;
                waitNanos -= (waitMillis * 1000000L);
                synchronized (ConnectionPool.this) {
                    try {
                        ConnectionPool.this.wait(waitMillis, (int) waitNanos);
                    } catch (InterruptedException ignored) {
                    }
                }
            }
        }
    }
};
// 连接池中的连接集合
private final Deque<RealConnection> connections = new ArrayDeque<>();
// 用来记录连接失败的路线名单 ?? 暂时没懂啥意思...
final RouteDatabase routeDatabase = new RouteDatabase();
// 标记清理线程是否在运行
boolean cleanupRunning;
```

**ConnectionPool创建的位置:**
ConnectionPool是在OkhttpClient在build是创建的, 默认就会创建连接池, 使用默认的参数. 当然, 也可以自己设置连接池, 修改默认参数(maxIdleConnections, keepAliveDurationNs)


### cleanUpRunnable

会被放入线程池的清理任务

```java
private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
        while (true) {
            // cleanUp方法做connection的清理工作, 会返回一个long值, 表示清理线程之后将要挂起的时间
            long waitNanos = cleanup(System.nanoTime());
            if (waitNanos == -1) return;
            if (waitNanos > 0) {
                long waitMillis = waitNanos / 1000000L;
                waitNanos -= (waitMillis * 1000000L);
                synchronized (ConnectionPool.this) {
                    try {
                        // 挂起清理线程
                        ConnectionPool.this.wait(waitMillis, (int) waitNanos);
                    } catch (InterruptedException ignored) {
                    }
                }
            }
        }
    }
};
```

### cleanUp

找到需要清理的connection, 如果没有找到, 那么确定下次清理的时间.

```java
/**
* Performs maintenance on this pool, evicting the connection that has been idle the longest if
* either it has exceeded the keep alive limit or the idle connections limit.
*
* <p>Returns the duration in nanos to sleep until the next scheduled call to this method. Returns
* -1 if no further cleanups are required.
*/

long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    // 找到需要清理的connection, 如果没有找到, 那么确定下次清理的时间. 也就是说, 只会清理一个合适的connection
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // If the connection is in use, keep searching.
        // pruneAndGetAllocationCount方法判断当前connection是否正在使用中
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }

        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        // 记录空闲最长的那个connection, 并且记录空闲的最长时间
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        
        // 如果 最长空闲的connection空闲时间超过限制, 或者 如果空闲的connection数量超过限制
        // 从connections集合中remove掉该connection
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        // 最长空闲的connection没有超出时间限制, 返回下次开启清理的时间
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        // 没有空闲的connection, 等待keepAliveDurationNs时间之后再次清理
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        // 根本没有connection, 返回-1, 直接终止清理任务
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
}
```
当连接空闲时间超出限制 或者 空闲连接的数量超出限制时，会remove掉最长空闲时间的connection


### pruneAndGetAllocationCount

判断该连接是否是空闲的

```java
private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<StreamAllocation>> references = connection.allocations;
    for (int i = 0; i < references.size(); ) {
        Reference<StreamAllocation> reference = references.get(i);
    
        //如果存在引用，就说明是活跃连接，就继续看下一个StreamAllocation
        if (reference.get() != null) {
            i++;
            continue;
        }

        ... // 省略代码
        
        //如果没有引用，就移除
        references.remove(i);
        connection.noNewStreams = true;

        //如果列表为空，就说明此连接上没有StreamAllocation引用了，就返回0，表示是空闲的连接
        if (references.isEmpty()) {
            connection.idleAtNanos = now - keepAliveDurationNs;
            return 0;
        }
    }
    //遍历结束后，返回引用的数量，说明当前连接是活跃连接
    return references.size();
}
```

**判断连接是否空闲过程:**
RealConnection中有该连接所有的StreamAllocation的弱引用集合, 去除集合中为null的元素(弱引用...), 若集合为空, 则该连接时空闲的. 

### get和put

**get:**
```java
RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      //判断这个连接是否符合address和route, 判断过程很麻烦
      if (connection.isEligible(address, route)) {
        // 将streamAllocation和这个connection绑定
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    return null;
}
```

**put:**
```java
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      // 当清理任务没有工作的时候, 将任务放入线程池中运行
      // 因为当connections集合为空时, 清理任务会终止
      // ? 既然只有一个清理线程存在, 那使用线程池的意义是什么 ???
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```

插入和获取connection的key是adress， 它是 host ，cdn ，port 等的抽象

