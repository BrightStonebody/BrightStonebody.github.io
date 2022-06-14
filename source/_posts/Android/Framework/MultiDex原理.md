---
title: MultiDex原理
date: 2021-12-25 18:20:52
tags:
---

apk中的一个dex文件的方法索引是一个ushort类型，最大值为655535. 所以一个dex文件的最大方法数是65535. 
为了规避安卓项目编译65536最大方法数的限制，需要做分dex

## 1. Dalvik 和 ART 虚拟机的区别

Android5.0以上系统使用 ART 虚拟机，5.0以下使用 Dalvik 虚拟机

Dalvik 编译策略是 JIT 技术。JIT意思是Just In Time Compiler，就是即时编译技术。 
JIT是运行时编译，是动态编译，可以对执行次数频繁的dex代码进行编译和优化，减少以后使用时的翻译时间可以加快Dalvik运行速度。每次运行都需要重新编译。

AOT是指"Ahead Of Time"，与"Just In Time"不同，从字面来看是说提前编译。
AOT 是静态编译，应用在安装的时候会启动 dex2oat 过程把 dex预编译，一次性将所有dex代码转换为机器码，每次运行程序的时候不用重新编译。 


Android 7.0上，JIT 编译器被再次使用，采用AOT/JIT 混合编译的策略，特点是：

应用在安装的时候dex不会再被编译
App运行时,dex文件先通过解析器被直接执行，热点函数会被识别并被JIT编译后存储在 jit code cache 中并生成profile文件以记录热点函数的信息。
手机进入 IDLE（空闲） 或者 Charging（充电） 状态的时候，系统会扫描 App 目录下的 profile 文件并执行 AOT 过程进行编译

## 2. 不同虚拟机在 MultiDex 的区别。 

* Andorid 5.0 之后，ART天然支持从apk文件中加载多个.dex文件，在应用安装期间，它会执行一个预编译操作。将所有的dex编译成一个单一的.oat文件，在应用运行是去加载这个.oat文件，而不是一个一个的加载.dex文件。
  所以我们只需要 在 gradle 开启 multiDexEnabled true 即可
* Dalvik 使用的 JIT ，JIT每次运行都需要重新编译。 所以我们需要在应用启动时手动去一个一个加载 dex 文件

## 3. 在 Dalvik 加载多dex

简单来说 hook PathClassLoader 。
1. 获取到所有的dex文件
2. 通过反射拿到DexPathList的Element数组，把dex添加到 Element数组中

通过这样的方式也可以实现热热修复，将修改后的新class打包成一个dex， 然后通过反射将 新dex插入到 PathClassLoader 中的 element数组最前面。从而实现对旧类的覆盖
