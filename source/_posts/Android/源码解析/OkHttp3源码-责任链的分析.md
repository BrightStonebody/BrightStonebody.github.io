---
title: OkHttp3源码-责任链的分析
date: 2019-11-14 15:52:46
tags:
- okhttp3
- 源码分析
categories:
- Android
---

# okhttp3源码-责任链的分析

okhttp3中一个请求封装为一个call, 无论是一个同步的Realcall还是一个异步的AsyncCall, 最终都是辗转调用自己的getResponseWithInterceptorChain()方法, 开始请求的操作流程

## getResponseWithInterceptorChain()

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    Response response = chain.proceed(originalRequest);
    if (retryAndFollowUpInterceptor.isCanceled()) {
      closeQuietly(response);
      throw new IOException("Canceled");
    }
    return response;
}

```

### 每个拦截器的简单说明:
* 1. 用户拦截器：通过Builder的addInterceptor方法添加的拦截器。
* 2. RetryAndFollowUpInterceptor：负责失败自动重连和必要的重定向。
* 3. BridgeInterceptor：负责将用户的Request转换成一个实际的网络请求Request，再调用下一个拦截器获取Response，然后将Response转换成用户的Response。
* 4. CacheInterceptor：负责控制缓存，缓存的逻辑就在这里面。如果发现有缓存的话, 就直接返回response了, 不会进行后面的流程
* 5. ConnectInterceptor：负责进行连接主机，在这里会完成socket连接，并将连接返回。
* 6. CallServerInterceptor：和服务器通信，完成Http请求。

Chain是Interceptor声明在Interceptor内部的一个接口. RealInterceptorChian实现了Chain接口, 来看看这个类

### RealInterceptorChian

```java
@Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
}

public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
    RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    ... // 省略一些异常处理的代码

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
        throw new IllegalStateException("network interceptor " + interceptor
            + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
        throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
        throw new IllegalStateException(
            "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
}
```

### interceptor.intercept(chain)

**伪代码**
```java
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();

    ... // 处理request

    Response response;
    try {
        // 继续责任链的下一个结点
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
    } catch (Exception e) {
        ... //
    }

    ... // 处理response
}
```

梳理一下这个责任链模式的流程: (' -> '表示调用关系)
-> getResponseWithInterceptorChain()
-> chain.process(request)
-> interceptor.intercept(nextChain)
-> chain.process(request)
-> interceptor.intercept(nextChain)
-> ... // 循环直至责任链最后一个结点




