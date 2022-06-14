---
title: 自定义注解处理器并发布为gradle组件
date: 2022-03-13 15:27:24
tags:
- Gradle
categories: 
- Android
---

自定义一个基于注解的路由框架

## 定义注解类

1. 新建一个模块 router_annotations 
2. 在 router_annotations 目录中创建 build.gradle 

```groovy
apply plugin: "kotlin" // 使用 kotlin

targetCompatibility = JavaVersion.VERSION_1_8
sourceCompatibility = JavaVersion.VERSION_1_8

// 应用 发布脚本，后面会写
apply from : rootProject.file("maven-publish.gradle")
```

3. 定义注解类

```kotlin
package com.example.test

// 设置使用注解的对象是 类
@Target(AnnotationTarget.CLASS)
// 设置注解只在编译期存在
@Retention(AnnotationRetention.BINARY)
annotation class Destination(
    val url: String
)

```

## 定义注解处理器模块

1. 新建一个模块 router_compile
2. 在 router_compile 目录中创建 build.gradle

```groovy

apply plugin: "kotlin" // 使用kotlin插件
apply plugin: "kotlin-kapt" // 使用kotlin注解


// 设置源码兼容性
targetCompatibility = JavaVersion.VERSION_1_8
sourceCompatibility = JavaVersion.VERSION_1_8

dependencies {
    implementation project(":router_annotations")
    implementation "com.google.auto.service:auto-service:1.0-rc6"
    kapt "com.google.auto.service:auto-service:1.0-rc6"
}

// 应用 发布脚本。后面会写
apply from : rootProject.file("maven-publish.gradle")
```

在注解处理器模块，如果使用kotlin写注解处理器代码，需要引用 `kotlin-kapt` 这个插件
`implementation "com.google.auto.service:auto-service:1.0-rc6"` 使用google的一个注解框架。 
同时还要加上 `kapt "com.google.auto.service:auto-service:1.0-rc6"` 。 这是kotlin的写法，如果是java，需要换成
`annotationProcessor 'com.google.auto.service:auto-service:1.0-rc6'`

3. 编写注解处理器
```kotlin
@AutoService(Processor::class)
class DestinationProcessor : AbstractProcessor() {

    private val TAG = "DestinationProcessor"

    override fun process(
        set: MutableSet<out TypeElement>,
        roundEnvironment: RoundEnvironment
    ): Boolean {
        if (roundEnvironment.processingOver()) {
            return false
        }
        println("$TAG >>>> process start")

        println(set)

        val destinationClasses = roundEnvironment.getElementsAnnotatedWith(Destination::class.java)

        println(TAG + " " + destinationClasses.size)
        if (destinationClasses.size < 1) {
            return false
        }

        for (element in destinationClasses) {
            val typeElement = element as TypeElement
            val destination = typeElement.getAnnotation(Destination::class.java)
                ?: continue
            val url = destination.url
            val className = typeElement.qualifiedName
            println("$TAG $url $className")
        }

        println("$TAG >>>> process finish")

        return true
    }

    override fun getSupportedAnnotationTypes(): MutableSet<String> {
        return mutableSetOf(
            Destination::class.java.canonicalName
        )
    }
}
```

`@AutoService(Processor::class)` 是固定写法

`AbstractProcessor` 继承这个抽象类。 
重写`getSupportedAnnotationTypes`方法，返回需要处理的注解。
重写`process`方法，处理注解信息。 返回值为true表示已处理这个注解，相同的注解不会再被其他注解处理器处理。
`roundEnvironment.processingOver()`表示之前已经有处理过一轮，此时`roundEnvironment.elements.size==0` 不是很明白gradle为什么会执行多轮。。。

## 在本地使用注解
```groovy
implementation project(":router_annotations")
kapt project(":router_compile")
```
 
## 将注解处理器打包成组件并发布

1. 在最外层的 build.properties 中写通用参数
```groovy
POM_URL=../repo  // 发布的仓库地址，这里是本地
GROUP_ID=com.example.test // group
VERSION_NAME=1.0.0 // 版本
```

2. 在 router_annotation 和 router_compile 中写 group_id
```groovy
// 在 router_annotation build.properties
POM_ARTIFACT_ID=router-test-annotation

// 在 router_compile build.properties
POM_ARTIFACT_ID=router-test-processor
```

3. 编写发布脚本 maven-publish.gradle

```groovy
apply plugin: 'maven' // 引入maven插件

// 从根目录的 gradle.properties 中获取通用参数 start
def rootProperties = new Properties()
rootProperties.load(new FileInputStream(project.rootProject.file("gradle.properties")))

def versionName = rootProperties.getProperty("VERSION_NAME")
def pomUrl = rootProperties.getProperty("POM_URL") // ../repo
def groupId = rootProperties.getProperty("GROUP_ID")
// 获取通用参数 end

// 从当前项目的 gradle.properties 中获取参数
def projectProperties = new Properties()
projectProperties.load(new FileInputStream(project.file("gradle.properties")))
def pomArtifactId = projectProperties.getProperty("POM_ARTIFACT_ID")

println("maven-publish $versionName $pomUrl $groupId $pomArtifactId")

// 编写maven的发布任务
uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: uri(pomUrl)) {
                pom.groupId = groupId
                pom.artifactId = pomArtifactId
                pom.version = versionName
            }

            pom.whenConfigured { pom ->
                pom.dependencies.forEach { dep ->
                    if (dep.getVersion() == "unspecified") {
                        dep.setGroupId(groupId)
                        dep.setVersion(versionName)
                    }

                }

            }
        }
    }
}
```

注意有一个 `pom.whenConfigured { pom -> ...` 的代码，这是因为 router_compile 应用到了 router_annotations 模块。如果不做处理，发布之后， router_comiple 模块是找不到 router_annotations 的正确位置和版本的。所以要强制制定

## 执行发布任务
```bash
./gradlew :router_compile:uploadArchives
./gradlew :router_annotations:uploadArchives
```
我们设置的 `POM_URL=../repo` 所以最终会在项目的repo目录里生成aar包。 

## 使用发布后的组件

1. 首先，moven本地仓库地址加入到工程里
```groovy
buildscript {
    repositories {
        maven { url uri('/Users/bytedance/code/test/repo') }
        google()
        mavenCentral()
    }
}

allprojects {
    repositories {
        maven { url uri('/Users/bytedance/code/test/repo') }
        google()
        mavenCentral()
    }
}
```
buildscript.repositories 和 allprojects.repositories 都要加，这样才会应用到所有子工程

2. 在app模块里引用组件
```groovy
    implementation "com.example.test:router-test-annotation:1.0.0"
    kapt "com.example.test:router-test-processor:1.0.0"
```

3. gradle版本的区别

在 gradle 7.+ 版本有很多东西不兼容，上述实现只有在 gradle 6.+ 的版本有效。 gradle 的版本升级和项目创建还是要谨慎一些。 
