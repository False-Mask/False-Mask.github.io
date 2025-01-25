---
title: android-vscode
cover: 'https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250125172843126.png'
date: 2025-01-25 17:50:47
tags:
- env
---




# Android Vscode Remote Develop环境搭建



# 背景



> 探索 Android 上运行 eBPF 的环境，需要在 Android 上执行 vscode server 进行。
>
> 使用 SSH 进行代码的远程开发以及运行。



# 实现



在 Android 上执行一个 chroot，并在 chroot 中执行 sshd，最后通过 vscode remote ssh 插件进行vscode server 的配置。





# 过程



## pre0



1.手机 root

2.安装 Magisk-SSH 模块开启 ssh（如果没有 ssh，eadb 无法连接到手机上）



## chroot配置



> 下载 eadb ，使用 eadb 安装 debian 11 rootfs

Refs: https://github.com/tiann/eadb

1.安装eadb

```shell
# 安装 rust
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
# 安装eadb
cargo install eadb
```

2.构建debian 11 rootfs

```shell
sudo eadb build
```

3.配置debian rootfs

```shell
eadb --ssh root@ip prepare -a path/to/archive
```





## chroot ssh



### pre0 apt配置



> debian rootfs由于只做了最简单的配置，apt是存在一些问题，有些包是无法安装的

```plaintText
W: Download is performed unsandboxed as root as file '/var/lib/apt/lists/partial/ftp.us.debian.org_debian_dists_bullseye_InRelease' couldn't be accessed by user '_apt'. - pkgAcquire::Run (13: Permission denied)
```

> 这个警告信息表明，APT 在以 root 用户身份下载文件时，无法让 `_apt` 用户访问某些目录或文件。以下是解决这个问题的步骤：

- 创建 `_apt` 用户和组：

  ```bash
  groupadd _apt # 添加_apt group
  useradd -g _apt -d /nonexistent -s /usr/sbin/nologin _apt 
  
  sudo chown -R _apt:_apt /var/lib/apt/lists/partial/
  sudo chmod 700 /var/lib/apt/lists/partial/
  # sudo: 以超级用户权限执行命令。
  # useradd: 创建新用户的命令。
  # -g _apt: 指定用户所属的初始组为 _apt。
  # -d /nonexistent: 指定用户的主目录为 /nonexistent，表示该用户没有实际的主目录。
  # -s /usr/sbin/nologin: 指定用户的登录shell为 /usr/sbin/nologin，防止用户直接登录。
  # _apt: 新用户的用户名。
  
  ```

- 设置正确的权限

```shell
sudo chown -R _apt:_apt /var/lib/apt/lists/partial/
sudo chmod 700 /var/lib/apt/lists/partial/
```

- 安装sudo(针对于chroot环境约等于什么都不做，只是很多时候会习惯使用sudo，但是报错很不舒服。)

```shell
apt install sudo
```





### 软件环境安装



1.安装ssh

```shell
sudo apt install ssh
```

2.配置ssh端口号

```shell
vim /etc/ssh/sshd_config
```

3.启动sshd

> 很奇怪？
>
> 不是已经配置了sshd吗？难道systemd在chroot的时候不会自动启动sshd？没错还真是这样(･o･)ﾉYes!!
>
> 因为chroot的隔离性比较差，所以systemd是没有单独适配的，所以依赖systemd启动的服务并不会正常启动。
>
> 而sshd正是这样的服务。
>
> 我们需要自己手动调用，这样才有ssh链接到。

```shell
/usr/sbin/sshd -f /etc/ssh/sshd_config
```



## vscode链接配置



1.安装Remote SSH插件

![image-20250125171918274](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250125171918274.png)

2.配置 config 文件



## 最终效果图



![image-20250125172843126](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250125172843126.png)



## 其他



### 网络环境配置





Way1：

安装 clash meta，开启 tun 代理

Way2：

完整 clash meta，通过 adb reverse 代理给主机，通过主机的 clash 节点开启网络。



### QA



1. 为什么不直接 SSH 连接主机而是连接 chroot

> 最开始也是考虑过的，只不过由于android环境的差异，vscode server 直接崩溃了。
>
> ```plaintText
> [2025-01-25 09:32:20] error This machine does not meet Visual Studio Code Server's prerequisites, expected either...
> 
>  \- find libstdc++.so or ldconfig for GNU environments
> 
>  \- find /lib/ld-musl-aarch64.so.1, which is required to run the Visual Studio Code Server in musl environments
> ```
>
> 没办法了只能使用 chroot 了

![image-20250125173722641](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250125173722641.png)
