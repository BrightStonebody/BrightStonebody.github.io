---
title: kotlin-flow
date: 2021-10-04 14:58:00
tags: Kotlin
---

# 基础的流

## demo
```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

              
fun simple(): Flow<Int> = flow { // 流构建器
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
    simple().collect { value -> println(value) } 
}
```

## flow 是 ”冷流“

- 冷流的定义：  flow 构建器中的代码直到流被调用 collect 的时候才运行。

```kotlin
public fun <T> flow(@BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)

private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}

public interface FlowCollector<in T> {
    public suspend fun emit(value: T)
}
```

flow 的实现如上，只不过是传入了 代码块作为 FlowCollector 的拓展函数。
当 流开始收集 （调用collect），会执行这个 flow 的代码块。 
在 collect 中传入的代码块， 实际上是一个 FlowCollector 匿名对象，当 flow 的代码块调用 emit 的时候，就直接调用了 collect 后的代码块。

用一个例子体现这一点：

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..1) {
        log("Emit start $i")
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
        log("Emit end $i")
    }
}

fun main() = runBlocking<Unit> {
    simple()
        .collect { value ->
            log("Collected start $value")
            delay(300)
            log("Collected end $value")
        }
}
```

最终输出：

```shell
[main] Emit start 1
[main] Collected start 1
[main] Collected end 1
[main] Emit end 1
```

## 流的上下文保存

- 上下文保存： **flow { ... } 构建器中的代码运行在相应流的收集器提供的上下文中**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
           
fun simple(): Flow<Int> = flow {
    log("Started simple flow")
    for (i in 1..3) {
        emit(i)
    }
}  

fun main() = runBlocking<Unit> {
    simple().collect { value -> log("Collected $value") } 
} 
```

```shell
[main @coroutine#1] Started simple flow
[main @coroutine#1] Collected 1
[main @coroutine#1] Collected 2
[main @coroutine#1] Collected 3
```

由于 simple().collect 是在主线程调用的，那么 simple 的流主体也是在主线程调用的。 
**如果在 调用`emit`时切换协程上下文，会直接抛出异常**

- flowOn: 正确的切换 flow 的上下文

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
           
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100) // 假装我们以消耗 CPU 的方式进行计算
        log("Emitting $i")
        emit(i) // 发射下一个值
    }
}.flowOn(Dispatchers.Default) // 在流构建器中改变协程上下文的正确方式

fun main() = runBlocking<Unit> {
    simple().collect { value ->
        log("Collected $value") 
    } 
} 
```

## 解决 背压问题

名词解释：**背压： 假设有一个快的数据生产者和一个慢的数据消费者，背压就是一种 "倒逼 "生产者而使自己不被数据淹没的机制**

### buffer
并发运行这个 流中发射元素的代码以及收集的代码， 而不是顺序运行它们； buffer() 会给切换 flow 的协程，保证 发送端 是不阻塞的 。 可以设置 buffer 容量和策略


```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        simple()
            .buffer() // 缓冲发射项，无需等待
            .collect { value -> 
                delay(300) // 假装我们花费 300 毫秒来处理它
                println(value) 
            } 
    }   
    println("Collected in $time ms")
}
```

### conflate

一种特殊 buffer 策略 新数据会覆盖老数据

### collectLatest

它并不会直接用新数据覆盖老数据，而是每一个都会被处理，只不过如果前一个还没被处理完后一个就来了的话，处理前一个数据的逻辑就会被取消。

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        simple()
            .collectLatest { value -> // 取消并重新发射最后一个值
                println("Collecting $value") 
                delay(300) // 假装我们花费 300 毫秒来处理它
                println("Done $value") 
            } 
    }   
    println("Collected in $time ms")
}
```

```shell
Collecting 1
Collecting 2
Collecting 3
Done 3
Collected in 741 ms
```


# StateFlow 

## demo

```kotlin
    val flow = MutableStateFlow(2)

    private suspend fun produce() {
        for (i in 0..5) {
            val success = flow.tryEmit(i)
            println("tryEmit $i $success")
//            delay(200)
        }
    }

    @JvmStatic
    fun main(args: Array<String>): Unit = runBlocking {
        launch(Dispatchers.IO) {
            delay(1000)
            produce()
        }

        launch(Dispatchers.IO){

            flow.onEach {
                println("receive-1 $it")
                delay(500)
            }.launchIn(this)
        }
    }
```

## StateFlow 是一个 "流"

不同于普通的 flow ， StateFlow 是一个热数据流。 不需要主动调动 collect{...} , 就能 emit 数据

## StateFlow 的 collect 是阻塞的，需要在单独的协程中运行

在demo中调用了 launchIn ，而不是直接 collect 。 看一下源码

```kotlin
// StateFlow 

    private val _state = atomic(initialState)

    override suspend fun collect(collector: FlowCollector<T>) {
        val slot = allocateSlot()
        try {
            if (collector is SubscribedFlowCollector) collector.onSubscription()
            val collectorJob = currentCoroutineContext()[Job]
            var oldState: Any? = null 
            while (true) {
                val newState = _state.value
                collectorJob?.ensureActive()
                if (oldState == null || oldState != newState) {
                    collector.emit(NULL.unbox(newState))
                    oldState = newState
                }
                if (!slot.takePending()) { // try fast-path without suspending first
                    slot.awaitPending() // only suspend for new values when needed
                }
            }
        } finally {
            freeSlot(slot)
        }
    }
```

可以看到，在 collec 中，一个 while 死循环，开始不断读取 _state.value 的值
solt 可以看做是 collector 的一个封装。 这里调用了 slot.awaitPending() 来挂起当前协程，让出cpu时间. 看一下 StateFlowSlot 是如何挂起的 

```kotlin
    suspend fun awaitPending(): Unit = suspendCancellableCoroutine sc@ { cont ->
        // 如果 _state 是 None ，就设置为 cont 
        // cont 的类型 CancellableContinuation ，只用调用 resume 或 fail 等，才会结束，否则挂起当前协程
        if (_state.compareAndSet(NONE, cont)) return@sc
        cont.resume(Unit)
    }
```

在 StateFlow 的 emit 被调用的时候， 会获取到 solt 中的 CancellableContinuation 对象，调用其 resume ，将协程进行唤醒

# ShareFlow

## demo 

```kotlin 
val flow = MutableSharedFlow<Int>(
    replay = 0, 
    extraBufferCapacity = 100, 
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)

private suspend fun produce() {
    for (i in 0..5) {
        val success = flow.tryEmit(i)
        println("tryEmit $i $success")
//            delay(200)
    }
}

@JvmStatic
fun main(args: Array<String>): Unit = runBlocking {
    launch(Dispatchers.IO) {
        delay(1000)
        produce()
    }

    launch(Dispatchers.IO){

        flow.onEach {
            println("receive-1 $it")
            delay(500)
        }.launchIn(this)
    }
}
```

MutableShareFlow 的三个参数：
- 通过 replay ，您可以针对新订阅者重新发送多个之前已发出的值
- extraBufferCapacity: 除了 replay 值以外的缓冲区容量。 extraBufferCapacity + replay 是缓冲区buffer的容量
- 通过 onBufferOverflow，您可以指定相关政策来处理缓冲区中已存满要发送的数据项的情况。默认值为 BufferOverflow.SUSPEND，这会使调用方挂起。其他选项包括 DROP_LATEST 或 DROP_OLDEST

ShareFlow 在收集侧 和 StateFlow 基本一致。 都是 while 死循环，然后将 solt 挂起。 这里不再赘述

<!-- ## BufferOverflow.SUSPEND, tryEmit 和 Emit

当  -->







