---
title: pve-install
tags:
cover:
---

# PVE虚拟机安装



# 概览



1. 下载pve镜像 https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso
2. 使用rufus创建U盘启动盘
3. 





# 下载PVE镜像



> ISO镜像下载地址
>
> （最好下载8.4的版本，有GUI）

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



> 由于设备是一个笔记本，因此只有一个万线网络，无有线网络环境。
>
> 需要单独配置网络环境

1. 使用手机USB联网

>  略
>
> 注意，如果家里的使用了OpenWrt, 手机先把网络断了，不然域名解析会有问题。

2. 配置网络连接

>  https://www.cnblogs.com/yyy413/p/18073909

3. 安装联网工具

>  wireless-tools，wpasupplicant

4. 启动网卡

``` shell
ifup 网卡名称
```

5. 使用ssid和password生成密钥

```
wpa_passphrase WiFi的SSID WiFi密码
```

6. 配置网络

``` shell
# nano /etc/network/interfaces

# 静态IP
auto 网络接口名称
iface 网络接口名称 inet static
    address 192.168.1.3/24
    gateway 192.168.1.1
    wpa-ssid xxxxxx
    wpa-psk xxxxxxxxxxxxx 

# dhcp自动分配ip

auto 网络接口名称
iface 网络接口名称 inet dhcp
    wpa-ssid xxxxxx
    wpa-psk xxxxxxxxxxxxxx
```

7. restart network

``` shell
systemctl restart network
```

