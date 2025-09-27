---
title: aosp-raspberry-15
tags:
cover:
---

# 树莓派AOSP刷机



# 准备





## 源码下载

 

https://github.com/raspberry-vanilla/android_local_manifest/tree/android-15.0



1.配置repo

```shell
# 配置远程仓库
repo init -u https://mirrors.ustc.edu.cn/aosp/platform/manifest.git -b android-15.0.0_r32
# 下载树莓派的manifset文件
curl -o .repo/local_manifests/manifest_brcm_rpi.xml -L https://raw.githubusercontent.com/raspberry-vanilla/android_local_manifest/android-15.0/manifest_brcm_rpi.xml --create-dirs
```

2.同步

```shell
repo sync
```

3.配置构建环境

``` shell
. build/envsetup.sh
```

4.启动

```shell
lunch aosp_rpi5-bp1a-userdebug
```

5.执行编译

```shell
make bootimage systemimage vendorimage -j$(nproc)
```

