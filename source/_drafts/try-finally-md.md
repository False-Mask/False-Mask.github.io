---
title: try-finally.md
date: 2024-12-08 16:36:17
tags:
- android
- 
cover:
---



# ASM插桩生成Try Finally



> 内容如标题，怎么通过ASM插桩生成Try Finally代码



# 基础概念







## 插桩



> 程序插桩，最早是由J.C. Huang 教授提出的，它是在保证被测程序原有逻辑完整性的基础上在程序中插入一些[探针](https://baike.baidu.com/item/探针/1846154?fromModule=lemma_inlink)（又称为“[探测仪](https://baike.baidu.com/item/探测仪/0?fromModule=lemma_inlink)”，本质上就是进行信息采集的[代码段](https://baike.baidu.com/item/代码段/9966451?fromModule=lemma_inlink)，可以是[赋值语句](https://baike.baidu.com/item/赋值语句/4248688?fromModule=lemma_inlink)或采集覆盖信息的[函数调用](https://baike.baidu.com/item/函数调用/4127405?fromModule=lemma_inlink)），通过[探针](https://baike.baidu.com/item/探针/1846154?fromModule=lemma_inlink)的执行并抛出程序运行的[特征](https://baike.baidu.com/item/特征/6205236?fromModule=lemma_inlink)数据，通过对这些数据的[分析](https://baike.baidu.com/item/分析/4327108?fromModule=lemma_inlink)，可以获得程序的[控制流](https://baike.baidu.com/item/控制流/854473?fromModule=lemma_inlink)和数据流信息，进而得到[逻辑覆盖](https://baike.baidu.com/item/逻辑覆盖/3231015?fromModule=lemma_inlink)等动态信息，从而实现测试目的的方法。



> 简单来说就是在代码中通过特殊的手段再插入一段代码。
>
> 比如，原来的代码逻辑是A -> B -> C插桩后的代码为
>
> A -> B -> X -> C



## ASM



> ASM其实是Assembly的缩写。ASM在编程领域有多种语义。
>
> 这里的[ASM](https://asm.ow2.io/index.html)指的是Java的字节码插桩框架。



## Try Finally



> Try Finally是Java的一种异常处理方案。

```java
try {
    
    // do something
    // ......
} finally {
    // 1. 如果发生异常，会执行如下代码
    // 2. 如果没有发生异常，也有执行如下代码
    // ......
    
}
```



> 如上所示，Try Finally的作用其实是保证代码在结束一定会执行。

这种逻辑在很多地方都可以用到，通常是用于资源的释放。

```java
try {
    
    // 资源申请
    // ......
    
} finally {
    // 资源释放
    // ......
}
```



> 这不是说一定用于上述的场景。很多其他常见也可以用。
>
> finally他们都有一个共性——方法执行完成前，必须执行完。



# 目标



# 过程





# 结果









# References





1.[CSDN Blog](https://blog.csdn.net/jdsjlzx/article/details/139304867)









