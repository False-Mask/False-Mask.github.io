---
title: 构建Pixel Kernel
date: 2025-06-28 23:46:26
tags:
  - aosp
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/android_stack.png
---




# 构建Android Kernel



> 手机设备为Pixel 6



> 编译主机为Ubuntu 24.04

```shell
                          ./+o+-       rose@rose
                  yyyyy- -yyyyyy+      OS: Ubuntu 24.04 noble
               ://+//////-yyyyyyo      Kernel: x86_64 Linux 6.11.0-26-generic
           .++ .:/++++++/-.+sss/`      Uptime: 14h 10m
         .:++o:  /++++++++/:--:/-      Packages: 2896
        o:+o+:++.`..```.-/oo+++++/     Shell: zsh 5.9
       .:+o:+o/.          `+sssoo+/    Resolution: 3840x2160
  .++/+:+oo+o:`             /sssooo.   DE: GNOME 46.7
 /+++//+:`oo+o               /::--:.   WM: Mutter
 \+/+o+++`o++o               ++////.   WM Theme: Adwaita
  .++.o+++oo+:`             /dddhhh.   GTK Theme: Yaru [GTK2/3]
       .+.o+oo:.          `oddhhhh+    Icon Theme: Yaru
        \+.++o+o``-````.:ohdhhhhh+     Font: Ubuntu Sans 11
         `:o+++ `ohhhhhhhhyo++os:      Disk: 2.1T / 5.5T (41%)
           .o:`.syhhhhhhh/.oo++o`      CPU: AMD Ryzen 9 7950X 16-Core @ 32x 5.881GHz
               /osyyyyyyo++ooo+++/     GPU: AMD Radeon Graphics (radeonsi, raphael_mendocino, LLVM 19.1.1, DRM 3.61, 6.11.0-26-generic)
                   ````` +oo+++o\:     RAM: 13761MiB / 95654MiB
                          `oo++.      

```



# 前置准备



1.抉择内核版本Pixel Kernel or GKI Kernel

> 针对于Pixel设备可选的内核分支有两个(非Pixel 设备就没有这个“甜蜜”的烦恼了。)
>
> - Pixel Kernel (厂商内核)——**性能强，内核版本更新不及时。**
>
>   Google团队内部自己维护的内核，部分开源，有很多Pixel设备的定制化优化，但内核版本较低。
>
> - GKI Kernel  (通用内核)——**性能弱，内核版本更新及时。**
>
>   完全开源的内核，仅包含通用优化，无任何专属优化，但版本更新迅速。

> 本文构建使用Pixel内核。



2.Pixel Kernel分支选择

> 可见[kernel manifest](https://android.googlesource.com/kernel/manifest/+refs)
>
> Pixel Kernel 分支为：
>
> android-gs-XXX-kernelVersion-androidVersion

> 由于笔者aosp版本为android 14.0.0_r73因此选用android-gs-raviole-5.10-android14-qpr3



# 下载



> 镜像站文档中好像都没有说明内核的镜像地址在那。
>
> 简单看了下中科大的教程https://mirrors.ustc.edu.cn/help/aosp.html推断出来的。

```shell
repo init -u http://mirrors.ustc.edu.cn/aosp/kernel/manifest -b android-gs-raviole-5.10-android14-qpr3
```



> 坑点（低版本才有的问题）
>
> 低版本repo manifest有问题会导致仓库绕过镜像站，从google 仓库下载。
>
> (别问我是怎么知道的，流量哗哗如流水。)
>
> repo sync以前记得修改.repo/manifests/default.xml文件内容

```xml
<manifest>

  <remote  name="aosp"
    fetch="https://android.googlesource.com/" /> # 有问题的代码，需要修改
    fetch=".." # 修改后
    
    
  <default revision="android-gs-raviole-5.10-android14-qpr3"
    remote="aosp" sync-j="4" /
    
    
</manifest>
```





# 构建



> 可以发现Pixel6属于是"天选之机"
>
> 他是首个支持GKI Kernel的手机
>
> 因此他需要做一些特殊的操作

![**Figure 1.** Kernel Update Flow Chart](https://source.android.com/static/docs/setup/build/images/flashing-flow-chart.png)



- Update the vendor ramdisk(只有Pixel 6系列需要做该操作)

> Note：
>
> 回答几个问题
>
> 1.Pixel 7以后为什么可以不用提取ramdisk？
>
> Pixel 7及以后的设备将vendor_boot分为了两部分
>
> - vendor_boot 包含供应商扩展的 ramdisk 和硬件相关文件。
> - vendor_kernel_boot 存放内核的构建产物
>
> 2.为什么Pixel 6设备需要提取ramdisk
>
> Pixel 6没有拆分vendor_kernel_boot因此包含两部分内核代码和ramdisk。但是我们其实只用更新其中的内核产物，不用更新ramdisk以及硬件相关文件。
>
> 因此Pixel6需要将ramdisk拷贝过去。

Extract the vendor ramdisk image from the [Pixel factory image](https://developers.google.com/android/images).

——也就是从vendor_boot中提取出ramdisk.img，当然不一定非得是Pixel Factory Image, 任何的镜像都可以的。

但是必须是手机当前正在使用的镜像！！！

``` shell
1. 获取vendor_boot.img （如果是原装镜像，可以从官网中搜索。 如果是自己打的也可以直接拿来用。 如果不知道镜像是啥，可以通过fastbot fetch PARTITION OUT_FILE）
......

2. 提取ramdisk
~ $KERNEL_REPO_ROOT/tools/mkbootimg/unpack_bootimg.py --boot_img vendor_boot.img \
      --out vendor_boot_out

3. 将ramdisk拷贝到源码路径下
Note: 关于DEVICE_RAMDISK_PATH的取值
Pixel 6 (oriole)/ Pixel 6 Pro (raven)	prebuilts/boot-artifacts/ramdisks/vendor_ramdisk-oriole.img
Pixel 6a (bluejay)	private/devices/google/bluejay/vendor_ramdisk-bluejay.img

cp vendor_boot_out/vendor-ramdisk-by-name/ramdisk_ \
      $KERNEL_REPO_ROOT/$DEVICE_RAMDISK_PATH
```



- 编译

```shell
1. android13-5.15 以下采用如下方式打包
tools/bazel run --lto=thin //gs/google-modules/soc-modules:slider_dist

2. 其他内核采用
build_slider.sh（Pixel 6）
build_xxx.sh (非Pixel 具体指令可见https://source.android.com/docs/setup/build/building-pixel-kernels#compile_the_kernel_kleaf)
```



- 编译成功

```shell

#  ......
========================================================
 Copying files
  arch/arm64/boot/dts/google/gs101-a0.dtb
  arch/arm64/boot/dts/google/gs101-b0.dtb
  arch/arm64/boot/dts/google/dtbo.img
  arch/arm64/boot/dts/google/gs101-dpm-eng.dtbo
  arch/arm64/boot/dts/google/gs101-dpm-user.dtbo
  arch/arm64/boot/dts/google/gs101-dpm-userdebug.dtbo
========================================================
 Installing UAPI kernel headers:
make: Entering directory '/mnt/data/code/android-kernel/out/mixed/device-kernel/private/gs-google'
  INSTALL /mnt/data/code/android-kernel/out/mixed/device-kernel/kernel_uapi_headers/usr/include
make: Leaving directory '/mnt/data/code/android-kernel/out/mixed/device-kernel/private/gs-google'
 Copying kernel UAPI headers to /mnt/data/code/android-kernel/out/mixed/dist/kernel-uapi-headers.tar.gz
========================================================
 Copying kernel headers to /mnt/data/code/android-kernel/out/mixed/dist/kernel-headers.tar.gz
/mnt/data/code/android-kernel/private/gs-google /mnt/data/code/android-kernel
/mnt/data/code/android-kernel
========================================================
 Copying modules files
========================================================
 Creating initramfs
========================================================
 Trimming unused modules
========================================================
 Creating vendor_dlkm image
========================================================
 Trimming unused modules
========================================================
 Copying unstripped module files for debugging purposes (not loaded on device)
========================================================
 Files copied to /mnt/data/code/android-kernel/out/mixed/dist
/mnt/data/code/android-kernel/prebuilts/boot-artifacts/ramdisks/vendor_ramdisk-oriole.img is LZ4 compressed
Created vendor_boot.img at /mnt/data/code/android-kernel/out/mixed/dist/vendor_boot.img
```





# 刷入



- 输出位置

> 输出保存位置
>
> 可根据内核版本选择

| Kernel branch   | DIST_DIR          |
| :-------------- | :---------------- |
| v5.10           | `out/mixed/dist`  |
| v5.15 and later | `out/DEVICE/dist` |



- 关闭校验

```shell
fastboot oem disable-verification
```



- 清空data分区

> 文中说的而是如果出现了内核patch降级才需要清空数据，
>
> 如果是升级理论上应该不需要，可根据自身条件使用。

``` shell
fastboot -w
```



- 刷入



> 如下是不同设备需要刷入分区
>
> 其中标明dynamic partition表示需要进入fastbootd进行刷入(即adb reboot fastboot)
>
> 未标明表示可在bootloader刷入(即adb reboot bootloader)



| Device                                                       | Kernel Partitions                                            |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| Pixel 6 (oriole) Pixel 6 Pro (raven) Pixel 6a (bluejay)      | boot <br />dtbo <br />vendor_boot <br />vendor_dlkm *(dynamic partition)* |
| Pixel 8 (shiba) Pixel 8 Pro (husky) Pixel Fold (felix) Pixel Tablet (tangorpro) Pixel 7a (lynx) Pixel 7 (panther) Pixel 7 Pro (cheetah) | boot <br />dtbo <br />vendor_kernel_boot <br />vendor_dlkm *(dynamic partition)* <br />system_dlkm *(dynamic partition)* |



> 如下为Pixel6刷机指令

```shell
fastboot flash boot        $out_dir/boot.img
fastboot flash dtbo        $out_dir/dtbo.img
fastboot flash vendor_boot $out_dir/vendor_boot.img
fastboot reboot fastboot
fastboot flash vendor_dlkm $out_dir/vendor_dlkm.img
```





# Q & A



## 怎么定位设备GKI内核分支



> 执行如下shell脚本

``` shell
~ adb shell uname -a
Linux localhost 5.10.198-android13-4-00050-g12f3388846c3-ab11920634 #1 SMP PREEMPT Mon Jun 3 20:51:42 UTC 2024 aarch64 Toybox
```



> 信息就藏在输出中g12f3388846c3

> 其中去掉g后的输出就是gki的commit信息



找到以后，我们就可以通过将kernel/common checkout到该commitId, 根据git即可知道branch.



Note: 如果不确认自己的kernel是否是gki kernel可通过如下链接确定。

https://android.googlesource.com/kernel/common/+/你的commit号

如果能找到就说明是gki版本号。



## GKI内核是什么？架构？



简单来说就是Google为了防止内核和厂商代码过度耦合，导致系统内核版本无法升级

将内核和厂商代码做了解耦和，在内核和厂商之间加了一层KMI接口，同时原来的内核也被Google纳入到了维护/升级范围中变成了GKI Kernel

https://source.android.com/docs/core/architecture/kernel/generic-kernel-image



![GKI architecture](https://source.android.com/static/docs/core/architecture/images/generic-kernel-image-architecture.png)



## 如果我想为设备刷GKI,怎么选择GKI内核版本



1.确认当前设备的KMI版本是多少

具体可见https://source.android.com/docs/core/architecture/kernel/gki-versioning

例：

如下内核的KMI Version为5.10-android13-4

``` shell
~ adb shell uname -a
Linux localhost 5.10.198-android13-4-00050-g12f3388846c3-ab11920634 #1 SMP PREEMPT Mon Jun 3 20:51:42 UTC 2024 aarch64 Toybox
```



2.确认GKI是否兼容当前设备

https://source.android.com/docs/core/architecture/kernel/android-common#compatibility-between-kernels

a. GKI Kernel可以跨越Android的系统版本

如：android14-6.1能在android15的设备上运行。但是android15-6.1不能在android14上运行，即内核版本能对上。低版本能在高版本运行。

b. KMI不支持跨版本比如android14-6.1是不兼容android15-6.6内核的。

c. KMI是支持当前kernel的前缀版本如android15-6.6.2是可以安装在android15-6.1.1上。







## GKI内核和Pixel内核的区别是什么



GKI内核是毛坯，Pixel内核是精装修。

Pixel内核是在GKI内核的基础上做了一些定制的驱动 & 优化。





## 怎么为Pixel设备选择内核版本





1.查看最新的Kernel版本

可见如下网页：

https://source.android.com/docs/setup/build/building-pixel-kernels

但是上述网页的Kernel版本是最新版。看不到历史版本。



2.查看历史的Kernel版本

进入网页，搜索android-gs-设备名称

（设备名称可从#1中网页查看）

https://android.googlesource.com/kernel/manifest/+refs

如Pixel6 (raviole)的历史版本为

- [android-gs-raviole-5.10-android12-d1](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android12-d1)
- [android-gs-raviole-5.10-android12-qpr1-d](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android12-qpr1-d)
- [android-gs-raviole-5.10-android12-qpr3](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android12-qpr3)
- [android-gs-raviole-5.10-android12L](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android12L)
- [android-gs-raviole-5.10-android13](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android13)
- [android-gs-raviole-5.10-android13-qpr1](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android13-qpr1)
- [android-gs-raviole-5.10-android13-qpr2](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android13-qpr2)
- [android-gs-raviole-5.10-android13-qpr3](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android13-qpr3)
- [android-gs-raviole-5.10-android14](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android14)
- [android-gs-raviole-5.10-android14-qpr1](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android14-qpr1)
- [android-gs-raviole-5.10-android14-qpr2](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android14-qpr2)
- [android-gs-raviole-5.10-android14-qpr3](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android14-qpr3)
- [android-gs-raviole-5.10-android15-qpr1](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-5.10-android15-qpr1)
- [android-gs-raviole-6.1-android15-qpr2-beta](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-6.1-android15-qpr2-beta)
- [android-gs-raviole-6.1-android16](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-6.1-android16)
- [android-gs-raviole-6.1-android16-beta](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-6.1-android16-beta)
- [android-gs-raviole-6.1-android16-dp](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-6.1-android16-dp)
- [android-gs-raviole-mainline](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-raviole-mainline)



## 什么情况下选择Pixel Kernel? 什么情况下选择GKI Kernel?



省流：Pixel内核 = GKI内核 + Pixel定制优化

因此Pixel设备打GKI内核，其他设备不需要也不能用Pixel 内核，选择GKI内核



Pixel 设备最好打Pixel Kernel(性能好，有定制驱动 & 优化)，如果能接受损失性能也可以打GKI内核，只是需要注意版本兼容。

非Pixel设备，如三星，小米等打特定厂商的开源内核 or GKI Kernel







# refs



[官网-Build Pixel Kernels](https://source.android.com/docs/setup/build/building-pixel-kernels)

[官网-Building Kernels](https://source.android.com/docs/setup/build/building-kernels)

[官网-GKI 版本号](https://source.android.com/docs/core/architecture/kernel/gki-versioning)



[中科大镜像站](https://mirrors.ustc.edu.cn/help/aosp.html)
