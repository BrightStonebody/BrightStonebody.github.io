---
title: 自定义gradle插件
date: 2022-03-19 17:52:18
tags:
- Gradle
categories: 
- Android
---

这里只记录本地插件的写法

## 创建 buildSrc 目录
1. 在创建的 buildScr 目录中创建 build.gradle 文件

```groovy
apply plugin: "groovy"

repositories {
    jcenter()
    google()
    mavenCentral()

}

dependencies {
    implementation gradleApi()
    implementation localGroovy()
}

// 设置源码兼容性
targetCompatibility = JavaVersion.VERSION_1_8
sourceCompatibility = JavaVersion.VERSION_1_8

```

上面是固定写法，这里只使用了 groovy 插件，所以我们的插件代码只支持用 groovy 编写

2. 在 buildSrc 目录中，创建对应的 groovy 目录
文件路径是
src/main/groovy/...(包名)/

3. 创建 自定义的插件 RouterPlugin2
   
创建文件名 RouterPlugin2.Plugin
创建插件类，并实现 Plugin 接口

```groovy

class RouterPlugin2 implements Plugin<Project> {

    String TAG = "RouterPlugin2"

    @Override
    void apply(Project project) {
        ...
    }
}
```
   
4. 创建 res 文件暴露自己的 groovy
创建文件路径：
src/main/resources/META-INF.gradle-plugins/RouterPlugin.properties

内容如下
```groovy
implementation-class=com.example.test.RouterPlugin2
```

创建的文件名 META-INF.gradle-plugins 是固定目录名，xxxx.properties 中的xxxx就是对外暴露的插件的名字。 properties文件的内容表示插件的实现类

## 外部使用插件

1. 导入插件

在业务的module的 build.gradle 文件中

```groovy
apply plugin: "RouterPlugin2"
```

2. 向插件写入参数

在 buildSrc module 中创建一个 groovy 类，代表参数的model类

```groovy
class RouterExtension2 {
    String wiki;
}
```

3. 插件类读取参数
```groovy
def extension = project.extensions.create("router", RouterExtension2)
def path = extension.wiki
```

4. 在业务调用方传入参数
```groovy
apply plugin: "RouterPlugin2"

router {
    wiki "${getRootDir().absolutePath}/router_wiki.md"
}
```

## RouterPlugin2 的详细实现

插件中要实现三个功能
1. 在使用路由注解处理器的module，都向注解处理器插入一个参数，代表路由文件生成的路径。 
2. 在项目clean时清除掉注解中间文件
3. 将所有module生成的注解中间文件，汇总生成一个路由wiki文档

```groovy
package com.example.test

import org.gradle.api.Plugin
import org.gradle.api.Project

class RouterPlugin2 implements Plugin<Project> {

    String TAG = "RouterPlugin2"

    @Override
    void apply(Project project) {

        println("$TAG apply kotlin")

        def routerFileDir = new File(project.getRootDir(), "router_mapping")

        /**
         * kapt {
         *      rguments {
         *          arg("router_mapping", rootProject.rootProjectDir.absolutePath）
         *      }
         *  }
         **/
        // 1. 自动帮助用户传递路径参数到注解处理器中 start
        // 以下代码可以替代以上的 build.gradle 配置
        if (project.extensions.findByName("kapt") != null) {
            project.extensions.findByName("kapt").arguments {
                arg("router_file_dir", routerFileDir)
                arg("project_name", project.name)
            }
        }
        // 1. 自动帮助用户传递路径参数到注解处理器中 end

        // 2. 在clean时自动清理旧的构建产物
        project.clean.doFirst {
            def routerMappingFile = routerFileDir
            if (routerMappingFile.exists()) {
                routerMappingFile.deleteDir()
            }
        }

        // 3. 集合各个子project路由信息，生成路由文档
        def extension = project.extensions.create("router", RouterExtension2)
        project.afterEvaluate {
            // 在工程完成配置阶段之后才能获取到 外部配置的参数
            def wikiPath = extension.wiki
            project.tasks.findAll { task ->
                // 找到编译的task，compileDebugJavaWithJavac
                task.name.startsWith("compile") && task.name.endsWith("JavaWithJavac")
            }.each {task ->
                task.doLast {
                    def wikiFile = new File(wikiPath)
                    if (!wikiFile.exists()) {
                        wikiFile.createNewFile()
                    }
                    def jsonFiles = routerFileDir
                    jsonFiles.eachFile {file ->
                        def content = file.readBytes()
                        wikiFile.append(new String(content))
                    }
                    // write jsonFile content into wikiPath
                }
            }
        }
    }
}
```


