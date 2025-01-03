---
title: BCC工具安装
tags:
  - linux
  - bcc
date: 2025-01-03 23:59:24
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/overview.png
---






# BCC工具安装



pre：阅读本文章，最好具备eBPF的基础了解。知晓BPF，eBPF基础概念。

[BCC github地址](https://github.com/iovisor/bcc)





# pre



环境前提条件：

1.内核版本4.1以上

Linux kernel version 4.1 or newer is required

2.内核参数

```PlaintText
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
# [optional, for tc filters]
CONFIG_NET_CLS_BPF=m
# [optional, for tc actions]
CONFIG_NET_ACT_BPF=m
CONFIG_BPF_JIT=y
# [for Linux kernel versions 4.1 through 4.6]
CONFIG_HAVE_BPF_JIT=y
# [for Linux kernel versions 4.7 and later]
CONFIG_HAVE_EBPF_JIT=y
# [optional, for kprobes]
CONFIG_BPF_EVENTS=y
# Need kernel headers through /sys/kernel/kheaders.tar.xz
CONFIG_IKHEADERS=y
```

 内核配置文件路径：`/proc/config.gz` or `/boot/config-<kernel-version>`.





# 安装





## way1 安装二进制包



不同版本有不同的安装方式，具体可见

[Github Doc](https://github.com/iovisor/bcc/blob/master/INSTALL.md)

以Ubunut为例

```zsh
sudo apt-get install bpfcc-tools linux-headers-$(uname -r)
```



安装完成后能在`/sbin` (`/usr/sbin` in Ubuntu 18.04) 下看见很多的文件名称形如：*--bpfcc

如下工具是利用bcc编写的部分常用的工具集合。（**不是**bcc本体哈～）

```c
➜ False-Mask.github.io git:(main) ✗ ls /sbin/ | grep "\-bpfcc"
argdist-bpfcc
bashreadline-bpfcc
bindsnoop-bpfcc
biolatency-bpfcc
biolatpcts-bpfcc
biopattern-bpfcc
biosnoop-bpfcc
biotop-bpfcc
bitesize-bpfcc
bpflist-bpfcc
......
```



到这里bcc就安装完成了～



具体bcc内置的工具集合使用可见

[Github Docs](https://github.com/iovisor/bcc/blob/master/docs/tutorial.md)



##  way2 



略

自己编译安装

[Github Docs](https://github.com/iovisor/bcc/blob/master/INSTALL.md#source)





# Hello World

[文档参考](https://github.com/iovisor/bcc/blob/master/docs/tutorial_bcc_python_developer.md)



## 下载源代码



首先声明安装源代码不是为了编译。官方的hello world就在源码里面。

多周到，连hello world都帮忙写好了，只管运行～



```shell
git clone git@github.com:iovisor/bcc.git
cd bcc/examples # demos路径，后续默认认为在该路径运行代码
```



## 运行



报错了！

```shell
➜  examples git:(master) python3 hello_world.py
Traceback (most recent call last):
  File "/home/rose/code/gitcode/bcc/examples/hello_world.py", line 9, in <module>
    from bcc import BPF
ModuleNotFoundError: No module named 'bcc'

```



别急看一下python路径(很明显不在系统路径下，所以bpfcc-tools失效了，因为我下了asdf所以python被链接到到了asdf内部的python)

```shell
➜  examples git:(master) which python3
/home/linuxbrew/.linuxbrew/bin/python3
```



解决方法也很简单sudo一下就好了。

```
sudo python3 hello_world.py
```



> 代码成功run起来了，效果是每创建一个进程就会打印一此hello world.
>
> 这很hello world！

```shell
➜  examples git:(master) sudo python3 hello_world.py

b'           <...>-10013   [019] ...21  6449.556159: bpf_trace_printk: Hello, World!'
b''
b'           <...>-9994    [008] ...21  6450.583176: bpf_trace_printk: Hello, World!'
b''
b' ThreadPoolForeg-9994    [008] ...21  6450.583254: bpf_trace_printk: Hello, World!'
b''
b'           <...>-10028   [001] ...21  6450.583302: bpf_trace_printk: Hello, World!'
b''
b' ThreadPoolForeg-9994    [008] ...21  6450.583311: bpf_trace_printk: Hello, World!'
b''
b' ThreadPoolForeg-10119   [005] ...21  6450.583512: bpf_trace_printk: Hello, World!'
b''
b' ThreadPoolForeg-10121   [002] ...21  6450.583517: bpf_trace_printk: Hello, World!'
b''
b' ThreadPoolForeg-10121   [006] ...21  6450.583786: bpf_trace_printk: Hello, World!'
b''
b' ThreadPoolForeg-10064   [001] ...21  6453.864021: bpf_trace_printk: Hello, World!'
b''
b' ThreadPoolForeg-10065   [016] ...21  6453.864045: bpf_trace_printk: Hello, World!'
b''
b' ThreadPoolForeg-10065   [016] ...21  6453.864115: bpf_trace_printk: Hello, World!'
b''
b' ThreadPoolForeg-10066   [020] ...21  6453.864148: bpf_trace_printk: Hello, World!'
b''

```





## 解析



> 怎么样？没想到把？代码这么短！

```python
# 导包，apt bpfcc-tools已经安装过了
from bcc import BPF 
# 用指定的源代码创建一个bpf程序。（源代码是C语言）
# 并通过trace_print不断读取输出结果
BPF(text='int kprobe__sys_clone(void *ctx) { bpf_trace_printk("Hello, World!\\n"); return 0; }').trace_print()
```



> 我们来分析下核心的东西BPF 程序
>
> 1.函数名中是包含一些信息的：kprobe表明使用kernel kprobe，sys_clone表面hook的内核代码为sys_clone
>
> 2.void *ctx 函数参数
>
> 3.bpf_trace_printk 一个内核工具方法将输出打印到/sys/kernel/debug/tracing/trace_pipe
>
> 4.return 0； 必须的代码，否则会有性能问题。具体我也不太理解，简单来说就是kprobe会依据返回值决定后一步的行为。不返回或者乱返回都可能导致Kprobe运行异常，具体[Return 0；][https://github.com/iovisor/bcc/issues/139]

```c
int kprobe__sys_clone(void *ctx) { 
    bpf_trace_printk("Hello, World!\\n"); 
    return 0;
}
```





# End



小小总结一下



1.BCC本质上是一个工具包用于帮助我们更轻松地书写eBPF程序

2.BCC工程下包含两个东西

​    a.一些用BCC实现的工具包execsnoop，opensnoop，biotop......

​    b.BCC工程，BCC本身，用于和eBPF交互，编译运行、读取eBPF程序

3.书写BCC程序其实分为两步

​    a.书写Python代码（这个Python代码只是一个脚本，这个脚本只是用于加载eBPF程序）

​    b.用C语言书写eBPF程序，这个会实打实地在内核执行。



