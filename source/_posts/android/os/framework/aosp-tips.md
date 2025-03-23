---
title: Aosp小tips
tags:
  - aosp
date: 2025-03-23 23:41:17
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-andvanced-development-tricks.png
---


# 关于本文



> 主要记录在AOSP调试学习过程中遇到的坑～

![aosp-andvanced-development-tricks](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-andvanced-development-tricks.png)



# Android 14



## userdebug 编译out of space? out of inodes? the tree size XXX 



- 问题：

> out of space? out of inodes?XX
>
> 这个报错包含有3个原因。
>
> 1. 内存耗尽
> 2. inodes耗尽
> 3. 生成的文件超过文件系统的大小限制
>
> 
>
> ***这里的报错主要是因为#3导致，原因如下：***
>
> img文件生成目前主要有两种方式ex4和f2fs, 图中的报错主要是由于userdata默认采用了f2fs导致大小超过了f2fs的最大限制，需要***调整文件类型为ext4***并且将***文件的大小限制设置为合适的大小***

![Image_410772202840384](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/Image_410772202840384.jpg)

- 解决方式

https://juejin.cn/post/7484089347156295714



> note：
>
> 1. 报错原因有三个！！需要明确原因。不能盲目抄
> 2. 需要明确报错的文件，每个img类型都有不同的参数控制
>
> 3. 需要明确lunch type,不同的lunch type可能配置的参数不一样！！



## userdebug无法调试的问题





- 问题：

1.现象一

``` shell
# 当我们调用adb jdwp查看可调式的应用时会一直卡住。（）
➜  ~  adb jdwp
^C
```

2.现象二

> 设备能被识别，当时attach process找不到设备（右上角可见设备名称）

![Screenshot from 2025-03-23 23-23-12](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/Screenshot from 2025-03-23 23-23-12.png)



- 解决方案：

https://www.bilibili.com/opus/911162975264440345

