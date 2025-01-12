---
title: KProbe和ta的小伙伴！
tags:
- linux
cover:
---



# KProbe和ta的小伙伴



# Kprobes



Refs:[Linux Kernel Docs](https://docs.kernel.org/trace/kprobes.html#kernel-probes-kprobes)



> Kprobes enables you to dynamically break into any kernel routine and collect debugging and performance information non-disruptively. You can trap at almost any kernel code address [[1\]](https://docs.kernel.org/trace/kprobes.html#id2), specifying a handler routine to be invoked when the breakpoint is hit.



Kprobe是一种动态插桩的方法，你可以随意的插入到几乎所有内核的代码中（有黑名单），收集调试信息 & 性能信息，一旦命中kprobes名单立马会调用定义的处理函数。



Note：

1.Kprobe不能对所有的内核代码进行插桩，有黑名单（不能对kprobes本身插桩，不然会递归。）

2.Kprobe插桩的单位是指令

3.我们需要向内核注册Kprobes handler,这样到了指定的hook点内核才会执行我们的handler.





## Kprobes工作原理



[How Does Kprobes Works](https://docs.kernel.org/trace/kprobes.html#how-does-a-kprobe-work)

> When a kprobe is registered, Kprobes makes a copy of the probed instruction and replaces the first byte(s) of the probed instruction with a breakpoint instruction (e.g., int3 on i386 and x86_64).

[Blog](https://linux.laoqinren.net/kernel/linux-kprobe/)

![img](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/step1.png)



根据上面的参考资料我简单做个总结。（别问我细节，我也不知道～）

**在KProbes注册以前，程序指令正常执行。**

**在Kprobes注册以后，程序的指令会被动弹替换为一个中断指令，然后内核的控制逻辑会被劫持到我们注册的handler中去，最后再通过一次中断返回代码的执行。**

![kprobes](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/kprobes.png)





## Kprobes使用



[API Reference](https://docs.kernel.org/trace/kprobes.html#api-reference)



> 内核提供了部分的方法，简单列举。
>
> 具体可点击上方链接查看具体api.

```c
#include <linux/kprobes.h>
int register_kprobe(struct kprobe *kp);

#include <linux/kprobes.h>
#include <linux/ptrace.h>
int pre_handler(struct kprobe *p, struct pt_regs *regs);

#include <linux/kprobes.h>
#include <linux/ptrace.h>
void post_handler(struct kprobe *p, struct pt_regs *regs,
                  unsigned long flags);


#include <linux/kprobes.h>
int register_kretprobe(struct kretprobe *rp);

#include <linux/kprobes.h>
#include <linux/ptrace.h>
int kretprobe_handler(struct kretprobe_instance *ri,
                      struct pt_regs *regs);


#include <linux/kprobes.h>
void unregister_kprobe(struct kprobe *kp);
void unregister_kretprobe(struct kretprobe *rp);


#include <linux/kprobes.h>
int register_kprobes(struct kprobe **kps, int num);
int register_kretprobes(struct kretprobe **rps, int num);

#include <linux/kprobes.h>
void unregister_kprobes(struct kprobe **kps, int num);
void unregister_kretprobes(struct kretprobe **rps, int num);

```



# Uprobs



Refs:

- Uprobs基础讲解

[LWN.net](https://lwn.net/Articles/499190/)

[Linux Kernel Docs Uprob Traceer](https://docs.kernel.org/trace/uprobetracer.html)

> Uprobe based trace events are similar to kprobe based trace events. To enable this feature, build your kernel with CONFIG_UPROBE_EVENTS=y.

Uprobs和kProbes类似

[Blog](https://jayce.github.io/public/posts/trace/user-space-probes/)

- Uprobs原理分析

[eBPF Uprobes](https://www.cnxct.com/defeating-ebpf-uprobe-monitoring/)

[CSDN Blog](https://blog.csdn.net/u012489236/article/details/127954817)



通过上述Blog可知。无论是API上，还是从实现原理来看uprobes和kprobes都是类似的

1.API上

```c
#include <linux/uprobes.h>
int register_uprobe(struct uprobe *u);

include <linux/uprobes.h>
int register_uretprobe(struct uretprobe *rp);


#include <linux/uprobes.h>
#include <linux/ptrace.h>
void uretprobe_handler(struct uretprobe_instance *ri, struct pt_regs *regs);


#include <linux/uprobes.h>
void unregister_uprobe(struct uprobe *u);
void unregister_uretprobe(struct uretprobe *rp);


// ......

```

2.实现原理

Uprobs会函数进行插桩。将指令修改为中断，从而出发hook逻辑。

![在这里插入图片描述](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/d9c5f24e553b451414dd4f70b6e61644.png)





# Tracepoint









# USDT
