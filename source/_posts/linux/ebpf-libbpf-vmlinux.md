---
title: Android eBPF Demo 书写
tags:
  - linux
  - ebpf
cover: >-
  https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/diagram-b6b32006ea52570dc6773f5dbf9ef8dc.svg
date: 2025-02-03 11:43:52
---


# Android libbpf Demo 书写

![diagram-b6b32006ea52570dc6773f5dbf9ef8dc](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/diagram-b6b32006ea52570dc6773f5dbf9ef8dc.svg)

——From ebpf.io

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
//vmlinux.h

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



# Demo开发



refs：https://blog.csdn.net/weixin_45092290/article/details/138767704



## 前置准备

```shell
# 安装常规编译工具链
apt install clang clangd lldb cmake
# 安装libbpf-dev（安装 bpf/bpf_helpers.h 等头文件）
apt install libbpf-dev
# 提取vmlinux.h头文件
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```



## 内核Demo程序编写



clang -target bpf -I. -c ebpf-demo.bpf.c -o ebpf-demo.bpf.o

// ebpf-demo.bpf.c

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>

SEC("kprobe/__arm64_sys_openat")
int bpf_prog(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 *count, init_val = 1;

    bpf_printk("opennat calced"); 

    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```



## Skel生成

bpftool gen skeleton ebpf-demo.bpf.o > ebpf-demo.skel.h



## 用户态Demo程序



clang -Wall -I. -c ebpf-demo.c -o ebpf-demo.o

// ebpf-demo.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include "ebpf-demo.skel.h"

static int libbpf_print_fn(enum libbpf_print_level level, const char * format, va_list args){
    return vfprintf(stderr, format, args);
}

static void  bump_memlock_rlimit() {

    struct rlimit rlimt_new = {
        .rlim_cur = RLIM_INFINITY,
        .rlim_max = RLIM_INFINITY,
    };

    if (setrlimit(RLIMIT_MEMLOCK, &rlimt_new)) {
        fprintf(stderr, "Fail to increate the limit!");
        exit(1);    
    }

}

int main() {

    struct ebpf_demo_bpf* skel;
    int err;
    libbpf_set_print(libbpf_print_fn);

    bump_memlock_rlimit();

    skel = ebpf_demo_bpf__open_and_load();

    if (!skel) {
        fprintf(stderr,"Fail to open bpf");
    }

    err = ebpf_demo_bpf__attach(skel);

    if (err) {
        fprintf(stderr, "Fail to load %d", err);
        goto cleanup;
    }

    fprintf(stderr, "Successfully started! see output by cat /sys/kernel/tracing/trace_pipe\n");

    for(;;) {
        fprintf(stderr, ".");
        sleep(1);
    }



    cleanup:
    ebpf_demo_bpf__destroy(skel);
    return -err;

}
```





## 编译总程序



compile.sh

```shell
#!/bin/zsh
rm ebpf-demo ebpf-demo.bpf.o ebpf-demo.skel.h
clang -O2 -target bpf -I. -c ebpf-demo.bpf.c -o ebpf-demo.bpf.o
bpftool gen skeleton ebpf-demo.bpf.o > ebpf-demo.skel.h
clang -Wall -I. -c ebpf-demo.c -o ebpf-demo.o
clang -Wall ebpf-demo.o -L/usr/lib64 -lbpf -lelf -lz -o ebpf-demo
```



## 运行程序



```shell
➜  code ./ebpf-demo
libbpf: loading object 'ebpf_demo_bpf' from buffer
libbpf: elf: section(3) kprobe/__arm64_sys_openat, size 72, link 0, flags 6, type=1
libbpf: sec 'kprobe/__arm64_sys_openat': found program 'bpf_prog' at insn offset 0 (0 bytes), code size 9 insns (72 bytes)
libbpf: elf: section(4) license, size 4, link 0, flags 3, type=1
libbpf: license of ebpf_demo_bpf is GPL
libbpf: elf: section(6) .symtab, size 96, link 1, flags 0, type=2
libbpf: looking for externs among 4 symbols...
libbpf: collected 0 externs total
libbpf: object 'ebpf_demo_bpf': failed (-22) to create BPF token from '/sys/fs/bpf', skipping optional step...
Successfully started! see output by cat /sys/kernel/tracing/trace_pipe
.......
```



> 你可能会疑惑，输出呢？
>
> 由于我们内核程序使用的bpf_printk。
>
> 所以输出会被重定向到/sys/kernel/debug/tracing/trace_pipe文件
>
> 直接` cat /sys/kernel/tracing/trace_pip `即可。



> 你可能还有疑惑。
>
> 因为你cat /sys/kernel/tracing/trace_pipe 的输出结果很可能是空的。
>
> 那是因为你没有开启 trace
>
> 执行如下 shell 脚本
>
> `echo 1 > /sys/kernel/tracing/tracing_on`





# 总结



1.eBPF 的程序其实包含两个部分

​	a. 内核态程序（这个程序会在内核虚拟机内执行）

​	b. 用户态程序——前端（这个程序会用于加载并与内核态程序通讯，虽然 Demo 中没有体现通讯这一过程，但是他是可以实现的）

2.libbpf 程序

​	a. 最终产物只有一个 elf 文件，这个 elf 内包含用户态程序 & 内核态程序的二进制数据。

​	b. libbpf 编译过程中会生成 skel 文件，这个文件内包含一些 eBPF 生命周期模板方法。

​	c. libbpf 以及其他 eBPF 脚手架一样，都只是 eBPF 前端（后端在内核态上运行）

> eBPF 的脚手架特别多，libbpf 只是其中一种。
>
> 常见的有
>
> - libbpf——C/C++
> - [BCC](https://github.com/iovisor/bcc) ——Python
> - [Cilium](https://github.com/cilium/cilium) ——Go
> - [aya](https://github.com/aya-rs/aya)——Rust
> - [bpftrace](https://github.com/bpftrace/bpftrace) —— bpftrace 规则语法（把他当成语言可能有些牵强～）
>
> 所以没必要纠结需要使用哪种，哪种顺手就用哪种～















