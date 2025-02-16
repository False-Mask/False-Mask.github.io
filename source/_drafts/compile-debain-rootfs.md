---
title: compile-debain-rootfs
tags:
cover:
---



# 创建Debain rootfs



refs：

[博客1](https://www.liuwantong.com/2021/02/16/debian-rootfs/)

[博客2](https://ivonblog.com/posts/debootstrap-create-rootfs-for-android/)

[Debian Wiki](https://wiki.debian.org/zh_CN/Debootstrap)



# 背景







# 整体思路



其实通过上述的refs很容易能看出来rootfs的安装流程很简单。通过debootstrap，下载安装到制定路径。

下载安装完成以后就可以通过chroot check是否生效。





# Debian版本？



> 版本的选择是安装过程遇到的第一个问题，由于Debian是和Linux内核版本绑定的，所以我们需要根据内核版本确认Debian的安装版本。



> 经过和Gpt的深入交流，总结得知Debain的默认内核版本可以通过如下方式查找。（不唯一）

1. 首先我们可以通过release note地址去查看当前release的debian版本。

[Debian Release Note](https://www.debian.org/releases/)

![image-20250119221211832](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119221211832.png)



2.点击指定版本的link以后、进入release note，选择关注的架构![](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119221752992.png)



![image-20250119221819347](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119221819347.png)

搜索Kernel以后可发现，debian 10 ～ debian 11 默认内核从4.19 更新到了5.10

![image-20250119221850730](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119221850730.png)



因为目前Arm内核为5.10因此，Debian版本号就确定了，就是Debian 11～



# 安装



> 确认版本以后我们就可以开始愉快的安装了。



1.安装qemu(用于模拟其他CPU指令集，并虚拟操作系统)

```shell
sudo apt install qemu-user-static

```



2.安装debootstrap

```shell
sudo apt install debootstrap
```



3.安装debian 11

```shell
sudo debootstrap --arch arm64 --components=main,universe bullseye  my-debian http://ftp.cn.debian.org/debian/
```

