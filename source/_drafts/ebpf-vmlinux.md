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



clang -g -target bpf -I. -c ebpf-demo.bpf.c -o ebpf-demo.bpf.o

// ebpf-demo.bpf.c

```c
#include "vmlinux.h"
// #include <uapi/linux/ptrace.h>
// #include <uapi/linux/limits.h>
// #include <linux/sched.h>
// #include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, u32);
    __type(value, u64);
    __uint(max_entries, 1024);
} open_count SEC(".maps");

SEC("kprobe/__arm64_sys_openat")
int bpf_prog(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 *count, init_val = 1;

    count = bpf_map_lookup_elem(&open_count, &pid);
    if (count) {
        __sync_fetch_and_add(count, 1);
    } else {
        bpf_map_update_elem(&open_count, &pid, &init_val, BPF_ANY);
    }

    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```



## Skel生成

bpftool gen skeleton ebpf-demo.bpf.o > ebpf-demo.skel.h



## 用户态Demo程序



clang -Wall -I. -c ebpf-demo.c -o -g ebpf-demo.o

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

    skel = ebpf_demo_bpf__open();

    if (!skel) {
        fprintf(stderr,"Fail to open bpf");
    }

    //skel->bss->my_pid = getpid();

    err = ebpf_demo_bpf__attach(skel);

    if (err) {
        fprintf(stderr, "Fail to load %d", err);
        goto cleanup;
    }

    printf("Successfully started! see output by cat /sys/kernel/debug/tracing/trace_pipe");

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
clang -g -target bpf -I. -c ebpf-demo.bpf.c -o ebpf-demo.bpf.o
bpftool gen skeleton ebpf-demo.bpf.o > ebpf-demo.skel.h
clang -Wall -I. -c ebpf-demo.c -o -g ebpf-demo.o
clang -Wall ebpf-demo.o -L/usr/lib64 -lbpf -lelf -lz -g -o ebpf-demo
```



## 运行程序



```shell
➜  code ./ebpf-demo 
libbpf: loading object 'ebpf_demo_bpf' from buffer
libbpf: elf: section(3) kprobe/__arm64_sys_openat, size 240, link 0, flags 6, type=1
libbpf: sec 'kprobe/__arm64_sys_openat': found program 'bpf_prog' at insn offset 0 (0 bytes), code size 30 insns (240 bytes)
libbpf: elf: section(4) .relkprobe/__arm64_sys_openat, size 32, link 25, flags 40, type=9
libbpf: elf: section(5) .maps, size 32, link 0, flags 3, type=1
libbpf: elf: section(6) license, size 4, link 0, flags 3, type=1
libbpf: license of ebpf_demo_bpf is GPL
libbpf: elf: section(15) .BTF, size 1497, link 0, flags 0, type=1
libbpf: elf: section(17) .BTF.ext, size 288, link 0, flags 0, type=1
libbpf: elf: section(25) .symtab, size 312, link 1, flags 0, type=2
libbpf: looking for externs among 13 symbols...
libbpf: collected 0 externs total
libbpf: map 'open_count': at sec_idx 5, offset 0.
libbpf: map 'open_count': found type = 1.
libbpf: map 'open_count': found key [6], sz = 4.
libbpf: map 'open_count': found value [10], sz = 8.
libbpf: map 'open_count': found max_entries = 1024.
libbpf: sec '.relkprobe/__arm64_sys_openat': collecting relocation for section(3) 'kprobe/__arm64_sys_openat'
libbpf: sec '.relkprobe/__arm64_sys_openat': relo #0: insn #6 against 'open_count'
libbpf: prog 'bpf_prog': found map 0 (open_count, sec 5, off 0) for insn #6
libbpf: sec '.relkprobe/__arm64_sys_openat': relo #1: insn #19 against 'open_count'
libbpf: prog 'bpf_prog': found map 0 (open_count, sec 5, off 0) for insn #19
libbpf: prog 'bpf_prog': can't attach BPF program without FD (was it loaded?)
libbpf: prog 'bpf_prog': failed to attach to kprobe '__arm64_sys_openat+0x0': Invalid argument
libbpf: prog 'bpf_prog': failed to auto-attach: -22
Fail to load -22#
```



# Error



- 2025.01.27日

可以看见最终的运行结果是报错。

具体的报错内容是: 但是不确定这个-22错误码是什么。

```shell
libbpf: prog 'bpf_prog': failed to attach to kprobe '__arm64_sys_openat+0x0': Invalid argument
libbpf: prog 'bpf_prog': failed to auto-attach: -22
```

























