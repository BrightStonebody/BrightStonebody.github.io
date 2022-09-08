---
title: OkHttp3源码-CacheInterceptor
date: 2019-11-14 20:17:52
tags:
- okhttp3
- 源码解析
categories:
- Android
---

# OkHttp3源码-CacheInterceptor

## interceptor的详细过程

看过伪代码, 现在来看详细过程, 就好多了

```java
@Override 
public Response intercept(Chain chain) throws IOException {
    //默认cache为null, client可以配置cache,不为空尝试获取缓存中的response
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    //根据response, time, request 创建一个缓存策略，用于判断怎样使用缓存
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    // 如果不需要使用网络请求 networkRequest 会为空
    Request networkRequest = strategy.networkRequest;
    // 如果不能使用缓存， cacheResponse 会为空
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    //如果缓存策略中禁止使用网络，并且缓存又为空，则构建一个Resposne直接返回，注意返回码=504
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // 不使用网络，但是有缓存，直接返回缓存
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      //直接走后续拦截器
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // 当缓存响应和网络响应同时存在的时候，选择用哪个
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        // 如果返回码是304，客户端有缓存，并发出了一个条件性的请求
        // (一般是提供If-Modified-Since头表示客户只想比指定日期更新的文档).
        // 服务器告诉客户，原来缓冲的文档还可以继续使用。则使用缓存的响应
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }
    //使用网络响应
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    //所以默认创建的OkHttpClient是没有缓存的
    if (cache != null) {
      // 缓存response
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // 缓存Resposne里的request信息
        CacheRequest cacheRequest = cache.put(response);
        // 缓存Response里的body。 缓存到disk
        return cacheWritingResponse(cacheRequest, response);
      }
      // 只能okhttp3只能缓存GET请求....不然从cache中移除request
      // 很奇怪, 为什么要在这里加一个判断
      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
}
```

## Cache

**OkHttpClient创建时添加cache:**
```java
OkHttpClient.Builder builder = new OkHttpClient.Builder()
    .connectTimeout(15, TimeUnit.SECONDS)
    .writeTimeout(20, TimeUnit.SECONDS)
    .readTimeout(20, TimeUnit.SECONDS)
    .cache(new Cache(context.getExternalCacheDir(), 10*1024*1024));
```

**InternalCache和Cache:**

InternalCache时在框架内部进行调用的接口, Cache类中有InternalCache的成员变量

**实现原理:**

封装了对DiskLruCach的操作

**注意点:**
**Cache类只能缓存get请求.** 在put方法的注释中提到, 从技术上来说, 是可以缓存post方法的response的, 但是这样的代价太高, 效率很低.
首先，okhttp缓存的key是url。 而post请求包含了body。
而且对于标准的RESTful请求，GET就是用来获取数据，最适合使用缓存，而对于数据的其他操作缓存意义不大或者根本不需要缓存。


## CacheStrategy

**构造方法**
```java
CacheStrategy(Request networkRequest, Response cacheResponse)
```
生成的CacheStrategy中有2个变量，networkRequest和cacheResponse，如果networkRequest为null，则表示不进行网络请求；而如果cacheResponse为null，则表示没有有效的缓存。

**CacheStrategy.Factory**
```java
public Factory(long nowMillis, Request request, Response cacheResponse) 
```

这是CacheStrategy的一个静态内部类, 代码就不贴了. 主要作用是根据参数和创建CacheStrategy

request参数中附带有用户对缓存策略的配置: ( .cacheControl)

类如:
```java
Request request = new Request.Builder()
       .cacheControl(new CacheControl.Builder().noCache().build())
       .url("http://publicobject.com/helloworld.txt")
       .build();
```








