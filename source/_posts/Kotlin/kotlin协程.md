---
title: kotlin协程
date: 2020-07-15 11:30:02
tags: 
- Kotlin
categories:
- Kotlin
---


## Async

首先看一个例子，先后调用两个挂起函数

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假设我们在这里做了些有用的事
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 假设我们在这里也做了一些有用的事
    return 29
}

/** 输出

The answer is 42
Completed in 2017 ms

**/
```

可以发现，两个挂起函数是同步执行的，有先后顺序

使用async

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假设我们在这里做了些有用的事
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 假设我们在这里也做了些有用的事
    return 29
}

/** 输出
The answer is 42
Completed in 1024 ms
**/
```

在结果耗时上，两个挂起方法达到了异步的效果。这得益于`async`关键字
在概念上，`async` 就类似于 launch。它启动了一个单独的协程，这是一个轻量级的线程并与其它所有的协程一起并发的工作。不同之处在于 launch 返回一个 Job 并且不附带任何结果值，而 async 返回一个 Deferred —— 一个轻量级的非阻塞 future， 这代表了一个将会在稍后提供结果的 promise。你可以使用 .await() 在一个延期的值上得到它的最终结果， 但是 Deferred 也是一个 Job，所以如果需要的话，你可以取消它。

## 调度器与线程

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    launch { // 运行在父协程的上下文中，即 runBlocking 主协程
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // 不受限的——将工作在主线程中
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // 将会获取默认调度器
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // 将使它获得一个新的线程
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }    
}

/** 输出 注意先后顺序

Unconfined            : I'm working in thread main @coroutine#3
Default               : I'm working in thread DefaultDispatcher-worker-1 @coroutine#4
main runBlocking      : I'm working in thread main @coroutine#2
newSingleThreadContext: I'm working in thread MyOwnThread @coroutine#5

**/
```

* `launch { …… }` :
    * 当调用 launch { …… } 时不传参数，它从启动了它的 CoroutineScope 中承袭了上下文（以及调度器）。在这个案例中，它从 main 线程中的 runBlocking 主协程承袭了上下文。
* `launch(Dispatchers.Default) { …… }` :
    * 当协程在 GlobalScope 中启动时，使用的是由` Dispatchers.Default `代表的默认调度器。 默认调度器使用共享的后台线程池。 所以` launch(Dispatchers.Default) { …… } `与` GlobalScope.launch { …… } `使用相同的调度器。 
    `Dispatchers.Default `适合CPU密集型的任务，比如解析JSON文件，排序一个较大的list
* `launch(Dispatchers.IO) { …… }` :
    * 针对磁盘和网络IO进行了优化，适合IO密集型的任务，比如：读写文件，操作数据库以及网络请求 
* `launch(newSingleThreadContext("...")) { ... }` :
    * `newSingleThreadContext` 为协程的运行启动了一个线程。 一个专用的线程是一种非常昂贵的资源。 在真实的应用程序中两者都必须被释放，当不再需要的时候，使用 `close` 函数，或存储在一个顶层变量中使它在整个应用程序中被重用。???
* `launch(Dispatchers.Unconfined) { ... }` :
    * 完全没搞懂这玩意儿..


## 在Android中使用协程作用域

```kotlin
import kotlinx.coroutines.*

class Activity {
    private val mainScope = MainScope()
    
    fun destroy() {
        mainScope.cancel()
    }

    fun doSomething() {
        // 在示例中启动了 10 个协程，且每个都工作了不同的时长
        repeat(10) { i ->
            mainScope.launch {
                delay((i + 1) * 200L) // 延迟 200 毫秒、400 毫秒、600 毫秒等等不同的时间
                println("Coroutine $i is done")
            }
        }
    }
} // Activity 类结束

fun main() = runBlocking<Unit> {
    val activity = Activity()
    activity.doSomething() // 运行测试函数
    println("Launched coroutines")
    delay(500L) // 延迟半秒钟
    println("Destroying activity!")
    activity.destroy() // 取消所有的协程
    delay(1000) // 为了在视觉上确认它们没有工作    
}
```

通过创建一个 CoroutineScope 实例来管理协程的生命周期，并使它与 activity 的生命周期相关联。CoroutineScope 可以通过 CoroutineScope() 创建或者通过MainScope() 工厂函数。前者创建了一个通用作用域，而后者为使用 `Dispatchers.Main` 作为默认调度器的 UI 应用程序 创建作用域.

在上面的程序中, 当Actiivty被destroy()时, 会 cancel() 协程作用域, 从而终止所有的协程

## 异步流

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun foo(): Flow<Int> = flow { // 流构建器
    for (i in 1..3) {
        delay(100) // 假装我们在这里做了一些有用的事情
        emit(i) // 发送下一个值
    }
}

fun main() = runBlocking<Unit> {
    // 启动并发的协程以验证主线程并未阻塞
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // 收集这个流
    foo().collect { value -> println(value) } 
}

/** 输出

I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
**/
```

上方的程序: 
* 名为 flow 的 Flow 类型构建器函数。
* flow { ... } 构建块中的代码可以挂起。
* 函数 foo() 不再标有 suspend 修饰符。
* 流使用 emit 函数 发射 值。
* 流使用 collect 函数 收集 值。

kotlin支持对flow进行各种操作, 比如: flat, reduce, zip等等, 类似RxJava的各种运算

## suspendCancellableCoroutine

```kotlin
private suspend fun loadPageSuspend(productCategoryType: Int, page: Int): ProductListLoadResultBean =
            suspendCancellableCoroutine { cnt ->
                ProductManagementAPI.requestProductList(
                        category = productCategoryType,
                        page = page,
                        keyword = getKeyword(),
                        orderBy = getOrderType(),
                        desc =  isOrderDesc(),
                        listener = object : INetRequestListener<ProductListLoadResultBean> {
                            override fun onSuccess(result: DataHull<ProductListLoadResultBean>?) {
                                result?.let {
                                    cnt.resumeWith(Result.success(it.data))

                                } ?: kotlin.run {
                                    cnt.resumeWithException(Exception())
                                }
                            }

                            override fun onError(error: DataHull<ProductListLoadResultBean>?, isNetError: Boolean) {
                                cnt.resumeWithException(Exception(error?.stateBean?.message))
                            }

                        }
                )
            }
```
