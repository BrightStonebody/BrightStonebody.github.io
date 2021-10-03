---
title: 通知铃声的自定义遇到的问题-根据uri读取文件
date: 2019-08-28 23:07:53
tags:
- Android
- Notification
- Uri
Categories:
- Android
---

# 通知铃声自定义遇到的问题: 根据uri读取文件 

## 问题：
在自定义通知铃声是，当铃声的uri来自本地文件，或者是项目的资源文件，当通知弹出时，没有声音

## 问题根源：
### 1.从系统文件管理器中获取文件，从得到的uri中获取原始路径

**所有尝试从Uri中获取原始路径的方法，都是不推荐和不可靠的**

网上大部分的方法都是解析Uri，使用cursor读取对应的字段，然后获取到文件的路径，但是，经过尝试后发现，大部分机型中，cursor并没有那个字段，当然也就得不到对应的路径。

正确的做法应当是，使用ContentResolver获取到对应的文件流，拷问文件到App的目录中。

```
InputStream fis = getContentResolver().openInputStream(uri);
```

### 2. 将拷贝的文件转化为uri

**失败的尝试：使用Uri.fromFile():**
首先，App将自己的铃声文件设置到notificaiton中，系统要弹出一个notification时，会调用我们的项目目录的铃声文件。

在Android7.0以上的版本中，对这种行为进行了限制，获取文件的Uri一定要使用FileProvider，而不能直接调用fromFile()方法.

### 3. grantUriPermission()

在某些机型中，使用file自定义notification的声音，仍然不能正常工作。
**设置logcat的Filter选项为No Filter, 会出现FileProvider Permission denial 的 warning**
为什么要选择No Filter选项才会看到这个warning。
是因为，弹出notification是系统服务，和我们的APP项目不是一个包名
在file转uri之后添加一个授权的语句就可以了：
```
getApplicationContext().grantUriPermission("com.android.systemui",
                Uri.parse(info.uri), Intent.FLAG_GRANT_READ_URI_PERMISSION);
```
包名com.android.systemui是系统服务的一个包名，负责notification、StatusBar等的相关操作

到这里问题就解决了
