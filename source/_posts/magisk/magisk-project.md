---
title: Magisk基础
tags:
  - magisk
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/Magisk_Logo.png
date: 2024-06-02 19:41:15
---




# Magisk基础



[项目源代码地址](https://github.com/topjohnwu/Magisk)



# Magisk是什么？有什么用？



> Magisk是一个开源的用于获取Root权限的框架



> Magisk可以用于获取手机Root权限



# Magisk如何使用



> 具体的使用可见手册

[官方手册](https://topjohnwu.github.io/Magisk/install.html)



简单来说：

环境需要：

1. 已经解除BootLoader锁。
2. 已经安装adb、fastboot工具（以及驱动）
3. 系统boot镜像



使用步骤:

1. 根据是否有ramdisk分区确认初始镜像，如果有获取系统的boot/init_boot镜像，如果没有获取recovery镜像
2. 使用Magisk App对初始进行进行patch操作
3. 使用fastboot刷入boot/init_boot/recovery镜像





# Magisk项目源代码环境配置



> 具体内容可见

[官方手册](https://topjohnwu.github.io/Magisk/build.html)



1.设置环境

> 先下载Magisk项目
>
> ```shell
> git clone --recurse-submodules https://github.com/topjohnwu/Magisk.git
> ```
>
> 然后下载Magisk定制的ndk
>
> ```shell
> ./build.py ndk
> ```

2.编译项目

> ```shell
> $: python3 build.py --help
> actions:
> 	all                 build everything
>     binary              build binaries
>     cargo               run cargo with proper environment
>     app                 build the Magisk app
>     stub                build the stub app
>     emulator            setup AVD for development
>     avd_patch           patch AVD ramdisk.img
>     clean               cleanup
>     ndk                 setup Magisk NDK
> 
> ```
>
> 



# Magisk项目源代码的结构



> Magisk项目有4个module

```kotlin
rootProject.name = "Magisk"
include(":app", ":app:shared", ":native", ":stub")
```



- app

  > 就是Magisk App

- app:shared

  > 共享通用的功能

- native

  > c++编写的一些功能代码

- stub

  > 用于躲避扫描apk的，占位apk





# references



[Magisk官方文档](https://topjohnwu.github.io/Magisk)
