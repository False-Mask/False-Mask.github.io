---
title: 编译创建Debian rootfs并在Android 上运行
tags:
  - linux
  - debian
cover: 'https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/Debian-logo.png'
date: 2025-07-05 11:04:25
---



# 编译创建Debian rootfs并在Android 上运行

refs：

[博客1](https://www.liuwantong.com/2021/02/16/debian-rootfs/)

[博客2](https://ivonblog.com/posts/debootstrap-create-rootfs-for-android/)

[Debian Wiki](https://wiki.debian.org/zh_CN/Debootstrap)



# 背景



> Android ebpf环境搭建
>
> 编译构建可在Android上运行的Debian rootfs环境





# 整体思路



其实通过上述的refs很容易能看出来rootfs的安装流程很简单。通过debootstrap，下载安装到制定路径。

下载安装完成以后就可以通过chroot check是否生效。





# 确认Debian版本



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

**详细解释**

**`debootstrap`**：

- Debian 官方工具，用于在 **非 Debian 系统**（如 Ubuntu、其他 Linux）或 **空目录** 中创建一个最小化的 Debian 根文件系统。
- 常用于构建 **Docker 镜像**、**嵌入式系统** 或 **chroot 环境**。

**`--arch arm64`**：

- 指定目标系统的 CPU 架构为

   

  ARM64

  （64 位 ARM 处理器），适用于：

  - 树莓派 3B+/4B
  - ARM 服务器（如 AWS Graviton）
  - 嵌入式设备（如 NAS、路由器）

**`--components=main,universe`**：

- **`main`**：Debian 官方维护的开源软件包（默认包含）。
- **`universe`**：社区维护的开源软件包（注意：`universe` 是 **Ubuntu 仓库**的命名，Debian 对应的是 **`contrib` 和 `non-free`**，此处可能需更正为 `--components=main,contrib,non-free`）。
- 若需更广泛软件支持，应使用 Debian 的标准组件名。

**`bullseye`**：

- Debian 11 的代号（发布于 2021 年 8 月），是当前 **稳定版（Stable）**。
- 其他版本代号：`bookworm`（Debian 12，测试版）、`sid`（不稳定版）。

**`my-debian`**：

- 生成的根文件系统将保存在当前目录下的 `my-debian` 文件夹中。
- 完成后可通过 `chroot my-debian` 进入该环境。

**`http://ftp.cn.debian.org/debian/`**：

- 使用 **中国区镜像源**（位于北京外国语大学），加快国内下载速度。
- 其他可选镜像：
  - 官方源：`http://deb.debian.org/debian/`
  - 腾讯云：`http://mirrors.tencent.com/debian/`
  - 阿里云：`http://mirrors.aliyun.com/debian/`



> 日志输出

``` shell
sudo debootstrap --arch arm64 --components=main,universe bullseye  my-debian http://ftp.cn.debian.org/debian/
W: Cannot check Release signature; keyring file not available /usr/share/keyrings/debian-archive-keyring.gpg
......
I: Configuring whiptail...
I: Configuring procps...
I: Configuring libbpf0:arm64...
I: Configuring libnftables1:arm64...
I: Configuring nftables...
I: Configuring iproute2...
I: Configuring isc-dhcp-client...
I: Configuring ifupdown...
I: Configuring tasksel-data...
I: Configuring tasksel...
I: Configuring libc-bin...
I: Base system installed successfully.
```



4. 打包成tar包

```shell
sudo tar -cvf debian-mini.tar my-debian
```



5. push到客户端上

``` zsh
adb push debian-mini.tar /data/local/tmp
```



6. 解压

``` shell
tar -xvf debian-mini.tar
mv my-debian debian11
```



# 修改配置

> 后续操作均是在手机上运行

1. 添加mount文件路径

> 参考eadb
>
> 简单来说就是挂载proc，dev，sys，bpf，cgroup,debug,tracing文件

``` shell
do_mounts()
{
	mount --bind /proc debian11/proc/ > /dev/null

	mount --bind /dev debian11/dev/ > /dev/null
	mount --bind /dev/pts debian11/dev/pts > /dev/null

	mount --bind /sys debian11/sys/ > /dev/null
	mount --bind /sys/fs/bpf/ debian11/sys/fs/bpf/ > /dev/null
	mount --bind /sys/fs/cgroup/ debian11/sys/fs/cgroup/ > /dev/null
	mount --bind /sys/kernel/debug/ debian11/sys/kernel/debug/ > /dev/null

	mount --bind /sys/kernel/tracing/ debian11/sys/kernel/tracing/

	# some devices doesn't have a /sys/kernel/debug/tracing
	if [ -d /sys/kernel/debug/tracing ]; then
		mount --bind /sys/kernel/debug/tracing/ debian11/sys/kernel/debug/tracing/
	fi

	# Mount Android partitions
	if [ -d /d/ ]; then
		if [ ! -d debian11/d ]; then ln -s /sys/kernel/debug debian11/d; fi
	fi

	# don't mount /data to avoid data loss
	if [ -d /system/ ]; then
		mkdir -p debian11/system/
		mount --bind /system debian11/system/
	fi

	if [ -d /vendor/ ]; then
		mkdir -p debian11/vendor/
		mount --bind /vendor debian11/vendor/
	fi

	if [ -d /apex/ ]; then
		mkdir -p debian11/apex/
		mount --bind /apex debian11/apex/
		for f in /apex/*; do
			if [ -d $f ]; then
				mount --bind $f debian11/apex/$(basename $f)
			fi
		done
	fi

	if [ -d /dev/binderfs/ ]; then
		mkdir -p debian11/dev/binderfs
		mount --bind /dev/binderfs/ debian11/dev/binderfs
	fi

}

mount | grep debian11 > /dev/null
if [ $? -ne 0 ]; then do_mounts; fi

# unset any PRELOAD in Android environment, otherwise Linux linker cann't handle it.
unset LD_PRELOAD
```



2. 添加bashrc

> 参考eadb

``` shell
# vi debian11/.bashrc 记得添加bashrc文件
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/share/bcc/tools/
export TMPDIR=/tmp/
```



3. 运行

``` shell
source debian-mount.sh
chroot  debian11 /bin/bash --rcfile "/.bashrc"
```











