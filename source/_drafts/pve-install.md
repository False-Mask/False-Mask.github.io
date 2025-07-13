---
title: pve-install
tags:
cover:
---

# PVE虚拟机安装





# 概览



1. 下载pve镜像 https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso
2. 使用rufus创建U盘启动盘
3. 安装PVE





# 下载PVE镜像



> ISO镜像下载地址
>

https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso



略



# 创建U盘启动盘



略



# U盘引导启动



再此之前记得先关闭Secure Boot



如下为成功引导后，点击Install Proxmox VE安装PVE

> 选择Graphical

![IMG_20250704_230350](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/IMG_20250704_230350.jpg)



后续操作无脑下一步。





# 网络配置





> 由于设备是一个笔记本，因此只有一个无线网络，无有线网络环境。
>
> 需要单独配置无线网络环境

1. 使用手机USB联网

>  略
>
> 注意，如果家里的使用了OpenWrt, 手机先把网络断了，不然域名解析会有问题。

2. 配置网络连接

>  https://www.cnblogs.com/yyy413/p/18073909
>
>  只用借鉴USB网络配置。后续过程就不用看了，比较复杂

3. 安装联网工具

>  network-manager

``` shell
apt install network-manager
```

4. 配置网络

``` shell
nmtui
```

5. 连接网络



> 经过上述配置以后，PVE虚拟机才连上无线网络。
>
> 我们才可以通过Web后台进入PVE虚拟机进行配置操作。

默认情况下端口号为8006





