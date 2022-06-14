---
title: transform和asm初探
date: 2022-03-21 21:54:26
tags:
- Gradle
categories:
- Android
---

## 自定义RouterTransform
Transform 是AGP官方提供的接口，在 class->dex 的阶段提供一个时机，可以让我们对字节码文件做修改，或者动态添加一个新的类

### 添加依赖
因为是安卓官方提供的能力，需要在 buildSrc 的 build.gradle 中 添加 google() 仓库和安卓构建工具的依赖

```groovy
repositories {
    jcenter()
    google()
    mavenCentral()

}

dependencies {
    implementation gradleApi()
    implementation localGroovy()
    implementation "com.android.tools.build:gradle:3.5.3"

}
```

### 编写 RouterTranform

```groovy
class RouterTransform extends Transform {

    @Override
    String getName() {
        return "RouterMappingTransform"
    }

    /**
     * 需要插桩的输入类型， 这里是类文件
     * @return
     */
    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    /**
     * 需要插桩的范围, 这里是整个工程
     * @return
     */
    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    /**
     * 是否支持增量
     * @return
     */
    @Override
    boolean isIncremental() {
        return false
    }

    /**
     * 实现
     * @param transformInvocation
     * @throws TransformException
     * @throws InterruptedException
     * @throws IOException
     */
    @Override
    void transform(TransformInvocation transformInvocation)
        throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation)

        println("transform start")

        def collector = new RouterMappingCollector()

        // 遍历所有的输入
        transformInvocation.inputs.each {
            // 把 文件夹 类型的输入，拷贝到目标目录
            it.directoryInputs.each { directoryInput ->
                def destDir = transformInvocation.outputProvider
                    .getContentLocation(
                        directoryInput.name,
                        directoryInput.contentTypes,
                        directoryInput.scopes,
                        Format.DIRECTORY)

                collector.collect(directoryInput.file)
                FileUtils.copyDirectory(directoryInput.file, destDir)
            }

            // 把 JAR 类型的输入，拷贝到目标目录
            it.jarInputs.each { jarInput ->
                def dest = transformInvocation.outputProvider
                    .getContentLocation(
                        jarInput.name,
                        jarInput.contentTypes,
                        jarInput.scopes, Format.JAR)
                collector.collectFromJarFile(jarInput.file)
                println("transform jar input path ${jarInput.file.absolutePath}")
                println("transform jar output path ${dest.absolutePath}")
                FileUtils.copyFile(jarInput.file, dest)
            }
        }


        File mappingJarFile = transformInvocation.outputProvider.
            getContentLocation(
                "router_mapping",
                getOutputTypes(),
                getScopes(),
                Format.JAR)

        println("${getName()}  mappingJarFile = $mappingJarFile")

        if (mappingJarFile.getParentFile().exists()) {
            mappingJarFile.getParentFile().mkdirs()
        }

        if (mappingJarFile.exists()) {
            mappingJarFile.delete()
        }

        // 将生成的字节码，写入本地文件
        FileOutputStream fos = new FileOutputStream(mappingJarFile)
        JarOutputStream jarOutputStream = new JarOutputStream(fos)
        // CLASS_NAME = "com/imooc/router/mapping/generated/RouterMapping"
        ZipEntry zipEntry =
            new ZipEntry(RouterMappingByteCodeBuilder.CLASS_NAME + ".class")
        jarOutputStream.putNextEntry(zipEntry)
        println("transform collect class ${collector.mappingClassName}")
        jarOutputStream.write(
            // 写入字节码
            RouterMappingByteCodeBuilder.get(collector.mappingClassName)
        )
        jarOutputStream.closeEntry()
        jarOutputStream.close()
        fos.close()

        println("transform end")
    }
}
```

* `transformInvocation.inputs.each {`
表示遍历所有输入
* `it.directoryInputs.each { directoryInput ->`
表示遍历所有目录文件输入，directoryInput 是class文件的目录
* `it.jarInputs.each { jarInput ->`
遍历所有jar包输入， jarInput 是jar包
* `transformInvocation.outputProvider.getContentLocation(`
获取具体的输入
* `collector.collect(directoryInput.file)`
这里是一个自定义逻辑，表示从 class文件 或 jar包 中找到生成的 RouterMapping 路由映射类。 
* `FileUtils.copyFile(jarInput.file, dest)`
拷贝文件到目标目录， 最终transform执行之后会在 /...项目目录/app/build/intermediates/transforms 中生成一个RouterMappingTransform目录。 需要吧class文件和jar包都拷贝过去。有意思是，我原本以为我们工程自己的代码应该都是 class文件，都在 `it.directoryInputs.each { directoryInput ->` 这个输入里。 但是后来发现，只有 app工程是 class 文件，子工程打包成了 jar 包，和其他第三方库放在了一起

* 后面的代码都是动态生成了一个集合了所有子工程路由注册的路由类，使用asm写入了字节码

**即使我们没有对文件做任何处理，我们仍然需要把输入文件拷贝到目标目录下，否则下一个Task就没有TansformInput，如果我们没有将input目录复制到output指定目录，会导致最后的打包的apk缺少class**


## 注册transform

```groovy
class RouterPlugin2 implements Plugin<Project> {

    String TAG = "RouterPlugin2"

    @Override
    void apply(Project project) {

        if (project.plugins.hasPlugin(AppPlugin)) {
            // hasPlugin(AppPlugin) 表示这是 app 主工程
            // 注册进我们的 RouterTransform
            def extension = project.extensions.getByType(AppExtension)
            extension.registerTransform(new RouterTransform())
        }
    }
}
```

插件也是需要注册的。。。在resource目录的方式注册。 不写了，网上搜一下吧。

## 使用我们的 Router

使用反射实例化我们的类即可
```kotlin
        try {
            // GENERATED_MAPPING = "com.imooc.router.mapping.generated.RouterMapping"
            // 注意上面原来文件"/"的分割要变成"."
            val clazz = Class.forName(GENERATED_MAPPING)
            val method = clazz.getMethod("get")
            val allMapping = method.invoke(null) as Map<String, String>

            if (allMapping?.size > 0) {
                Log.i(TAG, "init: get all mapping:")
                allMapping.onEach {
                    Log.i(TAG, "    ${it.key} -> ${it.value}")
                }
                mapping.putAll(allMapping)
            }

        } catch (e: Throwable) {
            Log.e(TAG, "init: error while init router : $e")
        }
```


