---
title: # 使用Docker构建Android Kernel
tags:
cover:
---



# 使用Docker构建Android Kernel



# 背景



> Ubuntu 24.04 glibc版本过高，Android 构建时会由于一个缺陷导致编译报错
>
> https://android.googlesource.com/kernel/common/+/75f82c6a15c4188cbb32825892fc6ae3e95479f0





# 流程



1.下载Docker镜像

2.安装Docker

3.开启容器构建kernel





# 安装Docker



refs：https://docs.docker.com/engine/install/ubuntu/

1.如果安装过老版本的需要先卸载

```shell
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

2.安装docker

```shell
# Add Docker's official GPG key:
# 下载docker签名证书
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
# 添加docker镜像源 & 更新 apt package list
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# 下载最新版docker
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

```

3.修复报错



refs：https://www.baeldung.com/linux/docker-permission-denied-daemon-socket-error

```shell
# 由于docker很多指令是以root权限运行的。所以直接运行会报错
# permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.47/images/search?term=ubuntu": dial unix /var/run/docker.sock: connect: permission denied
# 可以通过用户组解决这个问题

# 1. 修改docker.sock的权限
sudo chmod 666 /var/run/docker.sock
# 2. 添加用户组
sudo groupadd docker
sudo usermod -aG docker $USER
# 重新登陆
```



# 构建Kernel



1.安装Ubuntu22.04

```shell
docker pull ubuntu:22.04
```



2.映射文件

```shell
docker run -it --rm -v /home/user/my-project:/mnt/my-project ubuntu:22.04
```



3.启动容器

```shell
docker run -itd --name "ubuntu22.04-kernel-build" -v /Users/rose/kernel:/tmp/kernel ubuntu:22.04
docker attach ubuntu22.04-kernel-build
```



4.执行指令安装必备的环境

```shell
apt update
# 安装必备的环境
apt install -y rsync python3 build-essential
apt install -y linux-headers-$(uname -r)
apt install -y git
git config --global --add safe.directory /tmp/kernel/aosp
```

