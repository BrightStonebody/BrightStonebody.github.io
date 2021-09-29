---
title: 编译aosp
date: 2021-05-22 17:12:59
tags:
- AOSP
---

# 注意事项
1. 在macOS上，默认文件系统是不区分大小写的，需要先建立一个区分大小写的磁盘


# 下载源代码

1. 下载 Repo 工具，并确保它可执行：

```shell
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

使用清华源
```shell
curl https://mirrors.tuna.tsinghua.edu.cn/git-repo-downloads/repo  > ~/bin/repo
chmod a+x ~/bin/repo
```

2. 初始化 Repo

-b 后面可以指定要同步代码的安卓系统版本代号
版本代号可以在下面的链接中查找
https://source.android.com/setup/start/build-numbers

```shell
repo init -u https://android.googlesource.com/platform/manifest -b android-4.0.1_r1
```

使用清华源
```shell
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-9.0.0_r40
```

3. 同步 Repo

需要几个小时的时间同步代码
```shell
repo sync 
```

# 编译

执行下面的命令，
lunch 选择编译模式（ 这里我选择的是  aosp_x86_64-eng）
m （m 命令系统会根据cpu性能自动选取需要使用的线程，你也可以根据cpu多少使用 make -jN, N表示cpu个数x2）


```shell
source build/envsetup.sh
lunch  aosp_x86_64-eng
m 
```

## 坑点
* 找不到对应的MacOSX.sdk
Could not find a supported mac sdk: [“10.10” “10.11” “10.12” “10.13”]

我的系统版本是10.15，你需要到 https://github.com/phracker/MacOSX-SDKs/releases 下载需要的版本

我这里下载的是MacOSX10.12.sdk，解压复制到如下目录：
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs

* 不知名报错、

找到文件：源文件根目录/system/sepolicy/tests/Android.bp，删除掉第14行的一行代码：

```
    // libsepolwrap gets loaded from the system python, which does not have the
    // ASAN runtime. So turn off sanitization for ourself, and  use static
    // libraries, since the shared libraries will use ASAN.
    static_libs: [
        "libbase",
        "libsepol",
    ],
    stl: "libc++_static", // 删除掉这一行
    sanitize: {
        never: true,
    },
```

# 使用 Android Studio 阅读源码

1. 生成 Android Studio 工程配置文件

生成 android.iml 和 android.ipr 文件。

其中 iml 文件 表示 information of modules, 用来描述 AOSP 的模块信息。
ipr 文件 表示 IDEA project configuration ，用来描述 IDEA 的工程配置信息，双击此文件时系统将直接使用 Andorid Studio 打开此项目。

```shell
# 设置 AOSP 编译所需的环境变量
source build/envsetup.sh
# 使用 idegen.sh 脚本生成 IDEA 工程文件
development/tools/idegen/idegen.sh
```


