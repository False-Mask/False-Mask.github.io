---
title: 记一次在AOSP上装Google Play的尝试
tags:
 - andorid
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/google-play.1024x1024.png
date: 2024-12-14 16:45:30
---





# 记一次在AOSP上装Google Play的尝试





## 结果

![失败GIF动图免费下载- 爱给网](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/46b680f2f72a429797a6e17098e9b7a9.gif)



# 背景

> 最近在工作中有需要用到 Google Play 来验证一些事情。
>
> 但是我手中的测试机Pixel5刷了 AOSP 是不具备 Google Play 的环境的。

> 经过多次调研 & 尝试在 AOSP 上安装 Google Play都已失败告终
>
> 1.NikeGapps（Pixel 5 不能刷 Recovery）
>
> 2.[bitgapps](https://bitgapps.io/)(Pixel 5 不能刷 Recovery)
>
> 3.[microGapps](https://microg.org/)（存在一些兼容问题）

> 在无意间发现了一篇文章[StackOverflow](https://stackoverflow.com/questions/41695566/install-google-apps-on-aosp-build)
>
> 文章大致内容是说可以通过提取官方镜像中的 APK，导入到 AOSP 中。
>
> 虽然这篇文章很早之前都看过，但是觉得比较麻烦，就没尝试。
>
> 本文章也算是最终尝试的记录。（要是还不行我真撂挑子了 这个 GP 谁爱装谁装了 : ）



# 过程



## 确认官方镜像的版本

通过Settings -> About Phone -> Build Number 可知AOSP 的版本

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241211232934789.png" alt="image-20241211232934789" style="zoom:25%;" />

我的是 TQ3A.230901.001.C2 其实也就是 android-13.0.0_r78 (Pixel 5)

经过确认可以下载的官方镜像版本是https://dl.google.com/dl/android/aosp/redfin-tq3a.230901.001.c2-factory-ca20bd02.zip?hl=zh-cn







## 提取APK



1.下载后我们得到一个压缩包，压缩包解压后如下。

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241211233127868.png" alt="image-20241211233127868" style="zoom: 50%;" />

2.再解压后如下

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241211233228358.png" alt="image-20241211233228358" style="zoom: 50%;" />

3.挂载 system 镜像

a. 查看文件类型

```shell
➜  image-redfin-tq3a.230901.001.c2 file system.img 
system.img: Linux rev 1.0 ext2 filesystem data, UUID=4f99442d-5b10-59c7-bafe-1ee733580bd7 (extents) (large files) (huge files)
```

b.挂载镜像(最好用 Linux 挂载)

```shell
sudo mount -o ro,loop system.img /mnt/system
```

> Note：
>
> 这里小小的踩了一个坑。Mac 针对于 ext文件系统支持不太好https://v2ex.com/t/658836
>
> “Linux 是最好的学习设备，不接受反驳”

c.提取

```shell
~ cd /mnt/system #进入挂载的系统镜像路径
~ cd system/ #进入系统路径
~ tar -cvf ~/Downloads/priv-app.tar priv-app #提取 apk
```

> 如下是system.img内的priv-app集合。but我找了以后发现没有我想要的google 3件套

![image-20241212001952936](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241212001952936.png)



> 最后通过mnt后发现三件套主要在
>
> system_ext、product 分区。
>
> 提取流程跟上面是一样的



## 安装



1.remount

```shell
adb remount # syetem 分区是 readonly。需要 remount 才能修改
```

2.push

```shell
adb push GoogleServicesFramework Phonesky PrebuiltGmsCore /system/priv-app/
```



## bootloop



> 最终的结果很显然，bootloop了（循环死机）

> 很无奈。通过重新flash system分区修复了这个问题

```shell
fastboot flashing system system.img
```





## 最后



第一点从理论上我知道，google play肯定是可以刷的。但是结果上我失败了。（就是菜的缘故）

理论上应该是漏掉了一些东西，[priv-app特殊权限](https://source.android.com/docs/core/permissions/perms-allowlist?hl=zh-cn)

但是也给了我一些启发吧。毕竟实践出真理（追求真理的同时需要planB，实验过程中差点手机就成砖了，好不容易救回来了。haha～）

**“越是菜越是爱折腾，我觉得我很励志。”**





最后

> La fanfare frémit au carrefour de ta forme
>
> 当你打开束缚你的牢笼
>
> Martellant sa poésie diforme 
>
> 我们将去乌托邦
>
> ——2024-12/14 

# refs



[挂载 system 文件](https://blog.csdn.net/netwalk/article/details/140108965)