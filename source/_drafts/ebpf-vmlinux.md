---
title: ebpf-vmlinux
tags:
cover:
---



# eBPF vmlinux 文件







# 基础概念

1. vmlinux 文件是什么

vmlinux 文件是一个 ELF 文件，可以理解就是 linux 内核

> 在linux系统中，**vmlinux**（**vmlinuz**）是一个包含linux kernel的静态链接的可执行文件，文件类型可能是linux接受的可执行文件格式之一（[ELF](https://zh.wikipedia.org/wiki/可執行與可鏈接格式)、[COFF](https://zh.wikipedia.org/wiki/COFF)或[a.out](https://zh.wikipedia.org/wiki/A.out)），vmlinux若要用于调试时则必须要在开机前增加symbol table。
>
> ——[From Wikipedia](https://zh.wikipedia.org/wiki/Vmlinux)

2. vmlinux.h 是什么

vmlinux.h 是使用工具为 vmlinux 内核文件生成的的所有类型定义文件。

vmlinux.h 部分内容输出

![image-20250125004025877](../../../../../Library/Application Support/typora-user-images/image-20250125004025877.png)

大体的逻辑如下

```c
// 如果没有 define
#ifndef __VMLINUX_H__
#define __VMLINUX_H__

// 定义结构体
struct ... {}
struct ... {}
struct ... {}
struct ... {}
......


// 结束条件定义。  
#endif /* __VMLINUX_H__ */
```





refs：https://www.ebpf.top/post/intro_vmlinux_h/



# 开发























