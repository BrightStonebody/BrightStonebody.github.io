---
title: gradle相关基础
date: 2020-06-18 19:54:57
tags: 
- Gradle
categories:
- Android
---

## Gradle中的对象
Gradle主要有三种对象
这三种对象和三种不同的脚本文件对应，在gradle执行的时候，会将脚本转换成对应的对象：

* Gradle对象：当我们执行gradle xxx或者什么的时候，gradle会从默认的配置脚本中构造出一个Gradle对象。在整个执行过程中，只有这么一个对象。Gradle对象的数据类型就是Gradle。我们一般很少去定制这个默认的配置脚本。
* Project对象：Gradle对每一个build.gradle会实例化成一个Project类，并且能够通过project变量使其隐式可用。
* Settings对象：显然，每一个settings.gradle都会转换成一个Settings对象。

构建的生命周期，首先根据settings.gradle文件构建出一个Seetings对象，然后根据Seetings中的配置，创建Project对象，去找各个project下的build.gradle文件，根据文件内容来对project对象进行配置。

## setting.gradle

```groovy
include ':app', ':progect_1', ':progect_2'
```
用于指示 Gradle 在构建应用时应将哪些模块包括在内

## gradle.properties

里面可以定义一些常量供build.gradle使用，如版本号等.
然后，我们就可以在build文件中进行引用了。引用方式，直接通过变量名就可以。

```groovy
COMPILE_SDK_VERSION = 23
BUILD_TOOLS_VERSION = 23.0.1
VERSION_CODE = 1
```

## build.gradle
build文件有两种，一个是针对当前的Module，一个是针对项目中所有的module
在顶层的build文件中，我们可以来添加一些子module所共有的一些配置

下面是一些常用的build.gradle的配置说明
```groovy
// 应用android官方插件，module中的build.gradle很多配置都是插件提供的支持
// 'com.android.application' 代表app主工程，
// 如果是子工程应该使用 'com.android.library'
apply plugin: 'com.android.application'

// 仓库， 
buildscript{
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath '' // 声明插件名称和版本号
    }
}
// 外部依赖，添加的依赖会在这些配置的仓库中去寻找
dependencies {
    // "group:name:version"
    implementation 'androidx.appcompat:appcompat:1.1.0'
}

// 所有子工程的仓库地址
allprojects {
    repositories {
        maven { url uri('/Users/bytedance/code/test/repo') }
        google()
        mavenCentral()
    }
}

```

**为什么`buildscript`和`allpojectes`都有`repositories`**
* buildScript块的repositories主要是为了Gradle脚本自身的执行，获取脚本依赖插件。
* allprojects块的repositories用于多项目构建，为所有项目提供共同所需依赖包。比如用到的Glide/OkHttp等
* 子项目可以配置自己的repositories以获取自己独需的依赖包。

## Task

包括任务动作和任务依赖。任务动作定义了一个最小的工作单元。可以定义依赖于其它任务、动作序列和执行条件。

* depandsOn: 依赖于其它任务
* doFirst, doLast(<<): Task是一个动作列表，doFirst表示在动作列表最前面添加一个动作，doLast在动作列表最后面添加一个动作

apply的插件自带和很多Task，在Gradle页面的 `<项目名>/Tasks/build` 目录里面可以看到。
我们也可以自己编写任务，自己的Task在Gradle页面的 `<项目名>/Tasks/other/` 目录里可以查找到


## Gradle的工作流程

![gradle的工作流程](/images/gradle的构建过程.jpg)

* Initialization: 初始化，在多模块的项目中，就是执行settings.gradle
* Configuration: Configration阶段的目标是解析每个project中的build.gradle。比如multi-project build例子中，解析每个子目录中的build.gradle。  Configuration阶段完了后，整个build的project以及内部的Task关系就确定了。一个Project包含很多Task，每个Task之间有依赖关系。Configuration会建立一个有向图来描述Task之间的依赖关系。
* Execution: 执行Task。 只有doFirst和doLast中的内容属于执行阶段

简言之，Gradle有一个初始化流程，这个时候settings.gradle会执行。
在配置阶段，每个Project都会被解析，其内部的任务也会被添加到一个有向图里，用于解决执行过程中的依赖关系。然后才是执行阶段。你在gradle xxx中指定什么任务，gradle就会将这个xxx任务链上的所有任务全部按依赖顺序执行一遍！

在每一步的步骤中间可以添加hook



```java

task a {
    println 'this is a'
    doFirst {
        println 'this is a do first'
    }
    doLast {
        println "this is a do last"
    }
}

task testBoth {
    // 配置阶段
    // 依赖 a task 先执行
    dependsOn("a")
    println 'this is b'
    doFirst {
        // 执行阶段
        println 'this is b first'
    }
    doLast {
        // 执行阶段
        println 'this is b last'
    }
}
/**
输出：

> Configure project :
this is a
this is b

> Task :a
this is a do first
this is a do last

> Task :testBoth
this is b first
this is b last
*/
```



## 解决依赖版本冲突

大部分情况下，不需要我们去解决版本冲突。当出现版本冲突时，Gradle会帮我们自动依赖最高版本的包。

以下是我们自己解决版本冲突的一般步骤

### 查看依赖报告
运行Gradle， `<项目名称>/app/Tasks/dependencies/` 查看依赖报告，输出如下

`xxxx -> xxxx` 表示依赖包自动提升到了最高版本

```
+--- androidx.lifecycle:lifecycle-extensions:2.1.0
|    +--- androidx.lifecycle:lifecycle-runtime:2.1.0
|    |    +--- androidx.lifecycle:lifecycle-common:2.1.0 -> 2.3.0-alpha01
|    |    |    \--- androidx.annotation:annotation:1.1.0
|    |    +--- androidx.arch.core:core-common:2.1.0
|    |    |    \--- androidx.annotation:annotation:1.1.0
|    |    \--- androidx.annotation:annotation:1.1.0
|    +--- androidx.arch.core:core-common:2.1.0 (*)
|    +--- androidx.arch.core:core-runtime:2.1.0

```

### 排除传递性冲突

```groovy
compile ('cn.qqtheme.framework:WheelPicker:1.5.1'){
    exclude group:'com.android.support', module:"appcompat"
}
```

### 强制一个版本
```groovy
configurations.all{
    resolutionStrategy{
        force 'androidx.appcompat:appcompat:1.1.0'
    }
}
```