---
title: Retrofit源码解析
date: 2022-06-12 18:21:16
tags:
---

```java
public interface RetrofitService {
    @GET("query")
    Observable<PostInfo> getPostInfoRx(@Query("type") String type, @Query("postid") String postid);
}

Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("http://www.kuaidi100.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .addCallAdapterFactory(RxJavaCallAdapterFactory.create()) // 支持RxJava
        .build();
        
RetrofitService service = retrofit.create(XxxInterface.class);
Call<PostInfo> call = service.getPostInfo("yuantong", "11111111111");
call.enqueue(new Callback<PostInfo>() {
    @Override
    public void onResponse(Call<PostInfo> call, Response<PostInfo> response) {
        Log.i("http返回：", response.body().toString() + "");
    }

    @Override
    public void onFailure(Call<PostInfo> call, Throwable t) {

    }
});
```

上面是 Retrofit 的主要用法，主要重要的东西有 ConverterFactory , CallAdapterFactory , retrofit.create(XxxInterface.class) 三个东西

## Retrofit.Builder()

builder就是一个常见的建造者模式，没什么好说的。 有一个点需要注意一下
```java
   public Retrofit build() {
      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
          // 默认callback回调是执行在主线程的
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

      List<Converter.Factory> converterFactories = new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);
      converterFactories.addAll(platform.defaultConverterFactories());

      return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
    }
```

```java
final class BuiltInConverters extends Converter.Factory {
    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
    if (type == ResponseBody.class) {
        return Utils.isAnnotationPresent(annotations, Streaming.class)
            ? StreamingResponseBodyConverter.INSTANCE
            : BufferingResponseBodyConverter.INSTANCE;
    }
    if (type == Void.class) {
        return VoidResponseBodyConverter.INSTANCE;
    }
    return null;
    }

    ...
}
```
这里首先插入了一个 BuiltInConverters 内置的转换器。 当返回类型是 ResponseBody 的时候就会使用 BuiltInConverters ，返回 ResponseBody 就和 okhttp 的返回一致了。

## Retrofit的重要部分
1. responseConverter : response的转换器
GsonConverterFactory 将string类型的返回值转换为javabean
2. callAdapter : call适配器。 
DefaultCallAdapterFactory 生产默认的适配器。 默认的行为是将子线程的请求callback切换到主线程。
RxJava2CallAdapterFactory 适配rxjava

## retrofit.create(XxxInterface.class)

```java
  public <T> T create(final Class<T> service) {
    ...
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];

          @Override public @Nullable Object invoke(Object proxy, Method method,
              @Nullable Object[] args) throws Throwable {
            ...
            return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
          }
        });
  }
```

这里使用了动态代理，create方法返回的接口对象是一个动态代理的实例

## loadServiceMethod

```java
  ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

缓存 + 解析Method

## ServiceMethod.parseAnnotations

```java
abstract class ServiceMethod<T> {
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
    ...
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }

  abstract @Nullable T invoke(Object[] args);
}
```

在抽象父类里先调用了 `RequestFactory.parseAnnotations(retrofit, method);` 去解析方法参数的注解。
`HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);` 里解析了返回类型，并匹配 CallAdapter 和 Converter
。。。抽象父类的静态方法区调用了子类的静态方法。。。有些迷惑

## HttpServiceMethod.parseAnnotations

RequestFactory.parseAnnotations 都是一些注解的处理，不说了

```java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
    boolean continuationWantsResponse = false;
    boolean continuationBodyNullable = false;

    Annotation[] annotations = method.getAnnotations();
    Type adapterType;
    if (isKotlinSuspendFunction) {
    // 是协程suspend方法调用
    // kotli的suspend方法转成java之后会在方法参数上生成一个Continuation类型的参数
      Type[] parameterTypes = method.getGenericParameterTypes(); // 获取方法的所有参数类型
      Type responseType = Utils.getParameterLowerBound(0,
          (ParameterizedType) parameterTypes[parameterTypes.length - 1]); // 获取最后一个 Continuation 类型参数的泛型
      if (getRawType(responseType) == Response.class && responseType instanceof ParameterizedType) {
        // Unwrap the actual body type from Response<T>.
        // 获取 Response<T> 中泛型T的类型
        responseType = Utils.getParameterUpperBound(0, (ParameterizedType) responseType);
        continuationWantsResponse = true;
      } else {
      }

    // 这里把suspend方法的返回类型的 AdapterType 当做 Call<T> 来处理了
      adapterType = new Utils.ParameterizedTypeImpl(null, Call.class, responseType);
      annotations = SkipCallbackExecutorImpl.ensurePresent(annotations);
    } else {
    // 如果是非suspend方法，那直接取返回类型就可以了
      adapterType = method.getGenericReturnType();
    }

    // 寻找合适的 CallAdapterFactory 生产 CallAdapter
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    Type responseType = callAdapter.responseType();
    ...

    // 寻找合适的 ConverterFactory 生产 Converter
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    // CallFactory 实质上就是 OkHttpClient
    okhttp3.Call.Factory callFactory = retrofit.callFactory; 

    // 下面是返回 HttpServiceMethod 的子类，这三个子类其实就是 CallAdapted wrap 了一层。 SuspendForResponse 和 SuspendForBody 调用了协程的 await 方法去等待协程的完成（其实就是加了个Continuation的回调）
    if (!isKotlinSuspendFunction) {
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForResponse<>(requestFactory,
          callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
    } else {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForBody<>(requestFactory,
          callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
          continuationBodyNullable);
    }
  }
```

## ServiceMethod.invode

```java
/**
HttpServiceMethod
**/
  @Override final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }

  protected abstract @Nullable ReturnT adapt(Call<ResponseT> call, Object[] args);
```

OkHttpCall 实现了 Retrofit 的 call 到 OkHttpCall 的转换
adapt 最终会调用 CallAdapter 的 adapt

```java
// OkHttpCall#enqueue
 @Override 
 public void enqueue(final Callback<T> callback) {
    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
          // 懒加载 rawCall
          // 传入真实请求参数，构建真正发起请求的 okhttp 的 Call 对象
          call = rawCall = createRawCall();
      }
    }
    ...

    call.enqueue(new okhttp3.Callback() {
      @Override 
      public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        // 调用 Converter 解析返回类型
        response = parseResponse(rawResponse);
        callback.onResponse(OkHttpCall.this, response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        callback.onFailure(OkHttpCall.this, e);
      }
    });
  }
```


可以看一下默认的 DefaultCallAdapterFactory 是怎么实现的
```java
/**
DefaultCallAdapterFactory
**/
  @Override public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    ...
    // 获取 Call<T> 的具体类型
    final Type responseType = Utils.getParameterUpperBound(0, (ParameterizedType) returnType);

    // 是否有 SkipCallbackExecutor 这个注解，一般是没有的
    final Executor executor = Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
        ? null
        : callbackExecutor;

    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return executor == null
            ? call
            : new ExecutorCallbackCall<>(executor, call);
      }
    };
  }
```

```java
  static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }

    @Override public void enqueue(final Callback<T> callback) {
      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          callbackExecutor.execute(new Runnable() {
            ...
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            ...
          });
        }
      });
    }

    @Override public Response<T> execute() throws IOException {
      return delegate.execute();
    }
```

enqueue 里的返回 onResponse 和 onFailure 都是在 callbackExecutor
callbackExecutor 也是在 retrofit 构建时传入的， 默认是 `Android.MainThreadExecutor` 也就是说默认 enqueue 的 callback 默认是执行在主线程的

## 对kotlin协程的支持

动态代理的api接口的方法调用支持suspend协程，具体的方法是使用 suspendCancellableCoroutine 

```java
suspend fun <T : Any> Call<T>.await(): T {
  return suspendCancellableCoroutine { continuation ->
    continuation.invokeOnCancellation {
      cancel()
    }
    enqueue(object : Callback<T> {
      override fun onResponse(call: Call<T>, response: Response<T>) {
        if (response.isSuccessful) {
          val body = response.body()
          if (body == null) {
            ...
            continuation.resumeWithException(e)
          } else {
            continuation.resume(body)
          }
        } else {
          continuation.resumeWithException(HttpException(response))
        }
      }

      override fun onFailure(call: Call<T>, t: Throwable) {
        continuation.resumeWithException(t)
      }
    })
  }
}
```





