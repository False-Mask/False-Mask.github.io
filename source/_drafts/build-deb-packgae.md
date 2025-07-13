---
title: build-deb-packgae
tags:
cover:
---



# 构建deb包



# 背景



> 在配置Android ebpf环境的过程中打包了linux-header，但是linux-header是一个tar文件。
>
> 而寻常linux-header都是通过apt直接安装，突然就想着能不能把我自己编译的linux-header.tar也打成一个
>
> deb文件。



# 过程



## 解压并整理tar包



``` shell
tar -xvf linux-header.tar -C linux-header/linux
```



## 初始化模板



> 这个时候会生成一个debian/目录

``` shell
cd linux-header  && dh_make --createorig -s -n -p pixel6-5.10-kernel-headers_1.0.0-beta1
```



## 编辑模板



1. install规则

> 此规则声明了包在安装过程中的隐射规则。
>
> linux/* usr/src/linux-headers-5.10.198-android13-4-00050-g12f3388846c3-ab11920634表明
>
> linux路径下的所有文件最终在安装后会拷贝到usr/src/linux-headers-5.10.198-android13-4-00050-g12f3388846c3-ab11920634下

``` shell
echo "linux/* usr/src/linux-headers-5.10.198-android13-4-00050-g12f3388846c3-ab11920634" > debian/install
```



2. 修改debian/control信息

> 即deb的基础信息

``` shell
Source: pixel6-5.10-kernel-headers
Section: devel
Priority: optional
Maintainer: rose <rose@unknown>
Rules-Requires-Root: no
Build-Depends:
 debhelper-compat (= 13),
Standards-Version: 4.6.2
Homepage: https://false-mask.github.io/
#Vcs-Browser: https://salsa.debian.org/debian/pixel6-5.10-kernel-headers
#Vcs-Git: https://salsa.debian.org/debian/pixel6-5.10-kernel-headers.git

Package: pixel6-5.10-kernel-headers
Architecture: any
Multi-Arch: foreign
Depends:
 ${shlibs:Depends},
 ${misc:Depends},
Description: Custom kernel headers from tar archive
 These headers are manually packaged for specific development needs.
```



## 构建生成deb包



``` shell
dpkg-buildpackage -us -uc
# 输出文件：../pixel6-kernel-headers_5.10.123-1_amd64.deb
```



## 问题



> 发现暂时不能使用

``` shell
root@localhost:/tmp# dpkg -I pixel6-5.10-kernel-headers_1.0.0-beta1_amd64.deb 
dpkg-deb: error: archive 'pixel6-5.10-kernel-headers_1.0.0-beta1_amd64.deb' uses unknown compression for member 'control.tar.zst', giving up
```



>  查阅文档得知，打包镜像的操作在高系统版本属于多余操作。
>
> 因为内核模块中自带kheader.（笔者内核是自编内核，不需要做该操作）

https://bbs7.kanxue.com/thread-274546.htm

![image-20250706233434896](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250706233434896.png)
