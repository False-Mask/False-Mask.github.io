---
title: Android ebpf环境搭建
tags:
  - android
  - linux
  - ebpf
  - env
date: 2025-07-13 22:37:45
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/overview.png
---




# Android ebpf环境搭建

主要用于配置ebpf的运行环境，开发环境不包含在内。



# 前言



- 关于本文：

  本文主要用于介绍Android中如何搭建ebpf学习环境，难度较大/不断更新完善中。

- 前置准备：

  a.设备最好要有内核源代码，因为涉及到内核重新配置/编译。如果没有则内核必须得是GKI内核。

  b.设备需要解锁BL, 因为涉及Boot img刷入

  c.设备内核最好在5.10及其以上的版本，因为ebpf的特性和内核版本绑定。

  d.需要编译好的Linux rootfs，后续需要在Android上使用chroot运行rootfs

  



# 概览



1. 编译自定义内核并刷入

> Note：
>
> 别怕这里只是简单的改了几个内核参数。
>
> 改动不大，就是流程比较长。

2. 安装bpftrace
3. 安装BCC







# 内核配置准备



1. BCC内核配置

a.[BCC需要的内核配置列表](https://github.com/False-Mask/bcc/blob/master/INSTALL.md#kernel-configuration)

b.[Ftrace内核配置参数](https://source.android.com/docs/core/tests/debug/ftrace)——具体过程可见下方[内核开启ftrace](##内核开启Dynamic FTrace)





# 安装bpftrace

``` zsh
apt install bpftrace
```





# 安装bcc



> Note：
>
> 该过程必须使用源码编译，直接下载的bcc在Android上不能直接用。
>
> 因为Android上修改了tracefs的路径，需要进行修正才能使用。



## 源码修改



> 由于bcc的代码中的tracefs路径是写死的，并且和android kernel的tracefs文件路径
>
> 不一样，因此需要将代码clone下来修改下路径。

repace '/sys/kernel/debug/tracing/' '/sys/kernel/tracing/'



修改后的仓库地址：https://github.com/False-Mask/bcc.git 

分支名称：android-last-release



## 编译前rootfs环境配置

> 该阶段的代码在手机chroot环境中运行rootfs。

- 下载git

``` bash
apt install git
```

- 下载bcc

``` shell
git clone https://github.com/False-Mask/bcc.git -b v0.35.0 --depth 1
```

> Note：如果是自己改的源码，一定要注意，打tag否则，bcc打包的时候会读取commit作为python的版本号，这会导致构建错误，报错如下。
>
> ``` python
> Traceback (most recent call last):
>   File "/tmp/bcc/build/src/python/bcc-python3/setup.py", line 11, in <module>
>     setup(name='bcc',
>     ~~~~~^^^^^^^^^^^^
>           version='128-NOTFOUND+352d71be',
>           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
>     ...<4 lines>...
>           packages=['bcc'],
>           ^^^^^^^^^^^^^^^^^
>           platforms=['Linux'])
>           ^^^^^^^^^^^^^^^^^^^^
>   File "/usr/lib/python3/dist-packages/setuptools/__init__.py", line 117, in setup
>     return distutils.core.setup(**attrs)
>            ~~~~~~~~~~~~~~~~~~~~^^^^^^^^^
>   File "/usr/lib/python3/dist-packages/setuptools/_distutils/core.py", line 148, in setup
>     _setup_distribution = dist = klass(attrs)
>                                  ~~~~~^^^^^^^
>   File "/usr/lib/python3/dist-packages/setuptools/dist.py", line 334, in __init__
>     self.metadata.version = self._normalize_version(self.metadata.version)
>                             ~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^
>   File "/usr/lib/python3/dist-packages/setuptools/dist.py", line 370, in _normalize_version
>     normalized = str(Version(version))
>                      ~~~~~~~^^^^^^^^^
>   File "/usr/lib/python3/dist-packages/setuptools/_vendor/packaging/version.py", line 202, in __init__
>     raise InvalidVersion(f"Invalid version: {version!r}")
> packaging.version.InvalidVersion: Invalid version: '128-NOTFOUND+352d71be'
> ```
>

- 配置sid镜像源

https://mirrors.tuna.tsinghua.edu.cn/help/debian/

- 安装build依赖

``` bash
apt-get install arping bison clang-format cmake dh-python \
  dpkg-dev pkg-kde-tools ethtool flex inetutils-ping iperf \
  libbpf-dev libclang-dev libclang-cpp-dev libedit-dev libelf-dev \
  libfl-dev libzip-dev linux-libc-dev llvm-dev libluajit-5.1-dev \
  luajit python3-netaddr python3-pyroute2 python3-setuptools python3 \
  zip libpolly-19-dev
```



## 构建



> Note：需要在rootfs上运行

> 参考bcc官方文档进行代码构建

```bash
mkdir bcc/build; cd bcc/build
cmake ..
make
make install
```





# others



## 内核开启Dynamic FTrace



[Use ftrace —— Android官方文档](https://source.android.com/docs/core/tests/debug/ftrace)



流程：

> 对于内核我也不确定，我只是踩了很多坑，最早发现这样能跑起来 =.=

1. 修改aosp kernel

``` 
// 文件路径 aosp/arch/arm64/configs/gki_defconfig

CONFIG_TRACEFS_DISABLE_AUTOMOUNT=y
// 新增
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_PROFILER=y
CONFIG_IRQSOFF_TRACER=y
CONFIG_PREEMPT_TRACER=y
// ......
```

2. 修改Pixel Kernel

``` 
// private/gs-google/arch/arm64/configs/gki_defconfig

CONFIG_TRACEFS_DISABLE_AUTOMOUNT=y
// 新增
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_PROFILER=y
CONFIG_IRQSOFF_TRACER=y
CONFIG_PREEMPT_TRACER=y
```

3.update symbol list（不确定这个是不是要执行，但是我好像执行了这个就能跑起来了）

```
./update_symbol_list_slider-aosp.sh
```

4.编译内核

```
./build_slider.sh
```





# 引用



[BCC官方文档](https://github.com/False-Mask/bcc/blob/master/INSTALL.md#debian---source)





