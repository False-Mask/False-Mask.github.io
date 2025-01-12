---
title: 记录一次Debug AOSP system_server遇到的问题
tags:
  - android
  - aosp
  - error
cover: 'https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/android_stack.png'
date: 2025-01-12 22:40:59
---




![AOSP](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/android_stack.png)



# Debug AOSP system_server



系统环境：android-13.0.0_r78

问题描述：

在一次调试system_service dexopt的过程中发现一个很奇怪的现象：

“局部变量指拿不到”

![image-20250112210338195](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250112210338195.png)



问题线索：其实这个很容易就能想到，大概率就是被混淆了。导致局部变量的名称发生变化。





# 信息检索



[CSDN博客——“AOSP 14 framework debug无法看到变量的问题原创”](https://blog.csdn.net/longjie1986/article/details/139255951)



通过上文我们可以知道，主要原因是framework/base/services模块有一处定义了代码混淆逻辑



# 解决问题



尝试注销掉这部分代码。

frameworks/base/services/Android.bp

![image-20250112213830411](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250112213830411.png)



重新编译

```shell
mmm frameworks/base/services/
```



编译完成后刷入设备中

> 当然也可以全部分区flush. 但是没那个必要。

```shell
# 笔者设备为Pixel 5因此路径中会有redfin
cd out/target/product/redfin/system/framework
# 以root权限运行adb
adb root
# 使用将分区变为可读写
adb remount
# 将services.jar push 到 system特定分区
adb push services.jar /system/framework/
# 重启所有系统服务（也可以reboot但是可能会慢一点。）
adb shell stop && adb shell start
```



再调试就可以发现没问题了。全都能看了：）

![image-20250112222049907](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250112222049907.png)
