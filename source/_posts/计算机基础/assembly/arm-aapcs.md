---
title: ARM AAPCS64
tags:
  - arm
  - aapcs64
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250824232358406.png
date: 2025-08-24 23:16:35
---




# Arm64 aapcs解析之寄存器



# 基础概念



1.AAPCS (Procedure Call Standard for the Arm®  Architecture)

[see it](https://github.com/ARM-software/abi-aa/releases)

| Register | Special | Role in the procedure call standard                          |
| -------- | ------- | ------------------------------------------------------------ |
| SP       |         | The Stack Pointer.<br />堆栈指针                             |
| r30      | LR      | The Link Register.<br />链接寄存器                           |
| r29      | FP      | The Frame Pointer<br />堆栈指针寄存器                        |
| r19…r28  |         | Callee-saved registers<br />被调用方保存寄存器。调用方随意使用 |
| r18      |         | The Platform Register, if needed; otherwise a temporary register. See notes.<br />平台寄存器 |
| r17      | IP1     | The second intra-procedure-call temporary register (can be used by call veneers and PLT code); at other times may be used as a temporary register.<br />链接寄存器，非链接阶段随意使用 |
| r16      | IP0     | The first intra-procedure-call scratch register (can be used by call veneers and PLT code); at other times may be used as a temporary register.<br />链接寄存器，非链接阶段随意使用 |
| r9…r15   |         | Temporary registers<br />调用方保存，被调用方随意使用。      |
| r8       |         | Indirect result location register<br />非直接寄存器，当返回值>=16字节，通过x8传递返回值。 |
| r0…r7    |         | Parameter/result registers<br />用于传参 & 返回值，调用方和被调用方均不需要为其保存数据， |

# 分类



|    **类别**     |     **寄存器示例**      | **数量** | **位宽** |         **主要用途**          |
| :-------------: | :---------------------: | :------: | :------: | :---------------------------: |
|   通用寄存器    |    `X0`-`X30`, `XZR`    |  31 + 1  | 64/32 位 |  整型数据、参数传递、返回值   |
| 浮点/向量寄存器 | `V0`-`V31`（`S0`/`D0`） |    32    |  128 位  |    浮点运算、SIMD 并行操作    |
|   特殊寄存器    |  `SP`, `PC`, `PSTATE`   |   若干   | 64/混合  |  栈管理、指令指针、状态控制   |
|   系统寄存器    |  `SCTLR_EL1`, `TTBR0`   |   数百   | 64/32 位 | 系统配置（MMU、中断、计时等） |





# 通用寄存器



## x0 ~ x7

用于传参/返回数值

- Demo验证分析-[Demo Link](https://godbolt.org/z/vYeh9GE6e)

1. 传参：可见下方参数a1 ~ a8分别对应于x0~x7(图中写入的w0~w7效果等价使用x0 ~ x7,原因是w0~w7与x0~ x7共用同一个寄存器，区别在于位宽不一样)
2. 返回：可见下方test函数，将false返回值写入到x0中。

![image-20250727010754726](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250727010754726.png)

## x8

用作返回大对象>=16字节，假如一个函数的返回值的大小大于16字节(Arm64下Vn向量寄存器的最大位宽为128)，那么调用方需要传入一个x8寄存器，其中x8是函数返回值存放的地址。

其中x8通常是指向一块堆栈内存。具体逻辑可见下方。

- Demo验证

![image-20250727104541396](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250727104541396.png)

- 逻辑解释

可以发现main函数调用了testX8寄存器

其中testX8的返回值为20个字节，超过了最大的返回长度，因此需要通过x8寄存器做中转、

具体逻辑为：调用者需要开辟内存空间并将内存首地址写入到x8中，被调用者会将内存写入到x8下的内存中。

``` asm
main:
		// 开辟堆栈内存48个字节
        sub     sp, sp, #48
        // 将x29存储在[sp+32 ～ sp+39],x30存储在[sp+40,sp+47]
        stp     x29, x30, [sp, #32]
        // 修改基地址寄存器位置
        add     x29, sp, #32
        // 为返回值开辟空间[sp + 12 ~ sp + 31]
        add     x8, sp, #12
        // 调用testX8方法
        // testX8会将放回之写入到x8所在的内存空间即[sp + 12 ~ sp + 31]
        bl      testX8()
        mov     w0, wzr
        ldp     x29, x30, [sp, #32]
        add     sp, sp, #48
        ret
        
testX8():
		// 将x8值存在x9中
        mov     x9, x8
        // 下面两行指令为了将结构体默认值的内存地址保存在x8中
        adrp    x8, .L__const.testX8().bd
        add     x8, x8, :lo12:.L__const.testX8().bd
        // 将其前4个word共128个字节存储在q0寄存器中
        ldr     q0, [x8]
        // 保存到x9内存地址中
        str     q0, [x9]
        // 将最后一位值读取出来存在w8中
        ldr     w8, [x8, #16]
        // 将其save到x9 + 16 ～ x9 + 19处
        str     w8, [x9, #16]
        ret
        
.L__const.testX8().bd:
        .word   1
        .word   2
        .word   3
        .word   4
        .word   5
```



## x9 ～ x15

**调用者**保存的寄存器，对于**被调用者**可以认为是临时寄存器，随便用～不需要恢复到以前的状态值～



- Demo分析

如下方Demo可见testX9内会调用testX8方法。其中testX9中i < x, j < y 中的x,y均是临时存储在x9中。在调用testX8,方法体内部会修改x9的值，但并不会对程序造成任何影响。因为textX8作为**被调用者**可以任意使用x9～x15而作为**调用者**的testX9内部保存了x9的值，因此综合来看逻辑无任何异常。

![image-20250727122426152](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250727122426152.png)



## x16 ～ x18

| **寄存器**  |                           **用途**                           |
| :---------: | :----------------------------------------------------------: |
| `x16 (IP0)` | 过程间调用临时寄存器，动态链接或长跳转时专用；<br />否则为普通临时寄存器。 |
| `x17 (IP1)` |                            同上。                            |
|    `x18`    | 由操作系统或运行时环境（如 Linux、iOS、Android 等）保留使用。<br /> 若平台未明确使用 `x18`，则可作为临时寄存器（调用方保存）。 |



- x16, x17

从前面的列表可得知这两个寄存器和动态链接相关。但是没有详细说明x16,x17的具体含义。下面根据一个demo简单分析两个寄存器的作用

1.demo源码

```c++
// test.cpp
#include <cstdio>

void testPlt() {
    printf("Hello World!!!");
}
```

2.反汇编结果

``` asm
000000000001024c <_Z7testPltv>:
   1024c: a9bf7bfd      stp     x29, x30, [sp, #-0x10]!
   10250: 910003fd      mov     x29, sp
   10254: d503201f      nop
   10258: 10fa4cc0      adr     x0, 0x4bf0 <syscall+0x4bf0>
   // 由于涉及动态链接因此printf方法调用，编译器会为其生成一个printf@plt方法
   1025c: 9400129d      bl      0x14cd0 <printf@plt>
   10260: a8c17bfd      ldp     x29, x30, [sp], #0x10
   10264: d65f03c0      ret

......
// printf@plt方法内部会去从当前so的got表(即.got.plt段)中定位实际的函数地址位置
// 其中x16与x17就在定位过程中使用到。
// 其中x16寄存器中存储的是当前函数跳转信息的got表地址
// 其中x17寄存器中存储的是当前函数的实际跳转地址。
0000000000014cd0 <printf@plt>:
	// 1. 解析got表的基地址
   14cd0: d0000010      adrp    x16, 0x16000 <_DYNAMIC+0x198>
    // 2. 获取got表中printf的实际地址并存储在x17寄存器中 
   14cd4: f941da11      ldr     x17, [x16, #0x3b0]
   //  3. 获取got表中存放printf记录的地址。
   14cd8: 910ec210      add     x16, x16, #0x3b0
   //  4. 跳转到实际的printf函数中去
   14cdc: d61f0220      br      x17
```



- x18

平台寄存器，即不同平台用处可能不一样以Android平台为例[Android ABIs](https://developer.android.com/ndk/guides/abis#arm64-v8a)

> On Android, the platform-specific x18 register is reserved for [ShadowCallStack](https://source.android.com/devices/tech/debug/shadow-call-stack) and should not be touched by your code. Current versions of Clang default to using the `-ffixed-x18` option on Android, so unless you have hand-written assembler (or a very old compiler) you shouldn't need to worry about this.
>
> **ShadowCallStack（影子调用栈）** 是 Android 引入的一种安全机制，用于防御 **ROP（Return-Oriented Programming）攻击**。其核心原理是：
>
> 1. **备份返回地址**：将函数的返回地址存储在一个独立的、受保护的 **影子栈**（Shadow Stack）中，而非传统调用栈。
> 2. **运行时校验**：在函数返回时，比较调用栈中的返回地址和影子栈中备份的地址，若不一致则触发安全异常。
>
> 默认情况下Android Clang 编译器会启用 **-ffixed-x18** 强制编译器避免使用 x18 寄存器，确保其专用于 ShadowCallStack。



## x19~x28

**被调用者**保存寄存器。简单来说就是在使用这些寄存器以前需要对内容做保存，使用完后需要恢复。因为调用者可能存储了一些中间计算结果。

可以发现编译器会在使用X19～X29以前保存寄存器。

![image-20250727225323783](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250727225323783.png)



## X29～X30

| **寄存器** |           **功能**           |
| :--------: | :--------------------------: |
|  **X29**   | 栈帧基地址（调试和变量定位） |
|  **X30**   | 返回地址（函数调用流程控制） |

- X29

函数在刚开始执行时，会将调用方的x29,x30寄存器进行保存。在访问和存入局部变量的时候会通过x29进行访问(此处局部变量为return)

![image-20250728003637746](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250728003637746.png)

- X30

在**函数执行前**会将X30的值进行**保存**。在**执行完毕以后**会**恢复**该值。其中X30的寄存器的作用如下：

函数调用过程中会调用bl指令进行**跳转**，但是于此同时会**记录**函数执行完毕以后的**返回地址**(当前执行的下一条指令地址)**到X30中**。当函数执行完毕以后会调用return指令**返回上一个函数**的执行此时会**读取X30的值**并将程序执行跳转到X30指向地址(此时不会修改X30的值)。

![image-20250728004817977](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250728004817977.png)



## SP

堆栈指针，用于存放数据。

可以将其当成普通的寄存器进行内存空间的开辟和释放。由于堆栈的内存是**反向增长**的，因此可以通过对sp进行**减法操作开辟内存**，通过**加法操作释放内存**。

![image-20250728011559039](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250728011559039.png)





# 其他



## 为什么需要AAPCS规范？



The goals of the AAPCS64 are to:

- Support efficient execution on high-performance implementations of the Arm 64-bit Architecture.
- Clearly distinguish between mandatory requirements and implementation discretion.



其实核心就是支持在Arm64架构下高效执行应用程序。(前提是需要遵循规范。)



## 规范并不是强制的,我可以不遵守吗？



可以，比如如果你是手写汇编，你可以自行处置寄存器，你自己进行寄存器职责的划分。

但是如果不是有特殊的需求，建议还是遵循，毕竟长期以来的规范，经过时间的洗礼，性能稳定性都有一定的考量。



## 为什么需要那么多寄存器？



寄存器是CPU内部的元器件，当然也属于内存的范畴，而内存是有延迟的，就像金字塔一样[Wikipedia](https://zh.wikipedia.org/zh-cn/%E8%A8%98%E6%86%B6%E9%AB%94%E9%9A%8E%E5%B1%A4)

大多内存的延迟相比于CPU的执行速度来讲，都过于缓慢，然而寄存器处于存储金字塔的顶端，能满足CPU的高速计算。

因此更多的寄存器会显著提高程序的执行性能，同时更多的寄存器也意味着更高的造价。

