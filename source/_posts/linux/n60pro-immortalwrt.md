---
title: N60Pro刷入ImmortalWrt
tags:
  - openwrt
cover: >-
  https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/Screenshot from
  2025-07-04 21-41-23.png
date: 2025-07-04 21:42:09
---


# N60Pro刷入ImmortalWrt



# pre0



1.刷机最好使用Windows客户端(作者尝试过Linux, Mac均失败。)

2.本文所有文件均使用的是ImmortalWrt的原装软件包(过程可能比较麻烦，可以使用恩山社区大佬闭源的uboot)

3.请在阅读完全文流程后刷入ImmortalWrt

4.务必做好原装分区的备份！！！务必！务必！务必！



# 概览



1.基础环境配置

> 根据官方手册配置好网络环境，打开ssh

2.备份

> 将N60Pro的所有分区进行备份

3,刷入uboot

> 通过官方镜像下载uboot

4.刷入immortalWrt镜像



# 基础环境配置

see https://mao.fan/article/333

略





# 备份分区



1.查看网关地址(192.168.100.1)

``` zsh
~ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.100.1   0.0.0.0         UG    600    0        0 wlp6s0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.94.0    0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-01
192.168.94.4    0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-02
192.168.94.8    0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-03
192.168.94.12   0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-04
192.168.94.16   0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-05
192.168.94.20   0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-06
192.168.94.24   0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-07
192.168.94.28   0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-08
192.168.94.32   0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-09
192.168.94.36   0.0.0.0         255.255.255.252 U     0      0        0 cvd-wifiap-10
192.168.96.0    0.0.0.0         255.255.255.0   U     0      0        0 cvd-wbr
192.168.97.0    0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-01
192.168.97.4    0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-02
192.168.97.8    0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-03
192.168.97.12   0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-04
192.168.97.16   0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-05
192.168.97.20   0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-06
192.168.97.24   0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-07
192.168.97.28   0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-08
192.168.97.32   0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-09
192.168.97.36   0.0.0.0         255.255.255.252 U     0      0        0 cvd-mtap-10
192.168.98.0    0.0.0.0         255.255.255.0   U     0      0        0 cvd-ebr
192.168.100.0   0.0.0.0         255.255.255.0   U     600    0        0 wlp6s0
```



2.ssh连接（注意用户名必须是useradmin）

``` zsh
ssh useradmin@192.168.100.1
useradmin@192.168.100.1's password: (此处输入管理员密码)
ash: od: not found
  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt 21.02-SNAPSHOT, r0-996d48103
 -----------------------------------------------------


BusyBox v1.33.2 (2024-10-14 01:32:52 UTC) built-in shell (ash)

useradmin@N60PRO:~# 
```



3.查看分区表

``` zsh
useradmin@N60PRO:~# cat /proc/mtd 
dev:    size   erasesize  name
mtd0: 08000000 00020000 "spi0.1"
mtd1: 00100000 00020000 "BL2"
mtd2: 00080000 00020000 "u-boot-env"
mtd3: 00200000 00020000 "Factory"
mtd4: 00200000 00020000 "FIP"
mtd5: 07280000 00020000 "ubi"
```



4.备份分区表（因为内存不够，先备份后5个）

``` zsh
~ dd if=/dev/mtd1 of=/tmp/mtd1
~ dd if=/dev/mtd2 of=/tmp/mtd2
~ dd if=/dev/mtd3 of=/tmp/mtd3
~ dd if=/dev/mtd4 of=/tmp/mtd4
~ dd if=/dev/mtd5 of=/tmp/mtd5
```



5.ssh拷贝(scp拷贝不下来。笑死......)

``` 
ssh useradmin@192.168.100.1 cat /tmp/mtd1 > /tmp/mtd1
ssh useradmin@192.168.100.1 cat /tmp/mtd2 > /tmp/mtd2
ssh useradmin@192.168.100.1 cat /tmp/mtd3 > /tmp/mtd3
ssh useradmin@192.168.100.1 cat /tmp/mtd4 > /tmp/mtd4
ssh useradmin@192.168.100.1 cat /tmp/mtd5 > /tmp/mtd5
```



6.计算md5sum对比下,（ssh客户端）

``` zsh
~ md5sum /tmp/mtd*
500c976ec5210b165c004613a69abd5a  /tmp/mtd1
38eb11b49b828dc070270680a8a37845  /tmp/mtd2
73aa9442c7fb65b300913d976adfcd74  /tmp/mtd3
94ba89b4b28d31ec36611d97c3a0d434  /tmp/mtd4
624ce2a5686e843a9474efca35465c3f  /tmp/mtd5
```



7.n60Pro的内容（完美match无问题）

``` zsh
useradmin@N60PRO:~# md5sum /tmp/mtd*
500c976ec5210b165c004613a69abd5a  /tmp/mtd1
38eb11b49b828dc070270680a8a37845  /tmp/mtd2
73aa9442c7fb65b300913d976adfcd74  /tmp/mtd3
94ba89b4b28d31ec36611d97c3a0d434  /tmp/mtd4
624ce2a5686e843a9474efca35465c3f  /tmp/mtd5
```



8.删除n60Pro已经备份的内容（腾空间）

``` zsh
 rm /tmp/mtd*
```



9.ssh拷贝第一个分区

``` zsh
~ dd if=/dev/mtd0 of=/tmp/mtd0
```



10.ssh备份

``` zsh
ssh useradmin@192.168.100.1 cat /tmp/mtd0 > /tmp/mtd0
```



11.验证md5(通过)

``` zsh
~ md5sum /tmp/mtd0 (本地)
9f6145d1304d822bf935257af026d074  /tmp/mtd0

~ useradmin@N60PRO:~# md5sum /tmp/mtd0 (N60Pro)
9f6145d1304d822bf935257af026d074  /tmp/mtd0
```



# 下载 & 刷入引导镜像



1.下载uboot

https://firmware-selector.immortalwrt.org/



> 笔者选择
>
> Netcore N60 Pro 
>
> 24.10.2

https://downloads.immortalwrt.org/releases/24.10.2/targets/mediatek/filogic/immortalwrt-24.10.2-mediatek-filogic-netcore_n60-pro-bl31-uboot.fip



2.计算sha256

``` zsh
~ sha256sum immortalwrt-24.10.2-mediatek-filogic-netcore_n60-pro-bl31-uboot.fip 
c8e27751545de6e228d168dd64aa2c2fa7da04c1a2424207c468c8dc549433ef  immortalwrt-24.10.2-mediatek-filogic-netcore_n60-pro-bl31-uboot.fip
```



3.push到N60Pro (别问我为不用scp，还不是会报错。)

``` zsh
cat immortalwrt-24.10.2-mediatek-filogic-netcore_n60-pro-bl31-uboot.fip | ssh useradmin@192.168.100.1 "cat - > /tmp/uboot.fip"
```



4.再次计算sha256 （满足）

```zsh
useradmin@N60PRO:~# sha256sum /tmp/uboot.fip 
c8e27751545de6e228d168dd64aa2c2fa7da04c1a2424207c468c8dc549433ef  /tmp/uboot.fip
```



5.使用uboot刷入

``` zsh
~ useradmin@N60PRO:~# mtd write /tmp/uboot.fip FIP
Unlocking FIP ...

Writing from /tmp/uboot.fip to FIP ...     
```



6.删除ubi分区

``` zsh
~ useradmin@N60PRO:~# mtd erase ubi
Unlocking ubi ...
Erasing ubi ...
useradmin@N60PRO:~# 
```



7.重启

``` zsh
~ reboot 
```



# 刷入recovery



> Note：这一步笔者做过很多尝试。
>
> Linux，Mac均无法成功刷入
>
> 目前只有Windows可以成功



1. 通过网线连接路由器，并将当前设备的ip设置为192.168.1.254, 网关设置为192.168.1.1
2. 下载tftpd64
3. 将路径修改为immortal-*.itb文件的路径*
4. *修改immortal-*.itb文件名称(重要)

> 具体文件名称可见Log Viewer日志输出



> 这里简单讲一下：
>
> 刷入uboot以后路由器会检索所有ip为192.168.1.254的相邻设备的文件。
>
> 而tftpd64相当于是将文件提供给N60Pro路由器。
>
> 再强调一遍！！！这一步的刷入recovery是N60Pro主动向主机请求文件。因此只有Windows客户端的tftpd64可以成功实现这一步。





# 刷入update镜像



> 进入ImmortalWrt (192.168.1.1)

选择文件上传sysupdate文件。

