---
title: Android Art Trampoline解析
tags:
cover:
---

# Android Art Trampoline



# pre


## 关于本文

主要针对于**Android14.0.0_r73**源码下的**arm64**下的Art Trampoline跳板执行逻辑进行分析


## ArtMethod::Invoke

```c++
NO_STACK_PROTECTOR
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
                       const char* shorty) {
  // check堆栈大小是否超边界  
  if (UNLIKELY(__builtin_frame_address(0) < self->GetStackEnd())) {
    ThrowStackOverflowError(self);
    return;
  }

  if (kIsDebugBuild) {
    self->AssertThreadSuspensionIsAllowable();
    CHECK_EQ(ThreadState::kRunnable, self->GetState());
    CHECK_STREQ(GetInterfaceMethodIfProxy(kRuntimePointerSize)->GetShorty(), shorty);
  }

  // push 一个StackFragment
  // Push a transition back into managed code onto the linked list in thread.
  ManagedStack fragment;
  self->PushManagedStackFragment(&fragment);

  Runtime* runtime = Runtime::Current();
  // Call the invoke stub, passing everything as arguments.
  // If the runtime is not yet started or it is required by the debugger, then perform the
  // Invocation by the interpreter, explicitly forcing interpretation over JIT to prevent
  // cycling around the various JIT/Interpreter methods that handle method invocation.
  // 使用interpretor执行代码  
  if (UNLIKELY(!runtime->IsStarted() ||
               (self->IsForceInterpreter() && !IsNative() && !IsProxyMethod() && IsInvokable()))) {
    if (IsStatic()) {
      art::interpreter::EnterInterpreterFromInvoke(
          self, this, nullptr, args, result, /*stay_in_interpreter=*/ true);
    } else {
      mirror::Object* receiver =
          reinterpret_cast<StackReference<mirror::Object>*>(&args[0])->AsMirrorPtr();
      art::interpreter::EnterInterpreterFromInvoke(
          self, this, receiver, args + 1, result, /*stay_in_interpreter=*/ true);
    }
  } else {
    DCHECK_EQ(runtime->GetClassLinker()->GetImagePointerSize(), kRuntimePointerSize);

    constexpr bool kLogInvocationStartAndReturn = false;
    bool have_quick_code = GetEntryPointFromQuickCompiledCode() != nullptr;
    // 调用quick_point执行逻辑
    if (LIKELY(have_quick_code)) {
      if (kLogInvocationStartAndReturn) {
        LOG(INFO) << StringPrintf(
            "Invoking '%s' quick code=%p static=%d", PrettyMethod().c_str(),
            GetEntryPointFromQuickCompiledCode(), static_cast<int>(IsStatic() ? 1 : 0));
      }

      // Ensure that we won't be accidentally calling quick compiled code when -Xint.
      if (kIsDebugBuild && runtime->GetInstrumentation()->IsForcedInterpretOnly()) {
        CHECK(!runtime->UseJitCompilation());
        const void* oat_quick_code =
            (IsNative() || !IsInvokable() || IsProxyMethod() || IsObsolete())
            ? nullptr
            : GetOatMethodQuickCode(runtime->GetClassLinker()->GetImagePointerSize());
        CHECK(oat_quick_code == nullptr || oat_quick_code != GetEntryPointFromQuickCompiledCode())
            << "Don't call compiled code when -Xint " << PrettyMethod();
      }

      if (!IsStatic()) {
        (*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
      } else {
        (*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
      }
      if (UNLIKELY(self->GetException() == Thread::GetDeoptimizationException())) {
        // Unusual case where we were running generated code and an
        // exception was thrown to force the activations to be removed from the
        // stack. Continue execution in the interpreter.
        self->DeoptimizeWithDeoptimizationException(result);
      }
      if (kLogInvocationStartAndReturn) {
        LOG(INFO) << StringPrintf("Returned '%s' quick code=%p", PrettyMethod().c_str(),
                                  GetEntryPointFromQuickCompiledCode());
      }
    } else {
       // 如果没有quick_point直接返回空
      LOG(INFO) << "Not invoking '" << PrettyMethod() << "' code=null";
      if (result != nullptr) {
        result->SetJ(0);
      }
    }
  }

  // Pop transition.
  self->PopManagedStackFragment(fragment);
}
```



# 一层跳板



1.art寄存器规范(看看就行，跳板函数基本上为了考虑性能，都是汇编写的，了解了方便后续看汇编的执行过程)

```asm
/**
 * ARM64 Runtime register usage conventions.
 *
 *   r0     : w0 is 32-bit return register and x0 is 64-bit.
 *   r0-r7  : Argument registers.
 *   r8-r15 : Caller save registers (used as temporary registers).
 *   r16-r17: Also known as ip0-ip1, respectively. Used as scratch registers by
 *            the linker, by the trampolines and other stubs (the compiler uses
 *            these as temporary registers).
 *   r18    : Reserved for platform (SCS, shadow call stack)
 *   r19    : Pointer to thread-local storage.
 *   r20-r29: Callee save registers.
 *   r30    : (lr) is reserved (the link register).
 *   rsp    : (sp) is reserved (the stack pointer).
 *   rzr    : (zr) is reserved (the zero register).
 *
 *   Floating-point registers
 *   v0-v31
 *
 *   v0     : s0 is return register for singles (32-bit) and d0 for doubles (64-bit).
 *            This is analogous to the C/C++ (hard-float) calling convention.
 *   v0-v7  : Floating-point argument registers in both Dalvik and C/C++ conventions.
 *            Also used as temporary and codegen scratch registers.
 *
 *   v0-v7 and v16-v31 : Caller save registers (used as temporary registers).
 *   v8-v15 : bottom 64-bits preserved across C calls (d8-d15 are preserved).
 *
 *   v16-v31: Used as codegen temp/scratch.
 *   v8-v15 : Can be used for promotion.
 *
 *   Must maintain 16-byte stack alignment.
 *
 * Nterp notes:
 *
 * The following registers have fixed assignments:
 *
 *   reg nick      purpose
 *   x19  xSELF     self (Thread) pointer
 *   x20  wMR       marking register
 *   x21  xSUSPEND  suspend check register
 *   x29  xFP       interpreted frame pointer, used for accessing locals and args
 *   x22  xPC       interpreted program counter, used for fetching instructions
 *   x23  xINST     first 16-bit code unit of current instruction
 *   x24  xIBASE    interpreted instruction base pointer, used for computed goto
 *   x25  xREFS     base of object references of dex registers.
 *   x16  ip        scratch reg
 *   x17  ip2       scratch reg (used by macros)
 *
 * Macros are provided for common operations.  They MUST NOT alter unspecified registers or
 * condition codes.
*/
```

2.执行逻辑概览

一层跳板的逻辑其实**比较简单**，主要还是在**做铺垫**吧～核心执行逻辑都在二级跳板中～

a. 寄存器信息保存(主要是AAPCS64规定的callee saved 寄存器)

b. 将Java的传参信息从堆栈拷贝到寄存器中, 为后续方法执行做准备(INVOKE_STUB_LOAD_ALL_ARGS)

c. 执行二级跳板,并将返回值保存到ArtMethod.Invoke传入的JValue参数中去。







## art_quick_invoke_static_stub



1.遵循AAPCS64规范对寄存器进行保存

2.将Java函数参数保存到寄存器中

3.调用二级trampoline跳板

```asm
/*  extern"C"
 *     void art_quick_invoke_static_stub(ArtMethod *method,   x0
 *                                       uint32_t  *args,     x1
 *                                       uint32_t argsize,    w2
 *                                       Thread *self,        x3
 *                                       JValue *result,      x4
 *                                       char   *shorty);     x5
 */
ENTRY art_quick_invoke_static_stub
    // Spill registers as per AACPS64 calling convention.
    INVOKE_STUB_CREATE_FRAME

    // Load args into registers.
    INVOKE_STUB_LOAD_ALL_ARGS _static

    // Call the method and return.
    INVOKE_STUB_CALL_AND_RETURN
END art_quick_invoke_static_stub
```



#### INVOKE_STUB_CREATE_FRAME



堆栈内存结构，从高到低 

x4, x5, \<padding\>(8 bytes), x19, x20, x21, FP, LR saved.

```asm
.macro INVOKE_STUB_CREATE_FRAME
SAVE_SIZE=8*8   // x4, x5, <padding>, x19, x20, x21, FP, LR saved.
	// 保存x4,x5寄存器并更新sp的值为sp - 64
    SAVE_TWO_REGS_INCREASE_FRAME x4, x5, SAVE_SIZE
    // 保存x19寄存器
    SAVE_REG      x19,      24
    // 保存x20, x21寄存器
    SAVE_TWO_REGS x20, x21, 32
    // 保存x29, x30寄存器
    SAVE_TWO_REGS xFP, xLR, 48
	// 将sp寄存器存储到x29中
    mov xFP, sp                            // Use xFP for frame pointer, as it's callee-saved.
    .cfi_def_cfa_register xFP
	// 为ArtMethod保留一些内存，并进行16字节对其
	// Note: 此处x2为ArtMethod::Invoke中调用art_quick_invoke_stub传入的，其值为args_size
    add x10, x2, #(__SIZEOF_POINTER__ + 0xf) // Reserve space for ArtMethod*, arguments and
    and x10, x10, # ~0xf                   // round up for 16-byte stack alignment.
    sub sp, sp, x10                        // Adjust SP for ArtMethod*, args and alignment padding.
	// 将线程对象保存到xSELF x19中
    mov xSELF, x3                          // Move thread pointer into SELF register.

    // Copy arguments into stack frame.
    // Use simple copy routine for now.
    // 4 bytes per slot.
    // X1 - source address
    // W2 - args length
    // X9 - destination address.
    // W10 - temporary
    
    // 将sp + 8， 为ArtMethod* 提供存储空间。（也就是str xzr, [sp]）
    add x9, sp, #8                         // Destination address is bottom of stack + null.

    // Copy parameters into the stack. Use numeric label as this is a macro and Clang's assembler
    // does not have unique-id variables.
    // 将所有参数拷贝到堆栈上。
    cbz w2, 2f
1:
    sub w2, w2, #4      // Need 65536 bytes of range.
    ldr w10, [x1, x2]
    str w10, [x9, x2]
    cbnz w2, 1b

2:
    // Store null into ArtMethod* at bottom of frame.
    str xzr, [sp]
.endm
```



#### INVOKE_STUB_LOAD_ALL_ARGS

将Java的参数从堆栈中保存到寄存器中

```c++
.macro INVOKE_STUB_LOAD_ALL_ARGS suffix
    add x10, x5, #1                 // Load shorty address, plus one to skip the return type.

    // Load this (if instance method) and addresses for routines that load WXSD registers.
    .ifc \suffix, _instance
        ldr w1, [x9], #4            // Load "this" parameter, and increment arg pointer.
        adr x11, .Lload_w2\suffix
        adr x12, .Lload_x2\suffix
    .else
        adr x11, .Lload_w1\suffix
        adr x12, .Lload_x1\suffix
    .endif
    adr  x13, .Lload_s0\suffix
    adr  x14, .Lload_d0\suffix

    // Loop to fill registers.
.Lfill_regs\suffix:
    ldrb w17, [x10], #1             // Load next character in signature, and increment.
    cbz w17, .Lcall_method\suffix   // Exit at end of signature. Shorty 0 terminated.

    cmp w17, #'J'                   // Is this a long?
    beq .Lload_long\suffix

    cmp  w17, #'F'                  // Is this a float?
    beq .Lload_float\suffix

    cmp w17, #'D'                   // Is this a double?
    beq .Lload_double\suffix

    // Everything else uses a 4-byte GPR.
    br x11

.Lload_long\suffix:
    br x12

.Lload_float\suffix:
    br x13

.Lload_double\suffix:
    br x14

// Handlers for loading other args (not float/double/long) into W registers.
    .ifnc \suffix, _instance
        INVOKE_STUB_LOAD_REG \
            .Lload_w1, w1, x9, 4, x11, .Lload_w2, x12, .Lload_x2, .Lfill_regs, \suffix
    .endif
    INVOKE_STUB_LOAD_REG .Lload_w2, w2, x9, 4, x11, .Lload_w3, x12, .Lload_x3, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_w3, w3, x9, 4, x11, .Lload_w4, x12, .Lload_x4, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_w4, w4, x9, 4, x11, .Lload_w5, x12, .Lload_x5, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_w5, w5, x9, 4, x11, .Lload_w6, x12, .Lload_x6, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_w6, w6, x9, 4, x11, .Lload_w7, x12, .Lload_x7, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_w7, w7, x9, 4, x11, .Lskip4, x12, .Lskip8, .Lfill_regs, \suffix

// Handlers for loading longs into X registers.
    .ifnc \suffix, _instance
        INVOKE_STUB_LOAD_REG \
            .Lload_x1, x1, x9, 8, x11, .Lload_w2, x12, .Lload_x2, .Lfill_regs, \suffix
    .endif
    INVOKE_STUB_LOAD_REG .Lload_x2, x2, x9, 8, x11, .Lload_w3, x12, .Lload_x3, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_x3, x3, x9, 8, x11, .Lload_w4, x12, .Lload_x4, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_x4, x4, x9, 8, x11, .Lload_w5, x12, .Lload_x5, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_x5, x5, x9, 8, x11, .Lload_w6, x12, .Lload_x6, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_x6, x6, x9, 8, x11, .Lload_w7, x12, .Lload_x7, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_x7, x7, x9, 8, x11, .Lskip4, x12, .Lskip8, .Lfill_regs, \suffix

// Handlers for loading singles into S registers.
    INVOKE_STUB_LOAD_REG .Lload_s0, s0, x9, 4, x13, .Lload_s1, x14, .Lload_d1, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_s1, s1, x9, 4, x13, .Lload_s2, x14, .Lload_d2, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_s2, s2, x9, 4, x13, .Lload_s3, x14, .Lload_d3, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_s3, s3, x9, 4, x13, .Lload_s4, x14, .Lload_d4, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_s4, s4, x9, 4, x13, .Lload_s5, x14, .Lload_d5, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_s5, s5, x9, 4, x13, .Lload_s6, x14, .Lload_d6, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_s6, s6, x9, 4, x13, .Lload_s7, x14, .Lload_d7, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_s7, s7, x9, 4, x13, .Lskip4, x14, .Lskip8, .Lfill_regs, \suffix

// Handlers for loading doubles into D registers.
    INVOKE_STUB_LOAD_REG .Lload_d0, d0, x9, 8, x13, .Lload_s1, x14, .Lload_d1, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_d1, d1, x9, 8, x13, .Lload_s2, x14, .Lload_d2, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_d2, d2, x9, 8, x13, .Lload_s3, x14, .Lload_d3, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_d3, d3, x9, 8, x13, .Lload_s4, x14, .Lload_d4, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_d4, d4, x9, 8, x13, .Lload_s5, x14, .Lload_d5, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_d5, d5, x9, 8, x13, .Lload_s6, x14, .Lload_d6, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_d6, d6, x9, 8, x13, .Lload_s7, x14, .Lload_d7, .Lfill_regs, \suffix
    INVOKE_STUB_LOAD_REG .Lload_d7, d7, x9, 8, x13, .Lskip4, x14, .Lskip8, .Lfill_regs, \suffix

// Handlers for skipping arguments that do not fit into registers.
    INVOKE_STUB_SKIP_ARG .Lskip4, x9, 4, .Lfill_regs, \suffix
    INVOKE_STUB_SKIP_ARG .Lskip8, x9, 8, .Lfill_regs, \suffix

.Lcall_method\suffix:
.endm
```



##### INVOKE_STUB_LOAD_REG

通过注释可以得知这个宏定义是想将java方法参数拷贝到寄存器中

- label 宏定义会为每个过程调用生成一个标签，label则是标签的前缀，后缀在哪呢？轻卡suffix
- reg 参数最终需要拷贝到的寄存器名称
- args 参数指针，每次读取会累加size个长度
- size 参数大小
- nh4_reg 4字节长度的寄存器，用于承载下一个处理单元的函数地址 
- nh4_l 下一个处理单元的label名称
- nh8_reg 同nh4_reg, 只不过长度为8字节
- nh8_l 同nh4_l， 长度为8自己
- cont 下一个用于处理shorty符号的label
- suffix 标签后缀

```asm
// Macro for loading an argument into a register.
//  label - the base name of the label of the load routine,
//  reg - the register to load,
//  args - pointer to current argument, incremented by size,
//  size - the size of the register - 4 or 8 bytes,
//  nh4_reg - the register to fill with the address of the next handler for 4-byte values,
//  nh4_l - the base name of the label of the next handler for 4-byte values,
//  nh8_reg - the register to fill with the address of the next handler for 8-byte values,
//  nh8_l - the base name of the label of the next handler for 8-byte values,
//  cont - the base name of the label for continuing the shorty processing loop,
//  suffix - suffix added to all labels to make labels unique for different users.
.macro INVOKE_STUB_LOAD_REG label, reg, args, size, nh4_reg, nh4_l, nh8_reg, nh8_l, cont, suffix
\label\suffix:
    ldr \reg, [\args], #\size
    adr \nh4_reg, \nh4_l\suffix
    adr \nh8_reg, \nh8_l\suffix
    b \cont\suffix
.endm
```



##### INVOKE_STUB_SKIP_ARG

用于跳过参数读取的处理方法

- label  标签名称
- args 执行参数的指针
- size 下个参数的大小
- cont shorty 处理循环地址
- suffix 标签后缀

```asm
// Macro for skipping an argument that does not fit into argument registers.
//  label - the base name of the label of the skip routine,
//  args - pointer to current argument, incremented by size,
//  size - the size of the argument - 4 or 8 bytes,
//  cont - the base name of the label for continuing the shorty processing loop,
//  suffix - suffix added to all labels to make labels unique for different users.
.macro INVOKE_STUB_SKIP_ARG label, args, size, cont, suffix
\label\suffix:
	// 读取一个函数参数并跳转到shorty loop中去
    add \args, \args, #\size
    b \cont\suffix
.endm
```





##### loop branch 



```asm
.Lfill_regs\suffix:
	// x10中存储了shorty args表示。
	// 首先先读取一个字符保存到w17中并进行index的累加
    ldrb w17, [x10], #1             // Load next character in signature, and increment.
    // 判断是参数签名是否已经全部处理完毕，如果处理完毕，直接跳转到结尾
    cbz w17, .Lcall_method\suffix   // Exit at end of signature. Shorty 0 terminated.
	// 如果是一个Long类型的参数，跳转到long参数处理方法
    cmp w17, #'J'                   // Is this a long?
    beq .Lload_long\suffix
	// 如果是float类型的参数，跳转到flaot参数处理方法中
    cmp  w17, #'F'                  // Is this a float?
    beq .Lload_float\suffix
	// 如果是double类型的参数，跳转到double参数处理方法中
    cmp w17, #'D'                   // Is this a double?
    beq .Lload_double\suffix

    // Everything else uses a 4-byte GPR.
    // 其他所有参数都默认为4字节长度
    br x11
```





##### call_method



// TODO确认下，call_method的执行逻辑

通过源码分析可以发现call_method label对应于INVOKE_STUB_LOAD_ALL_ARGS的下一行，也就是INVOKE_STUB_CALL_AND_RETURN的第一行指令。

为什么呢？

因为INVOKE_STUB_CREATE_FRAME, INVOKE_STUB_LOAD_ALL_ARGS, INVOKE_STUB_CALL_AND_RETURN都会被内联到art_quick_invoke_stub方法中

```asm
ENTRY art_quick_invoke_stub
    // Spill registers as per AACPS64 calling convention.
    INVOKE_STUB_CREATE_FRAME

    // Load args into registers.
    INVOKE_STUB_LOAD_ALL_ARGS _instance

    // Call the method and return.
    INVOKE_STUB_CALL_AND_RETURN
END art_quick_invoke_stub



.macro INVOKE_STUB_LOAD_ALL_ARGS

// ......

.Lcall_method\suffix:
.endm

```







##### load_x

这里的load_x是Lload_long/Lload_float/Lload_double

代码非常的简单，就是直接跳转到特定的寄存器

- x12 —— 存储读取long类型的参数的处理函数的地址
- x13 —— 存储读取float类型的类型的处理函数的地址
- x14——  存储读取double类型的类型的处理函数的地址

```asm
.Lload_long\suffix:
    br x12

.Lload_float\suffix:
    br x13

.Lload_double\suffix:
    br x14
```





#### INVOKE_STUB_CALL_AND_RETURN

该方法的执行流程可分为如下几个过程

1.读取artMethod.METHOD_QUICK_CODE_OFFSET并执行（二级跳板）

2.回收堆栈空间，弹出堆栈指针

3.获取返回值从x0/s0/d0保存到JValue中

4.返回函数执行

```asm
.macro INVOKE_STUB_CALL_AND_RETURN

    REFRESH_MARKING_REGISTER
    REFRESH_SUSPEND_CHECK_REGISTER

    // load method-> METHOD_QUICK_CODE_OFFSET
    // #1 执行二级跳板
    ldr x9, [x0, #ART_METHOD_QUICK_CODE_OFFSET_64]
    // Branch to method.
    blr x9

	// ＃2 弹出栈帧，包含artMethod(null), Java函数参数，对其字节
    // Pop the ArtMethod* (null), arguments and alignment padding from the stack.
    mov sp, xFP
    .cfi_def_cfa_register sp

	// 恢复之前保存的所有栈帧
    // Restore saved registers including value address and shorty address.
    RESTORE_REG      x19,      24
    RESTORE_TWO_REGS x20, x21, 32
    RESTORE_TWO_REGS xFP, xLR, 48
    RESTORE_TWO_REGS_DECREASE_FRAME x4, x5, SAVE_SIZE

	// #3 获取返回值保存到Jvalue中
	// 先读取shorty signature的第一个字符，也就是返回值类型。然后对不同类型的值进行单独操作
    // Store result (w0/x0/s0/d0) appropriately, depending on resultType.
    ldrb w10, [x5]

    // Check the return type and store the correct register into the jvalue in memory.
    // Use numeric label as this is a macro and Clang's assembler does not have unique-id variables.

    // Don't set anything for a void type.
    // Void类型，直接返回不设置返回值（jump 到label1）
    cmp w10, #'V'
    beq 1f

    // Is it a double?
    // double类型，jump到label 2 将d0的值设置到x4对应的内存地址中
    cmp w10, #'D'
    beq 2f

    // Is it a float?
    // float类型，jump到label 3 将s0的值设置到x4对应的内存地址中
    cmp w10, #'F'
    beq 3f

    // Just store x0. Doesn't matter if it is 64 or 32 bits.
    // 其他情况，将x0的值直接设置到x4对于的内存中去
    str x0, [x4]

	// #4 结束返回函数执行
1:  // Finish up.
    ret

2:  // Store double.
    str d0, [x4]
    ret

3:  // Store float.
    str s0, [x4]
    ret

.endm
```



##### REFRESH_MARKING_REGISTER

看着像是更新标记位

```asm
// Macro to refresh the Marking Register (W20).
//
// This macro must be called at the end of functions implementing
// entrypoints that possibly (directly or indirectly) perform a
// suspend check (before they return).
.macro REFRESH_MARKING_REGISTER
#ifdef RESERVE_MARKING_REGISTER
    ldr wMR, [xSELF, #THREAD_IS_GC_MARKING_OFFSET]
#endif
.endm
```



##### REFRESH_SUSPEND_CHECK_REGISTER

更新suspend标记

```shell
// Macro to refresh the suspend check register.
//
// We do not refresh `xSUSPEND` after every transition to Runnable, so there is
// a chance that an implicit suspend check loads null to xSUSPEND but before
// causing a SIGSEGV at the next implicit suspend check we make a runtime call
// that performs the suspend check explicitly. This can cause a spurious fault
// without a pending suspend check request but it should be rare and the fault
// overhead was already expected when we triggered the suspend check, we just
// pay the price later than expected.
.macro REFRESH_SUSPEND_CHECK_REGISTER
    ldr xSUSPEND, [xSELF, #THREAD_SUSPEND_TRIGGER_OFFSET]
.endm
```



## art_quick_invoke_stub


art_quick_invoke_stub主要是针对于成员方法进行执行，分析过了art_quick_invoke_static_stub不难发现art_quick_invoke_stub的执行逻辑也是类似的。


执行逻辑分为如下三个步骤：

1.遵循AAPCS64规范对寄存器进行保存

2.将Java函数参数保存到寄存器中

3.调用二级trampoline跳板
```asm
ENTRY art_quick_invoke_stub

// Spill registers as per AACPS64 calling convention.

INVOKE_STUB_CREATE_FRAME

  

// Load args into registers.

INVOKE_STUB_LOAD_ALL_ARGS _instance

  

// Call the method and return.

INVOKE_STUB_CALL_AND_RETURN

END art_quick_invoke_stub
```


不难发现他们主要的差异点在与上述执行流程中的#2，接下来我们针对于这个宏展开的执行逻辑进行分析。
```asm
// art_quick_invoke_static_stub
INVOKE_STUB_LOAD_ALL_ARGS _static

// art_quick_invoke_stub
INVOKE_STUB_LOAD_ALL_ARGS _instance
```


这个宏主要的作用是将方法参数填充到w1 ~ w7,x1 ~ x7,s0 ~ s7,d0 ~ d7。但是由于普通的成员方法有this参数，而静态方法是没有this参数的。因此：
- 对于普通成员方法，w1存储this, w2~w7/x2~x7存储方法参数
- 对于静态成员方法，w1～w7/x1~x7存储方法参数
因此w1/x1参数的读取需要单独处理。宏展开的主要差异点也就在这里。

```asm
.macro INVOKE_STUB_LOAD_ALL_ARGS suffix

add x10, x5, #1 // Load shorty address, plus one to skip the return type.

  

// Load this (if instance method) and addresses for routines that load WXSD registers.
// 程序一：加载this对象(按实际条件)
// 差异点一，成员方法的this参数需要单独加载
.ifc \suffix, _instance
	// 直接load成员方法的this对象到w1中
	ldr w1, [x9], #4 // Load "this" parameter, and increment arg pointer.
	// 后续从w2/x2开始
	adr x11, .Lload_w2\suffix
	adr x12, .Lload_x2\suffix
.else
	// 非成员方法(也就是静态方法)不需要load this,正常从w1/x1开始load
	adr x11, .Lload_w1\suffix
	adr x12, .Lload_x1\suffix

.endif

// ......

  
// 程序二：循环读取每一个参数
// .......
  

// 程序三：加载32位参数到w1 ~ w7
// 差异点二静态方法需要定义加载w1的handler，这对于成员方法是不需要的
.ifnc \suffix, _instance
	INVOKE_STUB_LOAD_REG \
	.Lload_w1, w1, x9, 4, x11, .Lload_w2, x12, .Lload_x2, .Lfill_regs, \suffix
.endif

// 定义handler用于加载32位int到w2 ~ w7
// ......


// 程序三：加载64位参数到x1~x7中  
// 差异点三静态方法需要定义加载x1的handler，这对于成员方法是不需要的
.ifnc \suffix, _instance
	INVOKE_STUB_LOAD_REG \
	.Lload_x1, x1, x9, 8, x11, .Lload_w2, x12, .Lload_x2, .Lfill_regs, \suffix
.endif

// 定义handler用于加载x2 ~ x7
// ......


// 程序四：定义handler用于float参数到s0 ~ s7
// ......


// 程序五：定义handler用于加载double参数到d0 ~ d7
// ......

// 程序六：定义handler用于做skip操作，也就是啥都不做(加载完毕通常会调用)～
// ......

.Lcall_method\suffix:

.endm
```





# art二级跳板函数分析



## 跳板函数分类

art的跳板较多，为了方便理解，在实际接触这些东西之前我们先简单做个分类。

| 类型                       | 备注                                                                                                      |
| ------------------------ | ------------------------------------------------------------------------------------------------------- |
| 解释器跳板                    | art_quick_to_interpreter_bridge<br />ExecuteNterpImpl<br />art_quick_resolution_trampoline              |
| JNI跳板                    | art_jni_dlsym_lookup_stub<br />art_jni_dlsym_lookup_critical_stub<br />art_quick_generic_jni_trampoline |
| 接口重载跳板                   | art_quick_imt_conflict_trampoline                                                                       |
| Deoptimized跳板(for Debug) | art_invoke_obsolete_method_stub<br />art_quick_deoptimize                                               |
| Proxy跳板                  | art_quick_proxy_invoke_handler                                                                          |



## art_quick_to_interpreter_bridge 

该方法用于跳转到解释器的跳板函数，通常情况下的调用堆栈如下为：

-> ArtMethod->entry_point_from_quick_compiled_code_
	->art_quick_to_interpreter_bridge 
		-> artQuickToInterpreterBridge 
			-> Execute 
				-> ExecuteSwitch 
					-> ExecuteSwitchImplAsm
						->ExecuteSwitchImplCpp
							->......

当期跳板函数的执行逻辑如下：

1.开辟栈帧

2.调用artQuickToInterpreterBridge方法，并通过寄存器传参

3.恢复堆栈指针/刷新堆栈指针

4.返回结果处理(抛异常/返回值)

``` c++
/*
 * Called to bridge from the quick to interpreter ABI. On entry the arguments match those
 * of a quick call:
 * x0 = method being called/to bridge to.
 * x1..x7, d0..d7 = arguments to that method.
 */
ENTRY art_quick_to_interpreter_bridge
    SETUP_SAVE_REFS_AND_ARGS_FRAME         // Set up frame and save arguments.

    //  x0 will contain mirror::ArtMethod* method.
    //  x0 本身就是mirror::ArtMethod
    
    //  x1 存储Thread指针，xSELF为x19平台寄存器的别名
    mov x1, xSELF                          // How to get Thread::Current() ???
    //  x2 存储堆栈指针
    mov x2, sp

    // uint64_t artQuickToInterpreterBridge(mirror::ArtMethod* method, Thread* self,
    //                                      mirror::ArtMethod** sp)
    bl   artQuickToInterpreterBridge

    RESTORE_SAVE_REFS_AND_ARGS_FRAME       // TODO: no need to restore arguments in this case.
    REFRESH_MARKING_REGISTER

    fmov d0, x0

    RETURN_OR_DELIVER_PENDING_EXCEPTION
END art_quick_to_interpreter_bridge
```



### SETUP_SAVE_REFS_AND_ARGS_FRAME



1.调用LOAD_RUNTIME_INSTANCE 将runtime::instance_对象加载到x16寄存器中

2.读取callee_save_methods_到寄存器内

3.开辟栈帧（其实就是减sp寄存器的值）

4.保存寄存器d0 ~ d7, x1 ~ x7, x20 ~ x30 到[sp + 16 , sp + 224]

5.将被调用方存入堆栈中

6.将sp的地址存入到Thread::Current()->top_quick_frame变量中

``` c++
    /*
     * Macro that sets up the callee save frame to conform with
     * Runtime::CreateCalleeSaveMethod(kSaveRefsAndArgs).
     *
     * TODO This is probably too conservative - saving FP & LR.
     */
.macro SETUP_SAVE_REFS_AND_ARGS_FRAME
    // #1
    // art::Runtime* xIP0 = art::Runtime::instance_;
    // Our registers aren't intermixed - just spill in order.
    LOAD_RUNTIME_INSTANCE xIP0

    // #2
    // ArtMethod* xIP0 = Runtime::instance_->callee_save_methods_[kSaveRefAndArgs];
    ldr xIP0, [xIP0, RUNTIME_SAVE_REFS_AND_ARGS_METHOD_OFFSET]
	// #3
    // Note: FRAME_SIZE_SAVE_REFS_AND_ARGS == 224
    INCREASE_FRAME FRAME_SIZE_SAVE_REFS_AND_ARGS
    // #4
    SETUP_SAVE_REFS_AND_ARGS_FRAME_INTERNAL sp
	// #5
    str xIP0, [sp]    // Store ArtMethod* Runtime::callee_save_methods_[kSaveRefsAndArgs].
    // Place sp in Thread::Current()->top_quick_frame.
    // #6
    mov xIP0, sp
    str xIP0, [xSELF, # THREAD_TOP_QUICK_FRAME_OFFSET]
.endm
```



#### 虚拟机栈



虚拟机栈

![image-20250913213659699](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250913213659699.png)



### artQuickToInterpreterBridge

art/runtime/entrypoints/quick/quick_trampoline_entrypoints.cc

``` c++

NO_STACK_PROTECTOR
extern "C" uint64_t artQuickToInterpreterBridge(ArtMethod* method, Thread* self, ArtMethod** sp)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  // Ensure we don't get thread suspension until the object arguments are safely in the shadow
  // frame.
  // 看着像是用于测试的代码
  ScopedQuickEntrypointChecks sqec(self);

  // #1 前置判断逻辑，确保method不是abstract，如果是abstract直接抛异常 
  if (UNLIKELY(!method->IsInvokable())) {
    method->ThrowInvocationTimeError(
        method->IsStatic()
            ? nullptr
            : QuickArgumentVisitor::GetThisObjectReference(sp)->AsMirrorPtr());
    return 0;
  }
  // #2 确保方法不是native方法
  DCHECK(!method->IsNative()) << method->PrettyMethod();

  JValue result;
  // #3 获取非proxy的方法对象  
  ArtMethod* non_proxy_method = method->GetInterfaceMethodIfProxy(kRuntimePointerSize);
  DCHECK(non_proxy_method->GetCodeItem() != nullptr) << method->PrettyMethod();
  std::string_view shorty = non_proxy_method->GetShortyView();

  ManagedStack fragment;
  // TODO 不太懂，这里是做了deoptimize吗。
  ShadowFrame* deopt_frame = self->MaybePopDeoptimizedStackedShadowFrame();
  if (UNLIKELY(deopt_frame != nullptr)) { // deoptimze分支
    HandleDeoptimization(&result, method, deopt_frame, &fragment);
  } else {
     //#4 创建解解释器的shadow frame
    CodeItemDataAccessor accessor(non_proxy_method->DexInstructionData());
    const char* old_cause = self->StartAssertNoThreadSuspension(
        "Building interpreter shadow frame");
    uint16_t num_regs = accessor.RegistersSize();
    // No last shadow coming from quick.
    ShadowFrameAllocaUniquePtr shadow_frame_unique_ptr =
        CREATE_SHADOW_FRAME(num_regs, method, /* dex_pc= */ 0);
    ShadowFrame* shadow_frame = shadow_frame_unique_ptr.get();
    size_t first_arg_reg = accessor.RegistersSize() - accessor.InsSize();
    BuildQuickShadowFrameVisitor shadow_frame_builder(
        sp, method->IsStatic(), shorty, shadow_frame, first_arg_reg);
    shadow_frame_builder.VisitArguments();
    self->EndAssertNoThreadSuspension(old_cause);

    // Potentially run <clinit> before pushing the shadow frame. We do not want
    // to have the called method on the stack if there is an exception.
    // #5 确保类初始化  
    if (!EnsureInitialized(self, shadow_frame)) {
      DCHECK(self->IsExceptionPending());
      return 0;
    }

    // Push a transition back into managed code onto the linked list in thread.
    // #6 将managed code推入thread内一个linkedList数据结构里面  
    self->PushManagedStackFragment(&fragment);
    self->PushShadowFrame(shadow_frame);
    // #7 调用解释器进行解释执行
    result = interpreter::EnterInterpreterFromEntryPoint(self, accessor, shadow_frame);
  }

  // Pop transition.
  self->PopManagedStackFragment(fragment);

  // Check if caller needs to be deoptimized for instrumentation reasons.
  instrumentation::Instrumentation* instr = Runtime::Current()->GetInstrumentation();
  if (UNLIKELY(instr->ShouldDeoptimizeCaller(self, sp))) {
    ArtMethod* caller = QuickArgumentVisitor::GetOuterMethod(sp);
    uintptr_t caller_pc = QuickArgumentVisitor::GetCallingPc(sp);
    DCHECK(Runtime::Current()->IsAsyncDeoptimizeable(caller, caller_pc));
    DCHECK(caller != nullptr);
    DCHECK(self->GetException() != Thread::GetDeoptimizationException());
    // Push the context of the deoptimization stack so we can restore the return value and the
    // exception before executing the deoptimized frames.
    self->PushDeoptimizationContext(result,
                                    shorty[0] == 'L' || shorty[0] == '[',  // class or array
                                    self->GetException(),
                                    /* from_code= */ false,
                                    DeoptimizationMethodType::kDefault);

    // Set special exception to cause deoptimization.
    self->SetException(Thread::GetDeoptimizationException());
  }

  // No need to restore the args since the method has already been run by the interpreter.
  // #8 获取代码的执行结果 
  return NanBoxResultIfNeeded(result.GetJ(), shorty[0]);
}
```



#### CodeItemDataAccessor



#### ShadowFrame

用于在解释器中函数间传参，





#### ManagedStack



#### EnterInterpreterFromEntryPoint

逻辑上没有啥东西

```c++

NO_STACK_PROTECTOR
JValue EnterInterpreterFromEntryPoint(Thread* self, const CodeItemDataAccessor& accessor,
                                      ShadowFrame* shadow_frame) {
  DCHECK_EQ(self, Thread::Current());
  bool implicit_check = Runtime::Current()->GetImplicitStackOverflowChecks();
  if (UNLIKELY(__builtin_frame_address(0) < self->GetStackEndForInterpreter(implicit_check))) {
    ThrowStackOverflowError(self);
    return JValue();
  }

  jit::Jit* jit = Runtime::Current()->GetJit();
  if (jit != nullptr) {
    jit->NotifyCompiledCodeToInterpreterTransition(self, shadow_frame->GetMethod());
  }
  return Execute(self, accessor, *shadow_frame, JValue());
}
```



#### Execute

这里就一个注意点，执行过程中如果发现代码已经被jit了会通过调用ArtInterpreterToCompiledCodeBridge执行jit的代码

```c++
NO_STACK_PROTECTOR
static inline JValue Execute(
    Thread* self,
    const CodeItemDataAccessor& accessor,
    ShadowFrame& shadow_frame,
    JValue result_register,
    bool stay_in_interpreter = false,
    bool from_deoptimize = false) REQUIRES_SHARED(Locks::mutator_lock_) {
  DCHECK(!shadow_frame.GetMethod()->IsAbstract());
  DCHECK(!shadow_frame.GetMethod()->IsNative());

  // We cache the result of NeedsDexPcEvents in the shadow frame so we don't need to call
  // NeedsDexPcEvents on every instruction for better performance. NeedsDexPcEvents only gets
  // updated asynchronoulsy in a SuspendAll scope and any existing shadow frames are updated with
  // new value. So it is safe to cache it here.
  shadow_frame.SetNotifyDexPcMoveEvents(
      Runtime::Current()->GetInstrumentation()->NeedsDexPcEvents(shadow_frame.GetMethod(), self));

  if (LIKELY(!from_deoptimize)) {  // Entering the method, but not via deoptimization.
    if (kIsDebugBuild) {
      CHECK_EQ(shadow_frame.GetDexPC(), 0u);
      self->AssertNoPendingException();
    }
    ArtMethod *method = shadow_frame.GetMethod();

    // If we can continue in JIT and have JITed code available execute JITed code.
    // 执行前判断一下代码有没有被jit,如果被jit了则直接调用jit 的 bridge方法。  
    if (!stay_in_interpreter &&
        !self->IsForceInterpreter() &&
        !shadow_frame.GetForcePopFrame() &&
        !shadow_frame.GetNotifyDexPcMoveEvents()) {
      jit::Jit* jit = Runtime::Current()->GetJit();
      if (jit != nullptr) {
        jit->MethodEntered(self, shadow_frame.GetMethod());
        if (jit->CanInvokeCompiledCode(method)) {
          JValue result;

          // Pop the shadow frame before calling into compiled code.
          self->PopShadowFrame();
          // Calculate the offset of the first input reg. The input registers are in the high regs.
          // It's ok to access the code item here since JIT code will have been touched by the
          // interpreter and compiler already.
          uint16_t arg_offset = accessor.RegistersSize() - accessor.InsSize();
          ArtInterpreterToCompiledCodeBridge(self, nullptr, &shadow_frame, arg_offset, &result);
          // Push the shadow frame back as the caller will expect it.
          self->PushShadowFrame(&shadow_frame);

          return result;
        }
      }
    }

    instrumentation::Instrumentation* instrumentation = Runtime::Current()->GetInstrumentation();
    if (UNLIKELY(instrumentation->HasMethodEntryListeners() || shadow_frame.GetForcePopFrame())) {
      instrumentation->MethodEnterEvent(self, method);
      if (UNLIKELY(shadow_frame.GetForcePopFrame())) {
        // The caller will retry this invoke or ignore the result. Just return immediately without
        // any value.
        DCHECK(Runtime::Current()->AreNonStandardExitsEnabled());
        JValue ret = JValue();
        PerformNonStandardReturn(self,
                                 shadow_frame,
                                 ret,
                                 instrumentation,
                                 accessor.InsSize(),
                                 /* unlock_monitors= */ false);
        return ret;
      }
      if (UNLIKELY(self->IsExceptionPending())) {
        instrumentation->MethodUnwindEvent(self,
                                           method,
                                           0);
        JValue ret = JValue();
        if (UNLIKELY(shadow_frame.GetForcePopFrame())) {
          DCHECK(Runtime::Current()->AreNonStandardExitsEnabled());
          PerformNonStandardReturn(self,
                                   shadow_frame,
                                   ret,
                                   instrumentation,
                                   accessor.InsSize(),
                                   /* unlock_monitors= */ false);
        }
        return ret;
      }
    }
  }

  ArtMethod* method = shadow_frame.GetMethod();

  DCheckStaticState(self, method);

  // Lock counting is a special version of accessibility checks, and for simplicity and
  // reduction of template parameters, we gate it behind access-checks mode.
  DCHECK_IMPLIES(method->SkipAccessChecks(), !method->MustCountLocks());

  VLOG(interpreter) << "Interpreting " << method->PrettyMethod();
  // 进入switch interpretor开始解释执行	
  return ExecuteSwitch(
      self, accessor, shadow_frame, result_register, /*interpret_one_instruction=*/ false);
}
```



#### ExecuteSwitch



``` c++
NO_STACK_PROTECTOR
static JValue ExecuteSwitch(Thread* self,
                            const CodeItemDataAccessor& accessor,
                            ShadowFrame& shadow_frame,
                            JValue result_register,
                            bool interpret_one_instruction) REQUIRES_SHARED(Locks::mutator_lock_) {
  if (Runtime::Current()->IsActiveTransaction()) {
    return ExecuteSwitchImpl<true>(
        self, accessor, shadow_frame, result_register, interpret_one_instruction);
  } else {
    return ExecuteSwitchImpl<false>(
        self, accessor, shadow_frame, result_register, interpret_one_instruction);
  }
}
```





#### ExecuteSwitchImpl



``` c++
template<bool transaction_active>
ALWAYS_INLINE JValue ExecuteSwitchImpl(Thread* self,
                                       const CodeItemDataAccessor& accessor,
                                       ShadowFrame& shadow_frame,
                                       JValue result_register,
                                       bool interpret_one_instruction)
  REQUIRES_SHARED(Locks::mutator_lock_) {
  SwitchImplContext ctx {
    .self = self,
    .accessor = accessor,
    .shadow_frame = shadow_frame,
    .result_register = result_register,
    .interpret_one_instruction = interpret_one_instruction,
    .result = JValue(),
  };
  void* impl = reinterpret_cast<void*>(&ExecuteSwitchImplCpp<transaction_active>);
  const uint16_t* dex_pc = ctx.accessor.Insns();
  ExecuteSwitchImplAsm(&ctx, impl, dex_pc);
  return ctx.result;
}
```



#### ExecuteSwitchImplAsm



```asm
// Wrap ExecuteSwitchImpl in assembly method which specifies DEX PC for unwinding.
//  Argument 0: x0: The context pointer for ExecuteSwitchImpl.
//  Argument 1: x1: Pointer to the templated ExecuteSwitchImpl to call.
//  Argument 2: x2: The value of DEX PC (memory address of the methods bytecode).
ENTRY ExecuteSwitchImplAsm
    SAVE_TWO_REGS_INCREASE_FRAME x19, xLR, 16
    mov x19, x2                                   // x19 = DEX PC
    CFI_DEFINE_DEX_PC_WITH_OFFSET(0 /* x0 */, 19 /* x19 */, 0)
    blr x1                                        // Call the wrapped method.
    RESTORE_TWO_REGS_DECREASE_FRAME x19, xLR, 16
    ret
END ExecuteSwitchImplAsm
```



#### ExecuteSwitchImplCpp



```c++
template<bool transaction_active>
NO_STACK_PROTECTOR
void ExecuteSwitchImplCpp(SwitchImplContext* ctx) {
  Thread* self = ctx->self;
  const CodeItemDataAccessor& accessor = ctx->accessor;
  ShadowFrame& shadow_frame = ctx->shadow_frame;
  self->VerifyStack();

  uint32_t dex_pc = shadow_frame.GetDexPC();
  const auto* const instrumentation = Runtime::Current()->GetInstrumentation();
  const uint16_t* const insns = accessor.Insns();
  const Instruction* next = Instruction::At(insns + dex_pc);

  DCHECK(!shadow_frame.GetForceRetryInstruction())
      << "Entered interpreter from invoke without retry instruction being handled!";

  bool const interpret_one_instruction = ctx->interpret_one_instruction;
  while (true) {
    const Instruction* const inst = next;
    dex_pc = inst->GetDexPc(insns);
    shadow_frame.SetDexPC(dex_pc);
    TraceExecution(shadow_frame, inst, dex_pc);
    uint16_t inst_data = inst->Fetch16(0);
    bool exit = false;
    bool success;  // Moved outside to keep frames small under asan.
    if (InstructionHandler<transaction_active, Instruction::kInvalidFormat>(
            ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit).
            Preamble()) {
      DCHECK_EQ(self->IsExceptionPending(), inst->Opcode(inst_data) == Instruction::MOVE_EXCEPTION);
      switch (inst->Opcode(inst_data)) {
#define OPCODE_CASE(OPCODE, OPCODE_NAME, NAME, FORMAT, i, a, e, v)                                \
        case OPCODE: {                                                                            \
          next = inst->RelativeAt(Instruction::SizeInCodeUnits(Instruction::FORMAT));             \
          success = OP_##OPCODE_NAME<transaction_active>(                                         \
              ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit);     \
          if (success && LIKELY(!interpret_one_instruction)) {                                    \
            continue;                                                                             \
          }                                                                                       \
          break;                                                                                  \
        }
  DEX_INSTRUCTION_LIST(OPCODE_CASE)
#undef OPCODE_CASE
      }
    }
    if (exit) {
      shadow_frame.SetDexPC(dex::kDexNoIndex);
      return;  // Return statement or debugger forced exit.
    }
    if (self->IsExceptionPending()) {
      if (!InstructionHandler<transaction_active, Instruction::kInvalidFormat>(
              ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit).
              HandlePendingException()) {
        shadow_frame.SetDexPC(dex::kDexNoIndex);
        return;  // Locally unhandled exception - return to caller.
      }
      // Continue execution in the catch block.
    }
    if (interpret_one_instruction) {
      shadow_frame.SetDexPC(next->GetDexPc(insns));  // Record where we stopped.
      ctx->result = ctx->result_register;
      return;
    }
  }
}  // NOLINT(readability/fn_size)
```



这里有两个点需要额外说明下

1.switch在那？

都在宏定义中DEX_INSTRUCTION_LIST包含了所有的指令实现

```c++
#define OPCODE_CASE(OPCODE, OPCODE_NAME, NAME, FORMAT, i, a, e, v)                                \
        case OPCODE: {                                                                            \
          next = inst->RelativeAt(Instruction::SizeInCodeUnits(Instruction::FORMAT));             \
          success = OP_##OPCODE_NAME<transaction_active>(                                         \
              ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit);     \
          if (success && LIKELY(!interpret_one_instruction)) {                                    \
            continue;                                                                             \
          }                                                                                       \
          break;                                                                                  \
        }
  DEX_INSTRUCTION_LIST(OPCODE_CASE)
#undef OPCODE_CASE
```

2.switch的实现在那？

依旧在宏定义中, 可以发现所有指令的实现逻辑都在InstructionHandler这个类里面

```c++
#define OPCODE_CASE(OPCODE, OPCODE_NAME, NAME, FORMAT, i, a, e, v)                                \
template<bool transaction_active>                                                                 \
ASAN_NO_INLINE NO_STACK_PROTECTOR static bool OP_##OPCODE_NAME(                                   \
    SwitchImplContext* ctx,                                                                       \
    const instrumentation::Instrumentation* instrumentation,                                      \
    Thread* self,                                                                                 \
    ShadowFrame& shadow_frame,                                                                    \
    uint16_t dex_pc,                                                                              \
    const Instruction* inst,                                                                      \
    uint16_t inst_data,                                                                           \
    const Instruction*& next,                                                                     \
    bool& exit) REQUIRES_SHARED(Locks::mutator_lock_) {                                           \
  InstructionHandler<transaction_active, Instruction::FORMAT> handler(                            \
      ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit);             \
  return LIKELY(handler.OPCODE_NAME());                                                           \
}
DEX_INSTRUCTION_LIST(OPCODE_CASE)
#undef OPCODE_CASE
```



### RESTORE_SAVE_REFS_AND_ARGS_FRAME

逻辑其实非常简单，就是恢复所有的寄存器参数

```asm
RESTORE_SAVE_REFS_AND_ARGS_FRAME       // TODO: no need to restore arguments in this case.
REFRESH_MARKING_REGISTER
```



- RESTORE_SAVE_REFS_AND_ARGS_FRAME

```asm
.macro RESTORE_SAVE_REFS_AND_ARGS_FRAME
    RESTORE_SAVE_REFS_AND_ARGS_FRAME_INTERNAL sp
    DECREASE_FRAME FRAME_SIZE_SAVE_REFS_AND_ARGS
.endm
```

很简单的恢复逻辑，将d0~d7, x1 ~ x6, x7, x20~ x30寄存器取值恢复。

```asm
// TODO: Probably no need to restore registers preserved by aapcs64. (That would require
// auditing all users to make sure they restore aapcs64 callee-save registers they clobber.)
.macro RESTORE_SAVE_REFS_AND_ARGS_FRAME_INTERNAL base
    // FP args.
    ldp d0, d1, [\base, #16]
    ldp d2, d3, [\base, #32]
    ldp d4, d5, [\base, #48]
    ldp d6, d7, [\base, #64]

    // Core args.
    ldp x1, x2, [\base, #80]
    ldp x3, x4, [\base, #96]
    ldp x5, x6, [\base, #112]

    // x7, callee-saves and LR.
    // Note: Likewise, we could avoid restoring X20 in the case of Baker
    // read barriers, as it is overwritten by REFRESH_MARKING_REGISTER
    // later; but it's not worth handling this special case.
    ldp x7, x20, [\base, #128]
    .cfi_restore x20
    RESTORE_TWO_REGS_BASE \base, x21, x22, 144
    RESTORE_TWO_REGS_BASE \base, x23, x24, 160
    RESTORE_TWO_REGS_BASE \base, x25, x26, 176
    RESTORE_TWO_REGS_BASE \base, x27, x28, 192
    RESTORE_TWO_REGS_BASE \base, x29, xLR, 208
.endm

```

- REFRESH_MARKING_REGISTER

刷新下x20寄存器, 可能是用于做GC检查，做线程的执行挂起操作的。

```asm
// Macro to refresh the Marking Register (W20).
//
// This macro must be called at the end of functions implementing
// entrypoints that possibly (directly or indirectly) perform a
// suspend check (before they return).
.macro REFRESH_MARKING_REGISTER
#ifdef RESERVE_MARKING_REGISTER
    ldr wMR, [xSELF, #THREAD_IS_GC_MARKING_OFFSET]
#endif
.endm
```





### ret

控制返回，正常的话就是直接返回，如果有异常的话中途会跳转到异常处理函数。

```asm
// #1 将内容拷贝到double寄存器中
fmov d0, x0
// #2 
RETURN_OR_DELIVER_PENDING_EXCEPTION
```

- RETURN_OR_DELIVER_PENDING_EXCEPTION

从线程对象中获取是否有异常信息，若有异常记录则进行异常处理，无异常则直接返回。

```asm
.macro RETURN_OR_DELIVER_PENDING_EXCEPTION
    RETURN_OR_DELIVER_PENDING_EXCEPTION_REG xIP0
.endm

.macro RETURN_OR_DELIVER_PENDING_EXCEPTION_REG reg
	// #1 获取下异常信息
    ldr \reg, [xSELF, # THREAD_EXCEPTION_OFFSET]   // Get exception field.
    // #2 如果有异常信息跳转到异常处理函数，没有就直接return
    cbnz \reg, 1f
    ret
1:
    DELIVER_PENDING_EXCEPTION
.endm
```



- 异常处理

调用了函数artDeliverPendingExceptionFromCode，这里就不做过多的分析了。

```asm
    /*
     * Macro that calls through to artDeliverPendingExceptionFromCode, where the pending
     * exception is Thread::Current()->exception_.
     */
.macro DELIVER_PENDING_EXCEPTION
	// 保存一些寄存器 & 状态 & 环境信息
    SETUP_SAVE_ALL_CALLEE_SAVES_FRAME
    // 调用异常处理函数
    DELIVER_PENDING_EXCEPTION_FRAME_READY
.endm

.macro DELIVER_PENDING_EXCEPTION_FRAME_READY
    mov x0, xSELF

    // Point of no return.
    bl artDeliverPendingExceptionFromCode  // artDeliverPendingExceptionFromCode(Thread*)
    brk 0  // Unreached
.endm
```



## ExecuteNterpImpl


ExecuteNterpImpl在分类上也属于跳转到解释器的跳板函数，只不过不是不再是Switch解释器，而是Nterp解释器

art/runtime/interpreter/mterp/arm64ng/main.S

```asm
OAT_ENTRY ExecuteNterpImpl
    .cfi_startproc
    sub x16, sp, #STACK_OVERFLOW_RESERVED_BYTES
    ldr wzr, [x16]
    /* Spill callee save regs */
    SPILL_ALL_CALLEE_SAVES

    ldr xPC, [x0, #ART_METHOD_DATA_OFFSET_64]
    // Setup the stack for executing the method.
    SETUP_STACK_FRAME xPC, xREFS, xFP, CFI_REFS, load_ins=1

    // Setup the parameters
    cbz w15, .Lxmm_setup_finished

    sub ip2, ip, x15
    ldr w26, [x0, #ART_METHOD_ACCESS_FLAGS_OFFSET]
    lsl x27, ip2, #2 // x27 is now the offset for inputs into the registers array.

    tbz w26, #ART_METHOD_NTERP_ENTRY_POINT_FAST_PATH_FLAG_BIT, .Lsetup_slow_path
    // Setup pointer to inputs in FP and pointer to inputs in REFS
    add x10, xFP, x27
    add x11, xREFS, x27
    mov x12, #0
    SETUP_REFERENCE_PARAMETER_IN_GPR w1, x10, x11, w15, x12, .Lxmm_setup_finished
    SETUP_REFERENCE_PARAMETER_IN_GPR w2, x10, x11, w15, x12, .Lxmm_setup_finished
    SETUP_REFERENCE_PARAMETER_IN_GPR w3, x10, x11, w15, x12, .Lxmm_setup_finished
    SETUP_REFERENCE_PARAMETER_IN_GPR w4, x10, x11, w15, x12, .Lxmm_setup_finished
    SETUP_REFERENCE_PARAMETER_IN_GPR w5, x10, x11, w15, x12, .Lxmm_setup_finished
    SETUP_REFERENCE_PARAMETER_IN_GPR w6, x10, x11, w15, x12, .Lxmm_setup_finished
    SETUP_REFERENCE_PARAMETER_IN_GPR w7, x10, x11, w15, x12, .Lxmm_setup_finished
    add x28, x28, #OFFSET_TO_FIRST_ARGUMENT_IN_STACK
    SETUP_REFERENCE_PARAMETERS_IN_STACK x10, x11, w15, x28, x12
    b .Lxmm_setup_finished

.Lsetup_slow_path:
    // If the method is not static and there is one argument ('this'), we don't need to fetch the
    // shorty.
    tbnz w26, #ART_METHOD_IS_STATIC_FLAG_BIT, .Lsetup_with_shorty
    str w1, [xFP, x27]
    str w1, [xREFS, x27]
    cmp w15, #1
    b.eq .Lxmm_setup_finished

.Lsetup_with_shorty:
    // TODO: Get shorty in a better way and remove below
    SPILL_ALL_ARGUMENTS
    bl NterpGetShorty
    // Save shorty in callee-save xIBASE.
    mov xIBASE, x0
    RESTORE_ALL_ARGUMENTS

    // Setup pointer to inputs in FP and pointer to inputs in REFS
    add x10, xFP, x27
    add x11, xREFS, x27
    mov x12, #0

    add x9, xIBASE, #1  // shorty + 1  ; ie skip return arg character
    tbnz w26, #ART_METHOD_IS_STATIC_FLAG_BIT, .Lhandle_static_method
    add x10, x10, #4
    add x11, x11, #4
    add x28, x28, #4
    b .Lcontinue_setup_gprs
.Lhandle_static_method:
    LOOP_OVER_SHORTY_STORING_GPRS x1, w1, x9, x12, x10, x11, .Lgpr_setup_finished
.Lcontinue_setup_gprs:
    LOOP_OVER_SHORTY_STORING_GPRS x2, w2, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x3, w3, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x4, w4, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x5, w5, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x6, w6, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x7, w7, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_INTs x9, x12, x10, x11, x28, .Lgpr_setup_finished
.Lgpr_setup_finished:
    add x9, xIBASE, #1  // shorty + 1  ; ie skip return arg character
    mov x12, #0  // reset counter
    LOOP_OVER_SHORTY_STORING_FPS d0, s0, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d1, s1, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d2, s2, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d3, s3, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d4, s4, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d5, s5, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d6, s6, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d7, s7, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_FPs x9, x12, x10, x28, .Lxmm_setup_finished
.Lxmm_setup_finished:
    CFI_DEFINE_DEX_PC_WITH_OFFSET(CFI_TMP, CFI_DEX, 0)

    // Set rIBASE
    adr xIBASE, artNterpAsmInstructionStart
    /* start executing the instruction at xPC */
    START_EXECUTING_INSTRUCTIONS
    /* NOTE: no fallthrough */
    // cfi info continues, and covers the whole nterp implementation.
    SIZE ExecuteNterpImpl

%def opcode_pre():

%def fetch_from_thread_cache(dest_reg, miss_label):
   // Fetch some information from the thread cache.
   // Uses ip and ip2 as temporaries.
   add      ip, xSELF, #THREAD_INTERPRETER_CACHE_OFFSET       // cache address
   ubfx     ip2, xPC, #2, #THREAD_INTERPRETER_CACHE_SIZE_LOG2  // entry index
   add      ip, ip, ip2, lsl #4            // entry address within the cache
   ldp      ip, ${dest_reg}, [ip]          // entry key (pc) and value (offset)
   cmp      ip, xPC
   b.ne     ${miss_label}

%def footer():
/*
 * ===========================================================================
 *  Common subroutines and data
 * ===========================================================================
 */

    .text
    .align  2

// Enclose all code below in a symbol (which gets printed in backtraces).
NAME_START nterp_helper

// Note: mterp also uses the common_* names below for helpers, but that's OK
// as the assembler compiled each interpreter separately.
common_errDivideByZero:
    EXPORT_PC
    bl art_quick_throw_div_zero

// Expect index in w1, length in w3.
common_errArrayIndex:
    EXPORT_PC
    mov x0, x1
    mov x1, x3
    bl art_quick_throw_array_bounds

common_errNullObject:
    EXPORT_PC
    bl art_quick_throw_null_pointer_exception

NterpCommonInvokeStatic:
    COMMON_INVOKE_NON_RANGE is_static=1, suffix="invokeStatic"

NterpCommonInvokeStaticRange:
    COMMON_INVOKE_RANGE is_static=1, suffix="invokeStatic"

NterpCommonInvokeInstance:
    COMMON_INVOKE_NON_RANGE suffix="invokeInstance"

NterpCommonInvokeInstanceRange:
    COMMON_INVOKE_RANGE suffix="invokeInstance"

NterpCommonInvokeInterface:
    COMMON_INVOKE_NON_RANGE is_interface=1, suffix="invokeInterface"

NterpCommonInvokeInterfaceRange:
    COMMON_INVOKE_RANGE is_interface=1, suffix="invokeInterface"

NterpCommonInvokePolymorphic:
    COMMON_INVOKE_NON_RANGE is_polymorphic=1, suffix="invokePolymorphic"

NterpCommonInvokePolymorphicRange:
    COMMON_INVOKE_RANGE is_polymorphic=1, suffix="invokePolymorphic"

NterpCommonInvokeCustom:
    COMMON_INVOKE_NON_RANGE is_static=1, is_custom=1, suffix="invokeCustom"

NterpCommonInvokeCustomRange:
    COMMON_INVOKE_RANGE is_static=1, is_custom=1, suffix="invokeCustom"

NterpHandleStringInit:
   COMMON_INVOKE_NON_RANGE is_string_init=1, suffix="stringInit"

NterpHandleStringInitRange:
   COMMON_INVOKE_RANGE is_string_init=1, suffix="stringInit"

NterpHandleHotnessOverflow:
    CHECK_AND_UPDATE_SHARED_MEMORY_METHOD if_hot=1f, if_not_hot=5f
1:
    mov x1, xPC
    mov x2, xFP
    bl nterp_hot_method
    cbnz x0, 3f
2:
    FETCH wINST, 0                      // load wINST
    GET_INST_OPCODE ip                  // extract opcode from wINST
    GOTO_OPCODE ip                      // jump to next instruction
3:
    // Drop the current frame.
    ldr ip, [xREFS, #-8]
    mov sp, ip
    .cfi_def_cfa sp, CALLEE_SAVES_SIZE

    // The transition frame of type SaveAllCalleeSaves saves x19 and x20,
    // but not managed ABI. So we need to restore callee-saves of the nterp frame,
    // and save managed ABI callee saves, which will be restored by the callee upon
    // return.
    RESTORE_ALL_CALLEE_SAVES
    INCREASE_FRAME ((CALLEE_SAVES_SIZE) - 16)

    // FP callee-saves
    stp d8, d9, [sp, #0]
    stp d10, d11, [sp, #16]
    stp d12, d13, [sp, #32]
    stp d14, d15, [sp, #48]

    // GP callee-saves.
    SAVE_TWO_REGS x21, x22, 64
    SAVE_TWO_REGS x23, x24, 80
    SAVE_TWO_REGS x25, x26, 96
    SAVE_TWO_REGS x27, x28, 112
    SAVE_TWO_REGS x29, lr, 128

    // Setup the new frame
    ldr x1, [x0, #OSR_DATA_FRAME_SIZE]
    // Given stack size contains all callee saved registers, remove them.
    sub x1, x1, #(CALLEE_SAVES_SIZE - 16)

    // We know x1 cannot be 0, as it at least contains the ArtMethod.

    // Remember CFA in a callee-save register.
    mov xINST, sp
    .cfi_def_cfa_register xINST

    sub sp, sp, x1

    add x2, x0, #OSR_DATA_MEMORY
4:
    sub x1, x1, #8
    ldr ip, [x2, x1]
    str ip, [sp, x1]
    cbnz x1, 4b

    // Fetch the native PC to jump to and save it in a callee-save register.
    ldr xFP, [x0, #OSR_DATA_NATIVE_PC]

    // Free the memory holding OSR Data.
    bl free

    // Jump to the compiled code.
    br xFP
5:
    DO_SUSPEND_CHECK continue_label=2b
    b 2b

// This is the logical end of ExecuteNterpImpl, where the frame info applies.
// EndExecuteNterpImpl includes the methods below as we want the runtime to
// see them as part of the Nterp PCs.
.cfi_endproc

nterp_to_nterp_static_non_range:
    .cfi_startproc
    SETUP_STACK_FOR_INVOKE
    SETUP_NON_RANGE_ARGUMENTS_AND_EXECUTE is_static=1, is_string_init=0
    .cfi_endproc

nterp_to_nterp_string_init_non_range:
    .cfi_startproc
    SETUP_STACK_FOR_INVOKE
    SETUP_NON_RANGE_ARGUMENTS_AND_EXECUTE is_static=0, is_string_init=1
    .cfi_endproc

nterp_to_nterp_instance_non_range:
    .cfi_startproc
    SETUP_STACK_FOR_INVOKE
    SETUP_NON_RANGE_ARGUMENTS_AND_EXECUTE is_static=0, is_string_init=0
    .cfi_endproc

nterp_to_nterp_static_range:
    .cfi_startproc
    SETUP_STACK_FOR_INVOKE
    SETUP_RANGE_ARGUMENTS_AND_EXECUTE is_static=1
    .cfi_endproc

nterp_to_nterp_instance_range:
    .cfi_startproc
    SETUP_STACK_FOR_INVOKE
    SETUP_RANGE_ARGUMENTS_AND_EXECUTE is_static=0
    .cfi_endproc

nterp_to_nterp_string_init_range:
    .cfi_startproc
    SETUP_STACK_FOR_INVOKE
    SETUP_RANGE_ARGUMENTS_AND_EXECUTE is_static=0, is_string_init=1
    .cfi_endproc

NAME_END nterp_helper

// This is the end of PCs contained by the OatQuickMethodHeader created for the interpreter
// entry point.
    .type EndExecuteNterpImpl, #function
    .hidden EndExecuteNterpImpl
    .global EndExecuteNterpImpl
EndExecuteNterpImpl:
```



### 主控制程序

1.保存callee-saved register

2.开辟栈帧

3.参数读取配置

​	a.slow path

​	b.fast path

```asm
.cfi_startproc
// pre stackoverflow检查

// arm64平台上STACK_OVERFLOW_RESERVED_BYTES = 0x4000
sub x16, sp, #STACK_OVERFLOW_RESERVED_BYTES
ldr wzr, [x16] // 测试一下内存是否越界？
/* Spill callee save regs */

// 1. 保存callee saved 寄存器列表
SPILL_ALL_CALLEE_SAVES

// 2.开辟栈帧

// 读取artMethod.data_属性(也就是code item的位置)
ldr xPC, [x0, #ART_METHOD_DATA_OFFSET_64]
// Setup the stack for executing the method.
// 为函数执行开辟栈帧
SETUP_STACK_FRAME xPC, xREFS, xFP, CFI_REFS, load_ins=1

// Setup the parameters
// 3. 参数配置
cbz w15, .Lxmm_setup_finished

sub ip2, ip, x15
ldr w26, [x0, #ART_METHOD_ACCESS_FLAGS_OFFSET]
lsl x27, ip2, #2 // x27 is now the offset for inputs into the registers array.
// 跳转slow path
tbz w26, #ART_METHOD_NTERP_ENTRY_POINT_FAST_PATH_FLAG_BIT, .Lsetup_slow_path
// Setup pointer to inputs in FP and pointer to inputs in REFS
// fast-path 先将参数从w1～ w7中读取，在通过去stack读取
add x10, xFP, x27
add x11, xREFS, x27
mov x12, #0
SETUP_REFERENCE_PARAMETER_IN_GPR w1, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w2, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w3, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w4, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w5, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w6, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w7, x10, x11, w15, x12, .Lxmm_setup_finished
add x28, x28, #OFFSET_TO_FIRST_ARGUMENT_IN_STACK
SETUP_REFERENCE_PARAMETERS_IN_STACK x10, x11, w15, x28, x12
// 参数解析完毕，跳转到下一步
b .Lxmm_setup_finished
```



#### 保存callee-saved registers

SPILL_ALL_CALLEE_SAVES是一个宏定义。用于开辟栈帧，保存callee-saved registers其中过程如下：

1.INCREASE_FRAME分配堆栈空间(修改sp指针)

2.SAVE_ALL_CALLEE_SAVES将d8 ～ d15, x19~ x30寄存器的值保存到#1分配的堆栈空间中

```asm
SPILL_ALL_CALLEE_SAVES

.macro SPILL_ALL_CALLEE_SAVES
	// 分配栈空间 大小为12 * 8 + 8 * 8
	// Note: arm64下CALLEE_SAVES_SIZE 定义在 art/runtime/arch/arm64/asm_support_arm64.h
    INCREASE_FRAME CALLEE_SAVES_SIZE
    // Note: we technically don't need to save x19 and x20,
    // but the runtime will expect those values to be there when unwinding
    // (see Arm64Context::DoLongJump checking for the thread register).
    // 保存d8 ～ d15, x19~ x30
    SAVE_ALL_CALLEE_SAVES 0
.endm
```



#### 开辟栈针

将art方法中的code item设置到xPC(x22)寄存器中，设置xREFS,设置xFP，并加载一个指令保存到w15寄存器中

```asm
ldr xPC, [x0, #ART_METHOD_DATA_OFFSET_64]
// Setup the stack for executing the method.
SETUP_STACK_FRAME xPC, xREFS, xFP, CFI_REFS, load_ins=1
```

核心流程如下：

1.获取代码项信息

2.获取code item信息

3.计算栈帧大小

4.分配新栈帧

5.设置应用表和栈帧指针

6.保存就栈帧指针设置CFI信息

7.清空虚拟寄存器

8.保存ArtMethod

```asm
// Setup the stack to start executing the method. Expects:
// - x0 to contain the ArtMethod
//
// Outputs
// - ip contains the dex registers size
// - x28 contains the old stack pointer.
// - \code_item is replaced with a pointer to the instructions
// - if load_ins is 1, w15 contains the ins
//
// Uses ip, ip2, x12, x13, x14 as temporaries.
.macro SETUP_STACK_FRAME code_item, refs, fp, cfi_refs, load_ins
    FETCH_CODE_ITEM_INFO \code_item, wip, wip2, w15, \load_ins

    // Compute required frame size: ((2 * ip) + ip2) * 4 + 24
    // 24 is for saving the previous frame, pc, and method being executed.
    // 1. 计算需要的栈帧大小：((2 * ip) + ip2) * 4 + 24
    add x14, ip, ip
    add x14, x14, ip2
    lsl x14, x14, #2
    add x14, x14, #24

    // Compute new stack pointer in x14
    // 2. 计算新的堆栈指针值，并做16字节对其
    sub x14, sp, x14
    // Alignment
    and x14, x14, #-16

    // Set reference and dex registers, align to pointer size for previous frame and dex pc.
    // 3. 设置refs和dex registers
    add \refs, x14, ip2, lsl #2
    add \refs, \refs, 28
    and \refs, \refs, #(-__SIZEOF_POINTER__)
    add \fp, \refs, ip, lsl #2

    // Now setup the stack pointer.
    // 4. 设置stack point
    mov x28, sp
    .cfi_def_cfa_register x28
    mov sp, x14
    str x28, [\refs, #-8]
    CFI_DEF_CFA_BREG_PLUS_UCONST \cfi_refs, -8, CALLEE_SAVES_SIZE

    // Put nulls in reference frame.
    cbz ip, 2f
    // 清空虚拟寄存器。
    mov ip2, \refs
1:
    str xzr, [ip2], #8  // May clear vreg[0].
    cmp ip2, \fp
    b.lo 1b
2:
    // Save the ArtMethod.
    // 将artMethod保存到堆栈上
    str x0, [sp]
.endm
```

- 宏定义参数

输入：

x0 指向ArtMethod

输出：

ip (x16寄存器) 包含dex register的大小

x28寄存器中会包含old 堆栈指针

宏定义中code_item(x22寄存器)，会被替换为指向字节码指令的指针

如果宏定义load_ins为1, w15中会包含指令

```asm
// Setup the stack to start executing the method. Expects:
// - x0 to contain the ArtMethod
//
// Outputs
// - ip contains the dex registers size
// - x28 contains the old stack pointer.
// - \code_item is replaced with a pointer to the instructions
// - if load_ins is 1, w15 contains the ins
//
// Uses ip, ip2, x12, x13, x14 as temporaries.
// 宏定义
.macro SETUP_STACK_FRAME code_item, refs, fp, cfi_refs, load_ins


// 调用处
// x22  xPC       interpreted program counter, used for fetching instructions
// x25  xREFS     base of object references of dex registers
// x29  xFP       interpreted frame pointer, used for accessing locals and args
// #define CFI_REFS 25
SETUP_STACK_FRAME xPC, xREFS, xFP, CFI_REFS, load_ins=1
```

- code item 信息获取

```asm
FETCH_CODE_ITEM_INFO \code_item, wip, wip2, w15, \load_ins
// code_item - xPC -> x22寄存器，宏定义执行完成后，最终会存储首个字节码指令的地址
// wip/wip2 -> w16/w17（临时寄存器）w16存储code item寄存器大小，w17存储outs size
// w15 -> 通用寄存器，存储输入寄存器的大小
// load_ins -> 是否需要加载ins
```



```asm
// Uses x12, x13, and x14 as temporaries.
.macro FETCH_CODE_ITEM_INFO code_item, registers, outs, ins, load_ins
	// 如果code item最低位为0跳转到label4
    tbz \code_item, #0, 4f
    // 去除最后一位的1（末尾的值表明他是一个compact dex file）
    and \code_item, \code_item, #-2 // Remove the extra bit that marks it's a compact dex file
    // 获取dex信息
    ldrh w13, [\code_item, #COMPACT_CODE_ITEM_FIELDS_OFFSET]
    ubfx \registers, w13, #COMPACT_CODE_ITEM_REGISTERS_SIZE_SHIFT, #4
    ubfx \outs, w13, #COMPACT_CODE_ITEM_OUTS_SIZE_SHIFT, #4
    .if \load_ins
    ubfx \ins, w13, #COMPACT_CODE_ITEM_INS_SIZE_SHIFT, #4
    .else
    ubfx w14, w13, #COMPACT_CODE_ITEM_INS_SIZE_SHIFT, #4
    add \registers, \registers, w14
    .endif
    ldrh w13, [\code_item, #COMPACT_CODE_ITEM_FLAGS_OFFSET]
    tst w13, #COMPACT_CODE_ITEM_REGISTERS_INS_OUTS_FLAGS
    b.eq 3f
    sub x14, \code_item, #4
    tst w13, #COMPACT_CODE_ITEM_INSNS_FLAG
    csel x14, x14, \code_item, ne

    tbz w13, #COMPACT_CODE_ITEM_REGISTERS_BIT, 1f
    ldrh w12, [x14, #-2]!
    add \registers, \registers, w12
1:
    tbz w13, #COMPACT_CODE_ITEM_INS_BIT, 2f
    ldrh w12, [x14, #-2]!
    .if \load_ins
    add \ins, \ins, w12
    .else
    add \registers, \registers, w12
    .endif
2:
    tbz w13, #COMPACT_CODE_ITEM_OUTS_BIT, 3f
    ldrh w12, [x14, #-2]!
    add \outs, \outs, w12
3:
    .if \load_ins
    add \registers, \registers, \ins
    .endif
    add \code_item, \code_item, #COMPACT_CODE_ITEM_INSNS_OFFSET
    b 5f
4:
    // Fetch dex register size.
    // 通过offset获取dex register size.
    ldrh \registers, [\code_item, #CODE_ITEM_REGISTERS_SIZE_OFFSET]
    // Fetch outs size.
    // 获取outs size大小
    ldrh \outs, [\code_item, #CODE_ITEM_OUTS_SIZE_OFFSET]
    // 获取输入寄存器的大小
    .if \load_ins
    ldrh \ins, [\code_item, #CODE_ITEM_INS_SIZE_OFFSET]
    .endif
    // 获取code_item的代码地址
    add \code_item, \code_item, #CODE_ITEM_INSNS_OFFSET
5:
.endm
```

- 计算新堆栈大小

```asm
// Compute required frame size: ((2 * ip) + ip2) * 4 + 24
// 24 is for saving the previous frame, pc, and method being executed.
// 1. 计算需要的栈帧大小：((2 * ip) + ip2) * 4 + 24
// ip - code item寄存器数目
// ip2 - outs size 
add x14, ip, ip
add x14, x14, ip2
lsl x14, x14, #2
add x14, x14, #24

// Compute new stack pointer in x14
// 2. 计算新的堆栈指针值，并做16字节对其
sub x14, sp, x14
// Alignment
and x14, x14, #-16
```

- 计算参数

```asm
// Set reference and dex registers, align to pointer size for previous frame and dex pc.
// 1. 设置refs和dex registers
add \refs, x14, ip2, lsl #2
add \refs, \refs, 28
and \refs, \refs, #(-__SIZEOF_POINTER__)
add \fp, \refs, ip, lsl #2

// Now setup the stack pointer.
// 2. 设置stack point
mov x28, sp
.cfi_def_cfa_register x28
mov sp, x14
str x28, [\refs, #-8]
CFI_DEF_CFA_BREG_PLUS_UCONST \cfi_refs, -8, CALLEE_SAVES_SIZE

// Put nulls in reference frame.
// 3. 清空虚拟寄存器
cbz ip, 2f
mov ip2, \refs
1:
str xzr, [ip2], #8  // May clear vreg[0].
cmp ip2, \fp
b.lo 1b
2:
// Save the ArtMethod.
// 4. 将artMethod保存到堆栈上
str x0, [sp]
```



#### 参数保存

```asm
 // Setup the parameters
 // 判断前一步，指令首地址是否正常加载。
cbz w15, .Lxmm_setup_finished

sub ip2, ip, x15
// 加载art method access flags
ldr w26, [x0, #ART_METHOD_ACCESS_FLAGS_OFFSET]
// 计算新开辟空间
lsl x27, ip2, #2 // x27 is now the offset for inputs into the registers array.
// 读取access flag fast flag是否设置，没设置则跳转到slowpath
tbz w26, #ART_METHOD_NTERP_ENTRY_POINT_FAST_PATH_FLAG_BIT, .Lsetup_slow_path
// fast path执行逻辑
// Setup pointer to inputs in FP and pointer to inputs in REFS
// 计算首个局部变量的地址，以及首个dex 寄存器的变量地址
add x10, xFP, x27
add x11, xREFS, x27
mov x12, #0
// 依次处理读取w1 ~ w7参数
SETUP_REFERENCE_PARAMETER_IN_GPR w1, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w2, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w3, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w4, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w5, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w6, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w7, x10, x11, w15, x12, .Lxmm_setup_finished
// 堆栈中读取参数信息
add x28, x28, #OFFSET_TO_FIRST_ARGUMENT_IN_STACK
SETUP_REFERENCE_PARAMETERS_IN_STACK x10, x11, w15, x28, x12
```



##### fast path



```asm
lsl x27, ip2, #2 // x27 is now the offset for inputs into the registers array.
// 读取access flag fast flag是否设置，没设置则跳转到slowpath
tbz w26, #ART_METHOD_NTERP_ENTRY_POINT_FAST_PATH_FLAG_BIT, .Lsetup_slow_path
// fast path执行逻辑
// Setup pointer to inputs in FP and pointer to inputs in REFS
// 计算首个局部变量的地址，以及首个dex 寄存器的变量地址
add x10, xFP, x27
add x11, xREFS, x27
mov x12, #0
// 依次处理读取w1 ~ w7参数
SETUP_REFERENCE_PARAMETER_IN_GPR w1, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w2, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w3, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w4, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w5, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w6, x10, x11, w15, x12, .Lxmm_setup_finished
SETUP_REFERENCE_PARAMETER_IN_GPR w7, x10, x11, w15, x12, .Lxmm_setup_finished
// 堆栈中读取参数信息
add x28, x28, #OFFSET_TO_FIRST_ARGUMENT_IN_STACK
SETUP_REFERENCE_PARAMETERS_IN_STACK x10, x11, w15, x28, x12
```



简单来说就是被特定的通用寄存器保存到[x11, offset], [x15, offset]位置

```asm
// gpr32 - 存储函数参数的通用寄存器
// regs  - 参数首地址
// ins   - 输入寄存器数
// arg_offset - 参数偏移量
// finished   - 完成label
.macro SETUP_REFERENCE_PARAMETER_IN_GPR gpr32, regs, refs, ins, arg_offset, finished
	// 保存内容到regs中 
    str \gpr32, [\regs, \arg_offset]
    // 递减待处理的寄存器总数
    sub \ins, \ins, #1
    // 保存内容到refs
    str \gpr32, [\refs, \arg_offset]
    // 更新args offset
    add \arg_offset, \arg_offset, #4
    // 判断是否完成
    cbz \ins, \finished
.endm
```



##### slow path 

```asm
.Lsetup_slow_path:
	// 1. 单独处理空参方法，迅速返回
		
    // If the method is not static and there is one argument ('this'), we don't need to fetch the
    // shorty.
    // 判断access flag中的ART_METHOD_IS_STATIC_FLAG_BIT 比特位是否为1
    // 第一次check 方法类型
    tbnz w26, #ART_METHOD_IS_STATIC_FLAG_BIT, .Lsetup_with_shorty
    str w1, [xFP, x27] // 保存this到堆栈中
    str w1, [xREFS, x27] // 保存this到对象引用数组中
    cmp w15, #1    // 判断输入参数总数是否为1
    b.eq .Lxmm_setup_finished // 如果输入参数为1，由于地一个参数已经处理过了，因此直接跳转到finish label
	
	// 处理非空参方法
.Lsetup_with_shorty:
    // TODO: Get shorty in a better way and remove below
    
    // 2. 读取shorty 描述符
    SPILL_ALL_ARGUMENTS // 保存x0~x7, d0 ~ d7寄存器到堆栈指针中，并更新sp = sp - 128
    bl NterpGetShorty // 获取shorty描述符
    // Save shorty in callee-save xIBASE.
    mov xIBASE, x0 // 将shorty 描述符保存到xIBASE(x24)中
    RESTORE_ALL_ARGUMENTS // 对应于SPILL_ALL_ARGUMENTS，恢复保存的寄存器

    // Setup pointer to inputs in FP and pointer to inputs in REFS
    // 计算fp和refs寄存器集合的位置。
    add x10, xFP, x27 // 计算堆栈位置保存到x10寄存器中
    add x11, xREFS, x27 // 计算引用数组的位置并保存到x11寄存器中
    mov x12, #0
	// 跳过shorty的首个字符，也就是返回值
    add x9, xIBASE, #1  // shorty + 1  ; ie skip return arg character
    
    // 判断方法类型是否是static方法
    // 第二次check方法类型
    tbnz w26, #ART_METHOD_IS_STATIC_FLAG_BIT, .Lhandle_static_method
    
    // 普通成员方法，由于第一个参数已经处理过了，跳过第一个参数 
    add x10, x10, #4 // 处理下一个堆栈
    add x11, x11, #4 // 处理下一个引用数组
    add x28, x28, #4 
    // 继续处理后续的参数
    b .Lcontinue_setup_gprs
.Lhandle_static_method:
    LOOP_OVER_SHORTY_STORING_GPRS x1, w1, x9, x12, x10, x11, .Lgpr_setup_finished
.Lcontinue_setup_gprs:
    LOOP_OVER_SHORTY_STORING_GPRS x2, w2, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x3, w3, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x4, w4, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x5, w5, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x6, w6, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_SHORTY_STORING_GPRS x7, w7, x9, x12, x10, x11, .Lgpr_setup_finished
    LOOP_OVER_INTs x9, x12, x10, x11, x28, .Lgpr_setup_finished
.Lgpr_setup_finished:
    add x9, xIBASE, #1  // shorty + 1  ; ie skip return arg character
    mov x12, #0  // reset counter
    LOOP_OVER_SHORTY_STORING_FPS d0, s0, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d1, s1, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d2, s2, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d3, s3, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d4, s4, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d5, s5, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d6, s6, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_SHORTY_STORING_FPS d7, s7, x9, x12, x10, .Lxmm_setup_finished
    LOOP_OVER_FPs x9, x12, x10, x28, .Lxmm_setup_finished
```

根据arg的类型int/long/object做一次loop将所有参数内容保存到栈针中

```asm
// Puts the next int/long/object parameter passed in physical register
// in the expected dex register array entry, and in case of object in the
// expected reference array entry.
.macro LOOP_OVER_SHORTY_STORING_GPRS gpr_64, gpr_32, shorty, arg_offset, regs, refs, finished
1: // LOOP
    ldrb wip, [\shorty], #1       // Load next character in shorty, and increment.
    cbz wip, \finished            // if (wip == '\0') goto finished
    cmp wip, #74                  // if (wip == 'J') goto FOUND_LONG
    b.eq 2f
    cmp wip, #70                  // if (wip == 'F') goto SKIP_FLOAT
    b.eq 3f
    cmp wip, #68                  // if (wip == 'D') goto SKIP_DOUBLE
    b.eq 4f
    str \gpr_32, [\regs, \arg_offset]
    cmp wip, #76                  // if (wip != 'L') goto NOT_REFERENCE
    b.ne 6f
    str \gpr_32, [\refs, \arg_offset]
6:  // NOT_REFERENCE
    add \arg_offset, \arg_offset, #4
    b 5f
2:  // FOUND_LONG
    str \gpr_64, [\regs, \arg_offset]
    add \arg_offset, \arg_offset, #8
    b 5f
3:  // SKIP_FLOAT
    add \arg_offset, \arg_offset, #4
    b 1b
4:  // SKIP_DOUBLE
    add \arg_offset, \arg_offset, #8
    b 1b
5:
.endm
```

### Lxmm_setup_finished

1. 将PC寄存器作为基地址获取解释器指令集合基地址
2. 解释执行xPC(x22)寄存器中的davik 指令

```asm
.Lxmm_setup_finished:
    CFI_DEFINE_DEX_PC_WITH_OFFSET(CFI_TMP, CFI_DEX, 0)

    // Set rIBASE
    // 关于xIBASE寄存器介绍如下：解释器指令基地址，用于计算goto跳转位置
    // x24  xIBASE    interpreted instruction base pointer, used for computed goto
    // 1. 将PC寄存器作为基地址获取解释器指令集合基地址
    adr xIBASE, artNterpAsmInstructionStart
    /* start executing the instruction at xPC */
    // 2. 解释执行xPC(x22)寄存器中的davik 指令
    START_EXECUTING_INSTRUCTIONS
    // 
    /* NOTE: no fallthrough */
    // cfi info continues, and covers the whole nterp implementation.
    SIZE ExecuteNterpImpl
```



### START_EXECUTING_INSTRUCTIONS

增加method热度，并在执行方法前做suspend check 

```asm
// Increase method hotness and do suspend check before starting executing the method.
.macro START_EXECUTING_INSTRUCTIONS
	// 1. 加载hotness 取值
    ldr x0, [sp]
    ldrh w2, [x0, #ART_METHOD_HOTNESS_COUNT_OFFSET]
#if (NTERP_HOTNESS_VALUE != 0)
#error Expected 0 for hotness value
#endif
    // If the counter is at zero, handle this in the runtime.
    // 处理hotness的取值
    cbz w2, 3f
    // 累计hotness
    add x2, x2, #-1
    strh w2, [x0, #ART_METHOD_HOTNESS_COUNT_OFFSET]
1:
    DO_SUSPEND_CHECK continue_label=2f
2:
    FETCH_INST
    GET_INST_OPCODE ip
    GOTO_OPCODE ip
3:
    CHECK_AND_UPDATE_SHARED_MEMORY_METHOD if_hot=4f, if_not_hot=1b
4:
    mov x1, xzr
    mov x2, xFP
    bl nterp_hot_method
    b 2b
.endm
```



核心逻辑：获取指令存入wINST，从指令中取出opcode，使用br跳转执行特定指令

- CHECK_AND_UPDATE_SHARED_MEMORY_METHOD

检查函数的是否是hot函数，执行跳转：

一是通过art method access flag检查线程热度，如果是hot则跳转到hot label中去

二是通过thread检查线程热度，如果是hot则进行跳转

如果一、二均不满足则直接跳转到not hot label中去。

```asm
.macro CHECK_AND_UPDATE_SHARED_MEMORY_METHOD if_hot, if_not_hot
	// 获取art method access flag
    ldr wip, [x0, #ART_METHOD_ACCESS_FLAGS_OFFSET]
    // 获取bit位是否被设置，如果被设置跳转到if_hot label中去
    tbz wip, #ART_METHOD_IS_MEMORY_SHARED_FLAG_BIT, \if_hot
    // 第二次check,从thread对象中获取
    ldr wip, [xSELF, #THREAD_SHARED_METHOD_HOTNESS_OFFSET]
    cbz wip, \if_hot
    // hotness递减
    add wip, wip, #-1
    // 保存hotness值
    str wip, [xSELF, #THREAD_SHARED_METHOD_HOTNESS_OFFSET]
    // 跳转到if_not_hot label中去
    b \if_not_hot
.endm
```

- DO_SUSPEND_CHECK

通过读取线程的flag判断，在执行方法前做挂起检测。

```asm
.macro DO_SUSPEND_CHECK continue_label
	// 获取线程标记为
    ldr wip, [xSELF, #THREAD_FLAGS_OFFSET]
    // check线程标记位是否是挂起 or checkpoint请求
    tst wip, #THREAD_SUSPEND_OR_CHECKPOINT_REQUEST
    // 如果flag没有设置则直接执行。
    b.eq \continue_label
    // 保存一下指令地址
    EXPORT_PC
    // 跳转到art_quick_test_suspend测试挂起
    bl    art_quick_test_suspend
.endm
```

- nterp_hot_method

nterp_hot_method是是通过下方的宏定义生产的。最终会跳转到NterpHotMethod中去

```asm
NTERP_TRAMPOLINE nterp_hot_method, NterpHotMethod

.macro NTERP_TRAMPOLINE name, helper
ENTRY \name
  SETUP_SAVE_REFS_ONLY_FRAME // save寄存器
  bl \helper // 
  RESTORE_SAVE_REFS_ONLY_FRAME
  REFRESH_MARKING_REGISTER
  ldr xIP0, [xSELF, # THREAD_EXCEPTION_OFFSET]   // Get exception field.
  cbnz xIP0, nterp_deliver_pending_exception
  ret
END \name
.endm
```

- fetch and run opcodes

读取一个虚拟机指令保存到x16寄存器中，提取opcode保存到x16中，计算opcode handler位置并执行无条件跳转。

``` asm
// 读取一个虚拟机指令保存到wINST(x23)寄存器中
FETCH_INST
// 通过与操作，获取虚拟机opcode保存到ip(x16)寄存器中
GET_INST_OPCODE ip
// 通过opcode取值计算opcode handler的地址，并使用br跳转指令执行
GOTO_OPCODE ip
/*
 * Fetch the next instruction from xPC into wINST.  Does not advance xPC.
 */
.macro FETCH_INST
    ldrh    wINST, [xPC]
.endm

/*
 * Put the instruction's opcode field into the specified register.
 */
.macro GET_INST_OPCODE reg
    and     \reg, xINST, #255
.endm

/*
 * Begin executing the opcode in _reg.  Clobbers reg
 */
.macro GOTO_OPCODE reg
    add     \reg, xIBASE, \reg, lsl #${handler_size_bits}
    br      \reg
.endm
```



### opcodes

这部分是用于解释GOTO_OPCODE宏，为什么我们可以通过一个偏移量获取特定指令的实现呢？

```asm
.macro GOTO_OPCODE reg
    add     \reg, xIBASE, \reg, lsl #${handler_size_bits}
    br      \reg
.endm
```

不同于Switch解释器，nterp解释器没有if else的分支，实现逻辑也是非常简单, 把指令的实现紧密排列到一起， 具体可见如下.

每一个指令的实现逻辑是按顺序并排在一起的，并且长度都是固定的。这样只要知道artNterpAsmInstructionStart的地址，我们就可以根据opcode的int值寻找到特定的实现类的地址。

例如如果我们需要执行opcode=0x3, 那么跳转的地址就是artNterpAsmInstructionStart + 3 * NTERP_HANDLER_SIZE, 通过这样的方式就解决switch解释器if else分支预测带来的性能损耗。

```asm
artNterpAsmInstructionStart = .L_op_nop
    .text
.L_op_nop: /* 0x00 */
.L_op_move: /* 0x01 */
.L_op_move_from16: /* 0x02 */
.L_op_move_16: /* 0x03 */
.L_op_move_wide: /* 0x04 */
.L_op_move_wide_from16: /* 0x05 */
.L_op_move_wide_16: /* 0x06 */
.L_op_move_object: /* 0x07 */
// ......
.L_op_const_method_type: /* 0xff */
```





### opcode_pre



### fetch_from_thread_cache



### other



common_errDivideByZero

common_errArrayIndex

common_errNullObject

NterpCommonInvokeStatic

NterpCommonInvokeStaticRange

NterpCommonInvokeInstance

NterpCommonInvokeInstanceRange

NterpCommonInvokeInterface

NterpCommonInvokeInterfaceRange

NterpCommonInvokePolymorphic

NterpCommonInvokePolymorphicRange

NterpCommonInvokeCustom

NterpCommonInvokeCustomRange

NterpHandleStringInit

NterpHandleStringInitRange

NterpHandleHotnessOverflow

1,2,3,4,5



nterp_to_nterp_static_non_range

nterp_to_nterp_string_init_non_range

nterp_to_nterp_instance_non_range

nterp_to_nterp_static_range

nterp_to_nterp_instance_range

nterp_to_nterp_string_init_range





## art_quick_resolution_trampoline



### 基础概念

该方法的存在有几个用途

1.为方便MethodTracing，对entryPoint进行转换

2.除开静态代码块的其余static方法为其进行类初始化操作

3.添加断点对断点处函数进行deoptimized,方便断点能停下

针对static方法进行initialization check,因为方法可能在jit gc过程中可能会被回收掉。

因此需要一个跳板去寻找入口的位置

``` c++
// If we have code but the method needs a class initialization check before calling
// that code, install the resolution stub that will perform the check.
// It will be replaced by the proper entry point by ClassLinker::FixupStaticTrampolines
// after initializing class (see ClassLinker::InitializeClass method).
// Note: this mimics the logic in image_writer.cc that installs the resolution
// stub only if we have compiled code or we can execute nterp, and the method needs a class
// initialization check.
if (aot_code != nullptr || method->IsNative() || CanUseNterp(method)) {
  if (kIsDebugBuild && CanUseNterp(method)) {
    // Adds some test coverage for the nterp clinit entrypoint.
    UpdateEntryPoints(method, interpreter::GetNterpWithClinitEntryPoint());
  } else {
    UpdateEntryPoints(method, GetQuickResolutionStub());
  }
} 
```



### 调用链路分析



#### methodTrace



- 解释

在外部通过Debug.startMethodTrace开启方法插桩的时候，会触发系统对Jit的所有方法进行entryPoint更新。

```c++
void Trace::Start(std::unique_ptr<File>&& trace_file_in,
                  size_t buffer_size,
                  int flags,
                  TraceOutputMode output_mode,
                  TraceMode trace_mode,
                  int interval_us) {
    
  // ......

  if (jit != nullptr) {
    jit->GetCodeCache()->InvalidateAllCompiledCode();
    jit->GetCodeCache()->TransitionToDebuggable();
    jit->GetJitCompiler()->SetDebuggableCompilerOption(true);
  }  
           
  // ......  
                  
}
```

- 调用链路

-> VMDebug_startMethodTracingFd

 -> Trace::Start

  -> JitCodeCache::InvalidateAllCompiledCode

   -> Instrumentation::InitializeMethodsCode

​    -> GetQuickResolutionStub/UpdateEntryPoints





#### static Method PreCheck



- 解释

jit gc可能会导致artMethod declaring class被回收，因此初始化的时候需要修改入口做initialization检查。

```c++
void __attribute__((optnone, noinline, used)) Instrumentation::InitializeMethodsCode(ArtMethod* method, const void* aot_code)
    REQUIRES_SHARED(Locks::mutator_lock_) {
    
  // ......

    
  // Special case if we need an initialization check.
  // The method and its declaring class may be dead when starting JIT GC during managed heap GC.
  if (method->StillNeedsClinitCheckMayBeDead()) {
    // If we have code but the method needs a class initialization check before calling
    // that code, install the resolution stub that will perform the check.
    // It will be replaced by the proper entry point by ClassLinker::FixupStaticTrampolines
    // after initializing class (see ClassLinker::InitializeClass method).
    // Note: this mimics the logic in image_writer.cc that installs the resolution
    // stub only if we have compiled code or we can execute nterp, and the method needs a class
    // initialization check.
    if (aot_code != nullptr || method->IsNative() || CanUseNterp(method)) {
      if (kIsDebugBuild && CanUseNterp(method)) {
        // Adds some test coverage for the nterp clinit entrypoint.
        UpdateEntryPoints(method, interpreter::GetNterpWithClinitEntryPoint());
      } else {
        UpdateEntryPoints(method, GetQuickResolutionStub());
      }
    } else {
      UpdateEntryPoints(method, GetQuickToInterpreterBridge());
    }
    return;
  }
    
  // ......
    
}

// 是否需要在进行initialize检查
inline bool ArtMethod::StillNeedsClinitCheckMayBeDead() {
  if (!NeedsClinitCheckBeforeCall()) {
    return false;
  }
  ObjPtr<mirror::Class> klass = GetDeclaringClassMayBeDead();
  return !klass->IsVisiblyInitialized();
}

// ......
static bool NeedsClinitCheckBeforeCall(uint32_t access_flags) {
    // The class initializer is special as it is invoked during initialization
    // and does not need the check.
    return IsStatic(access_flags) && !IsConstructor(access_flags);
}
```



- 调用路径

-> ClassLinker::LoadClass

 -> LinkCode

  -> Instrumentation::InitializeMethodsCode

   -> GetQuickResolutionStub/UpdateEntryPoints





#### Deoptimize



- 调用路径

-> ClassLinker::LoadClass

 -> LinkCode

  -> Instrumentation::InitializeMethodsCode

   -> GetQuickResolutionStub



### 跳板分析



#### art_quick_resolution_trampolin逻辑分析



art跳板代码如下

``` asm
ENTRY art_quick_resolution_trampoline
    SETUP_SAVE_REFS_AND_ARGS_FRAME
    mov x2, xSELF
    mov x3, sp
    bl artQuickResolutionTrampoline  // (called, receiver, Thread*, SP)
    CFI_REMEMBER_STATE
    cbz x0, 1f
    mov xIP0, x0            // Remember returned code pointer in xIP0.
    ldr x0, [sp, #0]        // artQuickResolutionTrampoline puts called method in *SP.
    RESTORE_SAVE_REFS_AND_ARGS_FRAME
    REFRESH_MARKING_REGISTER
    br xIP0
1:
    CFI_RESTORE_STATE_AND_DEF_CFA sp, FRAME_SIZE_SAVE_REFS_AND_ARGS
    RESTORE_SAVE_REFS_AND_ARGS_FRAME
    DELIVER_PENDING_EXCEPTION
END art_quick_resolution_trampoline
```



- SETUP_SAVE_REFS_AND_ARGS_FRAME  - 

  开辟创建新栈帧，具体可见上方art_quick_to_interpreter_bridge分析

- 调用artQuickResolutionTrampoline方法并依据返回值做反应

  - 若返回值为null -> 抛异常
  - 若返回值不为null -> 通过br跳转到返回值中



#### artQuickResolutionTrampoline逻辑分析



执行过程分为如下几步

1.被调用方法元信息补全(调用类型信息)

2.解析被调用方法的artMethod对象(通过ClassLinker->ResolveMethod从dex中解析method的基础信息)

3.虚方法/接口方法动态分派 实现多态行为，确保子类重写方法能被正确调用。

4.最终入口确定

```c++
// Lazily resolve a method for quick. Called by stub code.
extern "C" const void* artQuickResolutionTrampoline(
    ArtMethod* called, mirror::Object* receiver, Thread* self, ArtMethod** sp)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  // ......

  // Compute details about the called method (avoid GCs)
  // #1 被调用方法元信息补全(调用类型信息)
  // 调用类型信息如下
  ClassLinker* linker = Runtime::Current()->GetClassLinker();
  InvokeType invoke_type;
  MethodReference called_method(nullptr, 0);
  const bool called_method_known_on_entry = !called->IsRuntimeMethod();
  ArtMethod* caller = nullptr;
  if (!called_method_known_on_entry) {
    uint32_t dex_pc;
    caller = QuickArgumentVisitor::GetCallingMethodAndDexPc(sp, &dex_pc);
    called_method.dex_file = caller->GetDexFile();

    {
      CodeItemInstructionAccessor accessor(caller->DexInstructions());
      CHECK_LT(dex_pc, accessor.InsnsSizeInCodeUnits());
      const Instruction& instr = accessor.InstructionAt(dex_pc);
      Instruction::Code instr_code = instr.Opcode();
      bool is_range;
      switch (instr_code) {
        case Instruction::INVOKE_DIRECT:
          invoke_type = kDirect;
          is_range = false;
          break;
        case Instruction::INVOKE_DIRECT_RANGE:
          invoke_type = kDirect;
          is_range = true;
          break;
        case Instruction::INVOKE_STATIC:
          invoke_type = kStatic;
          is_range = false;
          break;
        case Instruction::INVOKE_STATIC_RANGE:
          invoke_type = kStatic;
          is_range = true;
          break;
        case Instruction::INVOKE_SUPER:
          invoke_type = kSuper;
          is_range = false;
          break;
        case Instruction::INVOKE_SUPER_RANGE:
          invoke_type = kSuper;
          is_range = true;
          break;
        case Instruction::INVOKE_VIRTUAL:
          invoke_type = kVirtual;
          is_range = false;
          break;
        case Instruction::INVOKE_VIRTUAL_RANGE:
          invoke_type = kVirtual;
          is_range = true;
          break;
        case Instruction::INVOKE_INTERFACE:
          invoke_type = kInterface;
          is_range = false;
          break;
        case Instruction::INVOKE_INTERFACE_RANGE:
          invoke_type = kInterface;
          is_range = true;
          break;
        default:
          DumpB74410240DebugData(sp);
          LOG(FATAL) << "Unexpected call into trampoline: " << instr.DumpString(nullptr);
          UNREACHABLE();
      }
      called_method.index = (is_range) ? instr.VRegB_3rc() : instr.VRegB_35c();
      VLOG(dex) << "Accessed dex file for invoke " << invoke_type << " "
                << called_method.index;
    }
  } else {
    invoke_type = kStatic;
    called_method.dex_file = called->GetDexFile();
    called_method.index = called->GetDexMethodIndex();
  }
  std::string_view shorty =
      called_method.dex_file->GetMethodShortyView(called_method.GetMethodId());
  RememberForGcArgumentVisitor visitor(sp, invoke_type == kStatic, shorty, &soa);
  visitor.VisitArguments();
  self->EndAssertNoThreadSuspension(old_cause);
  const bool virtual_or_interface = invoke_type == kVirtual || invoke_type == kInterface;
  // Resolve method filling in dex cache.
  // #2 解析被调用方ArtMethod对象，并存入DexCache中  
  if (!called_method_known_on_entry) {
    StackHandleScope<1> hs(self);
    mirror::Object* fake_receiver = nullptr;
    HandleWrapper<mirror::Object> h_receiver(
        hs.NewHandleWrapper(virtual_or_interface ? &receiver : &fake_receiver));
    DCHECK_EQ(caller->GetDexFile(), called_method.dex_file);
    called = linker->ResolveMethod<ClassLinker::ResolveMode::kCheckICCEAndIAE>(
        self, called_method.index, caller, invoke_type);
  }
  const void* code = nullptr;
  if (LIKELY(!self->IsExceptionPending())) {
    // Incompatible class change should have been handled in resolve method.
    CHECK(!called->CheckIncompatibleClassChange(invoke_type))
        << called->PrettyMethod() << " " << invoke_type;
    // #3 动态绑定，对virtual,interface,super调用做一些单独处理实现多态的能力
    if (virtual_or_interface || invoke_type == kSuper) {
      // Refine called method based on receiver for kVirtual/kInterface, and
      // caller for kSuper.
      ArtMethod* orig_called = called;
      if (invoke_type == kVirtual) {
        CHECK(receiver != nullptr) << invoke_type;
        called = receiver->GetClass()->FindVirtualMethodForVirtual(called, kRuntimePointerSize);
      } else if (invoke_type == kInterface) {
        CHECK(receiver != nullptr) << invoke_type;
        called = receiver->GetClass()->FindVirtualMethodForInterface(called, kRuntimePointerSize);
      } else {
        DCHECK_EQ(invoke_type, kSuper);
        CHECK(caller != nullptr) << invoke_type;
        ObjPtr<mirror::Class> ref_class = linker->LookupResolvedType(
            caller->GetDexFile()->GetMethodId(called_method.index).class_idx_, caller);
        if (ref_class->IsInterface()) {
          called = ref_class->FindVirtualMethodForInterfaceSuper(called, kRuntimePointerSize);
        } else {
          called = caller->GetDeclaringClass()->GetSuperClass()->GetVTableEntry(
              called->GetMethodIndex(), kRuntimePointerSize);
        }
      }

      CHECK(called != nullptr) << orig_called->PrettyMethod() << " "
                               << mirror::Object::PrettyTypeOf(receiver) << " "
                               << invoke_type << " " << orig_called->GetVtableIndex();
    }
    // Now that we know the actual target, update .bss entry in oat file, if
    // any.
    // 更新bss段信息，可能是用来优化性能吧。
    if (!called_method_known_on_entry) {
      // We only put non copied methods in the BSS. Putting a copy can lead to an
      // odd situation where the ArtMethod being executed is unrelated to the
      // receiver of the method.
      called = called->GetCanonicalMethod();
      if (invoke_type == kSuper || invoke_type == kInterface || invoke_type == kVirtual) {
        if (called->GetDexFile() == called_method.dex_file) {
          called_method.index = called->GetDexMethodIndex();
        } else {
          called_method.index = called->FindDexMethodIndexInOtherDexFile(
              *called_method.dex_file, called_method.index);
          DCHECK_NE(called_method.index, dex::kDexNoIndex);
        }
      }
      ArtMethod* outer_method = QuickArgumentVisitor::GetOuterMethod(sp);
      MaybeUpdateBssMethodEntry(called, called_method, outer_method);
    }

    // Static invokes need class initialization check but instance invokes can proceed even if
    // the class is erroneous, i.e. in the edge case of escaping instances of erroneous classes.
    bool success = true;
    // 调用前做一下类初始化的检查 
    if (called->StillNeedsClinitCheck()) {
      // Ensure that the called method's class is initialized.
      StackHandleScope<1> hs(soa.Self());
      Handle<mirror::Class> h_called_class = hs.NewHandle(called->GetDeclaringClass());
      success = linker->EnsureInitialized(soa.Self(), h_called_class, true, true);
    }
    if (success) {
      // When the clinit check is at entry of the AOT/nterp code, we do the clinit check
      // before doing the suspend check. To ensure the code sees the latest
      // version of the class (the code doesn't do a read barrier to reduce
      // size), do a suspend check now.
      self->CheckSuspend();
      instrumentation::Instrumentation* instrumentation = Runtime::Current()->GetInstrumentation();
      // Check if we need instrumented code here. Since resolution stubs could suspend, it is
      // possible that we instrumented the entry points after we started executing the resolution
      // stub.
      // #4 解析获取最终的entrypoint地址。
      code = instrumentation->GetMaybeInstrumentedCodeForInvoke(called);
    } else {
      DCHECK(called->GetDeclaringClass()->IsErroneous());
      DCHECK(self->IsExceptionPending());
    }
  }
  CHECK_EQ(code == nullptr, self->IsExceptionPending());
  // Fixup any locally saved objects may have moved during a GC.
  visitor.FixupReferences();
  // Place called method in callee-save frame to be placed as first argument to quick method.
  // 保存到堆栈中的的第一个位置（具体可见SETUP_SAVE_REFS_AND_ARGS_FRAME创建流程）  
  *sp = called;

  return code;
}

```



## art_quick_generic_jni_trampoline



- 基础概念

这个方法和前面的art_jni_dlsym_lookup_stub/art_jni_dlsym_lookup_critical_stub有一些区别

前面的两个方法设置的是artMethod.data, 这个方法设置的是compile_code

- 调用路径

-> LinkCode

 -> Instrumentation::InitializeMethodsCode

  -> GetQuickGenericJniStub

```c++
// art/runtime/instrumentation.cc
void Instrumentation::InitializeMethodsCode(ArtMethod* method, const void* aot_code)
    REQUIRES_SHARED(Locks::mutator_lock_) {
    
      // ......
      // Use instrumentation entrypoints if instrumentation is installed.
      if (UNLIKELY(EntryExitStubsInstalled() || IsForcedInterpretOnly() || IsDeoptimized(method))) {
        UpdateEntryPoints(
            method, method->IsNative() ? GetQuickGenericJniStub() : GetQuickToInterpreterBridge());
        return;
      }
      
      // .....
  
  }
```

- art跳板 

``` asm
ENTRY art_quick_generic_jni_trampoline
    SETUP_SAVE_REFS_AND_ARGS_FRAME_WITH_METHOD_IN_X0

    // Save SP, so we can have static CFI info.
    mov x28, sp
    .cfi_def_cfa_register x28

    mov xIP0, #GENERIC_JNI_TRAMPOLINE_RESERVED_AREA
    sub sp, sp, xIP0

    // prepare for artQuickGenericJniTrampoline call
    // (Thread*, managed_sp, reserved_area)
    //    x0         x1            x2   <= C calling convention
    //  xSELF       x28            sp   <= where they are

    mov x0, xSELF   // Thread*
    mov x1, x28     // SP for the managed frame.
    mov x2, sp      // reserved area for arguments and other saved data (up to managed frame)
    bl artQuickGenericJniTrampoline  // (Thread*, sp)

    // The C call will have registered the complete save-frame on success.
    // The result of the call is:
    //     x0: pointer to native code, 0 on error.
    //     The bottom of the reserved area contains values for arg registers,
    //     hidden arg register and SP for out args for the call.

    // Check for error (class init check or locking for synchronized native method can throw).
    cbz x0, .Lexception_in_native

    // Save the code pointer
    mov xIP0, x0

    // Load parameters from frame into registers.
    ldp x0, x1, [sp]
    ldp x2, x3, [sp, #16]
    ldp x4, x5, [sp, #32]
    ldp x6, x7, [sp, #48]

    ldp d0, d1, [sp, #64]
    ldp d2, d3, [sp, #80]
    ldp d4, d5, [sp, #96]
    ldp d6, d7, [sp, #112]

    // Load hidden arg (x15) for @CriticalNative and SP for out args.
    ldp x15, xIP1, [sp, #128]

    // Apply the new SP for out args, releasing unneeded reserved area.
    mov sp, xIP1

    blr xIP0        // native call.

    // result sign extension is handled in C code
    // prepare for artQuickGenericJniEndTrampoline call
    // (Thread*, result, result_f)
    //    x0       x1       x2        <= C calling convention
    mov x1, x0      // Result (from saved).
    mov x0, xSELF   // Thread register.
    fmov x2, d0     // d0 will contain floating point result, but needs to go into x2

    bl artQuickGenericJniEndTrampoline

    // Pending exceptions possible.
    ldr x2, [xSELF, THREAD_EXCEPTION_OFFSET]
    cbnz x2, .Lexception_in_native

    // Tear down the alloca.
    mov sp, x28

    LOAD_RUNTIME_INSTANCE x1
    ldrb w1, [x1, #RUN_EXIT_HOOKS_OFFSET_FROM_RUNTIME_INSTANCE]
    CFI_REMEMBER_STATE
    cbnz w1, .Lcall_method_exit_hook
.Lcall_method_exit_hook_done:

    // Tear down the callee-save frame.
    .cfi_def_cfa_register sp
    // Restore callee-saves and LR as in `RESTORE_SAVE_REFS_AND_ARGS_FRAME`
    // but do not restore argument registers.
    // Note: Likewise, we could avoid restoring X20 in the case of Baker
    // read barriers, as it is overwritten by REFRESH_MARKING_REGISTER
    // later; but it's not worth handling this special case.
#if (FRAME_SIZE_SAVE_REFS_AND_ARGS != 224)
#error "FRAME_SIZE_SAVE_REFS_AND_ARGS(ARM64) size not as expected."
#endif
    RESTORE_REG x20, 136
    RESTORE_TWO_REGS x21, x22, 144
    RESTORE_TWO_REGS x23, x24, 160
    RESTORE_TWO_REGS x25, x26, 176
    RESTORE_TWO_REGS x27, x28, 192
    RESTORE_TWO_REGS x29, xLR, 208
    // Remove the frame.
    DECREASE_FRAME FRAME_SIZE_SAVE_REFS_AND_ARGS

    REFRESH_MARKING_REGISTER

    // store into fpr, for when it's a fpr return...
    fmov d0, x0
    ret

.Lcall_method_exit_hook:
    CFI_RESTORE_STATE_AND_DEF_CFA x28, FRAME_SIZE_SAVE_REFS_AND_ARGS
    fmov d0, x0
    mov x4, FRAME_SIZE_SAVE_REFS_AND_ARGS
    bl art_quick_method_exit_hook
    b .Lcall_method_exit_hook_done

.Lexception_in_native:
    // Move to x1 then sp to please assembler.
    ldr x1, [xSELF, # THREAD_TOP_QUICK_FRAME_OFFSET]
    add sp, x1, #-1  // Remove the GenericJNI tag.
    bl art_deliver_pending_exception
END art_quick_generic_jni_trampoline
```



### 逻辑分析



1.保存寄存器信息/JNI数据

2.调用artQuickGenericJniTrampoline获取执行地址

3.调用JNI方法

4.执行artQuickGenericJniEndTrampoline

5.收尾

​	a.恢复寄存器状态

​	b.异常处理/执行返回

```asm
ENTRY art_quick_generic_jni_trampoline
    // #1 保存callee saved信息 + args
    SETUP_SAVE_REFS_AND_ARGS_FRAME_WITH_METHOD_IN_X0

    // Save SP, so we can have static CFI info.
    // 将sp存到x28中，for cfi检查？
    mov x28, sp
    .cfi_def_cfa_register x28

	// #2 JNI 保存数据区域大小为5k
    mov xIP0, #GENERIC_JNI_TRAMPOLINE_RESERVED_AREA
    sub sp, sp, xIP0

	// 准备参数调用artQuickGenericJniTrampoline方法
    // prepare for artQuickGenericJniTrampoline call
    // (Thread*, managed_sp, reserved_area)
    //    x0         x1            x2   <= C calling convention
    //  xSELF       x28            sp   <= where they are
	
    mov x0, xSELF   // Thread*
    mov x1, x28     // SP for the managed frame.
    mov x2, sp      // reserved area for arguments and other saved data (up to managed frame)
    bl artQuickGenericJniTrampoline  // (Thread*, sp)

	// 调用后结果处理
    // The C call will have registered the complete save-frame on success.
    // The result of the call is:
    //     x0: pointer to native code, 0 on error.
    //     The bottom of the reserved area contains values for arg registers,
    //     hidden arg register and SP for out args for the call.

    // Check for error (class init check or locking for synchronized native method can throw).
    // 异常检测
    cbz x0, .Lexception_in_native

    // Save the code pointer
    mov xIP0, x0

    // Load parameters from frame into registers.
    // 读取Native参数到寄存器中
    ldp x0, x1, [sp]
    ldp x2, x3, [sp, #16]
    ldp x4, x5, [sp, #32]
    ldp x6, x7, [sp, #48]

    ldp d0, d1, [sp, #64]
    ldp d2, d3, [sp, #80]
    ldp d4, d5, [sp, #96]
    ldp d6, d7, [sp, #112]

    // Load hidden arg (x15) for @CriticalNative and SP for out args.
    ldp x15, xIP1, [sp, #128]

    // Apply the new SP for out args, releasing unneeded reserved area.
    // 修改SP指针，将无需使用的栈内存释放掉
    mov sp, xIP1

	// 执行JNI方法
    blr xIP0        // native call.

	// 准备参数信息，执行artQuickGenericJniEndTrampoline方法
    // result sign extension is handled in C code
    // prepare for artQuickGenericJniEndTrampoline call
    // (Thread*, result, result_f)
    //    x0       x1       x2        <= C calling convention
    mov x1, x0      // Result (from saved).
    mov x0, xSELF   // Thread register.
    fmov x2, d0     // d0 will contain floating point result, but needs to go into x2

    bl artQuickGenericJniEndTrampoline

    // Pending exceptions possible.
    // 判断执行过程中是否有异常信息。
    ldr x2, [xSELF, THREAD_EXCEPTION_OFFSET]
    cbnz x2, .Lexception_in_native

    // Tear down the alloca.
    // 清空frame空间占用
    mov sp, x28
	// check是否需要调用exit hook方法
    LOAD_RUNTIME_INSTANCE x1
    ldrb w1, [x1, #RUN_EXIT_HOOKS_OFFSET_FROM_RUNTIME_INSTANCE]
    CFI_REMEMBER_STATE
    cbnz w1, .Lcall_method_exit_hook
.Lcall_method_exit_hook_done:

    // Tear down the callee-save frame.
    .cfi_def_cfa_register sp
    // Restore callee-saves and LR as in `RESTORE_SAVE_REFS_AND_ARGS_FRAME`
    // but do not restore argument registers.
    // Note: Likewise, we could avoid restoring X20 in the case of Baker
    // read barriers, as it is overwritten by REFRESH_MARKING_REGISTER
    // later; but it's not worth handling this special case.
#if (FRAME_SIZE_SAVE_REFS_AND_ARGS != 224)
#error "FRAME_SIZE_SAVE_REFS_AND_ARGS(ARM64) size not as expected."
#endif
	// 恢复寄存器
    RESTORE_REG x20, 136
    RESTORE_TWO_REGS x21, x22, 144
    RESTORE_TWO_REGS x23, x24, 160
    RESTORE_TWO_REGS x25, x26, 176
    RESTORE_TWO_REGS x27, x28, 192
    RESTORE_TWO_REGS x29, xLR, 208
    // Remove the frame.
    DECREASE_FRAME FRAME_SIZE_SAVE_REFS_AND_ARGS

    REFRESH_MARKING_REGISTER

    // store into fpr, for when it's a fpr return...
    fmov d0, x0
    ret

.Lcall_method_exit_hook:
    CFI_RESTORE_STATE_AND_DEF_CFA x28, FRAME_SIZE_SAVE_REFS_AND_ARGS
    fmov d0, x0
    mov x4, FRAME_SIZE_SAVE_REFS_AND_ARGS
    bl art_quick_method_exit_hook
    b .Lcall_method_exit_hook_done

.Lexception_in_native:
    // Move to x1 then sp to please assembler.
    ldr x1, [xSELF, # THREAD_TOP_QUICK_FRAME_OFFSET]
    add sp, x1, #-1  // Remove the GenericJNI tag.
    bl art_deliver_pending_exception
END art_quick_generic_jni_trampoline
```



#### 信息保存

```asm
// #1 保存callee saved信息 + args
SETUP_SAVE_REFS_AND_ARGS_FRAME_WITH_METHOD_IN_X0

// Save SP, so we can have static CFI info.
// 将sp存到x28中，for cfi检查？
mov x28, sp
.cfi_def_cfa_register x28

// #2 JNI 保存数据区域大小为5k
mov xIP0, #GENERIC_JNI_TRAMPOLINE_RESERVED_AREA
sub sp, sp, xIP0
```





#### artQuickGenericJniTrampoline

准备参数并调用方法

```asm
// 准备参数调用artQuickGenericJniTrampoline方法
// prepare for artQuickGenericJniTrampoline call
// (Thread*, managed_sp, reserved_area)
//    x0         x1            x2   <= C calling convention
//  xSELF       x28            sp   <= where they are

mov x0, xSELF   // Thread*
mov x1, x28     // SP for the managed frame.
mov x2, sp      // reserved area for arguments and other saved data (up to managed frame)
bl artQuickGenericJniTrampoline  // (Thread*, sp)

```



#### JNI Call

我们先读下代码的注释我们能知道几个信息

a.resered area的内存结构(从上到下)

 1. local reference cookie(除开@CriticalNative)
 2. HandlerScope (除开@CriticalNative)
 3. stack args (如果参数无法全部存入registers)
 4. 寄存器参数
 5. 隐藏参数 (for @CriticalNative)
 6. 为native调用保存Sp指针 (指向stack args区域)

b.这个方法的职责为

1. 填充上述的所有字段
1. 进行class初始化的检查(针对static方法)
1. 执行native方法调用
1. 执行同步操作(针对 synchronized方法)
1. 最后将返回值返回给stub

```asm
/*
 * Initializes the reserved area assumed to be directly below `managed_sp` for a native call:
 *
 * On entry, the stack has a standard callee-save frame above `managed_sp`,
 * and the reserved area below it. Starting below `managed_sp`, we reserve space
 * for local reference cookie (not present for @CriticalNative), HandleScope
 * (not present for @CriticalNative) and stack args (if args do not fit into
 * registers). At the bottom of the reserved area, there is space for register
 * arguments, hidden arg (for @CriticalNative) and the SP for the native call
 * (i.e. pointer to the stack args area), which the calling stub shall load
 * to perform the native call. We fill all these fields, perform class init
 * check (for static methods) and/or locking (for synchronized methods) if
 * needed and return to the stub.
 *
 * The return value is the pointer to the native code, null on failure.
 *
 * NO_THREAD_SAFETY_ANALYSIS: Depending on the use case, the trampoline may
 * or may not lock a synchronization object and transition out of Runnable.
 */
 为native call初始化数据，该数据位于managed_sp以下的reserved area
 在进入的时候，我们的堆栈中以及有了一个callee-saved frame(就是前面保存的一堆寄存器)， 这个frame的保存地址是在managed_sp上方，reserved area在managed_sp下方。
 从managed_sp下面开始讲起（也就是reserve space），reserve space保存了
 1. local reference cookie(除开@CriticalNative)
 2. HandlerScope (除开@CriticalNative)
 3. stack args (如果参数无法全部存入registers)
 4. 寄存器参数
 5. 隐藏参数 (for @CriticalNative)
 6. 为native调用保存Sp指针 (指向stack args区域)
 我们需要填充上述的所有字段、进行class初始化的检查(针对static方法)、执行native方法调用、执行同步操作(针对 synchronized方法)、最后将返回值返回给stub(这里应该是文中的二级跳板？)
 返回值是执行native code的指针，如果返回null表面解析失败了
```



JNI的内存分布，和我们之前看到的注释是能对的上的～

```c++
/*
 * Generic JNI frame layout:
 *
 * #-------------------#
 * |                   |
 * | caller method...  |
 * #-------------------#    <--- SP on entry
 * | Return X30/LR     |
 * | X29/FP            |    callee save
 * | X28               |    callee save
 * | X27               |    callee save
 * | X26               |    callee save
 * | X25               |    callee save
 * | X24               |    callee save
 * | X23               |    callee save
 * | X22               |    callee save
 * | X21               |    callee save
 * | X20               |    callee save
 * | X7                |    arg7
 * | X6                |    arg6
 * | X5                |    arg5
 * | X4                |    arg4
 * | X3                |    arg3
 * | X2                |    arg2
 * | X1                |    arg1
 * | D7                |    float arg 8
 * | D6                |    float arg 7
 * | D5                |    float arg 6
 * | D4                |    float arg 5
 * | D3                |    float arg 4
 * | D2                |    float arg 3
 * | D1                |    float arg 2
 * | D0                |    float arg 1
 * | padding           | // 8B
 * | Method*           | <- X0 (Managed frame similar to SaveRefsAndArgs.)
 * #-------------------#
 * | local ref cookie  | // 4B
 * | padding           | // 0B or 4B to align stack args on 8B address
 * #-------------------#
 * | JNI Stack Args    | // Empty if all args fit into registers x0-x7, d0-d7.
 * #-------------------#    <--- SP on native call (1)
 * | Free scratch      |
 * #-------------------#
 * | SP for JNI call   | // Pointer to (1).
 * #-------------------#
 * | Hidden arg        | // For @CriticalNative
 * #-------------------#
 * |                   |
 * | Stack for Regs    |    The trampoline assembly will pop these values
 * |                   |    into registers for native call
 * #-------------------#
 */
```



接着我们看看代码

```c++
extern "C" const void* artQuickGenericJniTrampoline(Thread* self,
                                                    ArtMethod** managed_sp,
                                                    uintptr_t* reserved_area)
    REQUIRES_SHARED(Locks::mutator_lock_) NO_THREAD_SAFETY_ANALYSIS {
  // Note: We cannot walk the stack properly until fixed up below.
  ArtMethod* called = *managed_sp;
  DCHECK(called->IsNative()) << called->PrettyMethod(true);
  Runtime* runtime = Runtime::Current();
  std::string_view shorty = called->GetShortyView();
  bool critical_native = called->IsCriticalNative();
  bool fast_native = called->IsFastNative();
  bool normal_native = !critical_native && !fast_native;

  // Run the visitor and update sp.
  // TODO
  BuildGenericJniFrameVisitor visitor(self,
                                      called->IsStatic(),
                                      critical_native,
                                      shorty,
                                      managed_sp,
                                      reserved_area);
  {
    ScopedAssertNoThreadSuspension sants(__FUNCTION__);
    visitor.VisitArguments();
  }

  // Fix up managed-stack things in Thread. After this we can walk the stack.
  self->SetTopOfStackGenericJniTagged(managed_sp);

  self->VerifyStack();
    
  // ......

  // static代码类加载检查
  if (called->StillNeedsClinitCheck()) {
    // Ensure static method's class is initialized.
    StackHandleScope<1> hs(self);
    Handle<mirror::Class> h_class = hs.NewHandle(called->GetDeclaringClass());
    if (!runtime->GetClassLinker()->EnsureInitialized(self, h_class, true, true)) {
      DCHECK(Thread::Current()->IsExceptionPending()) << called->PrettyMethod();
      return nullptr;  // Report error.
    }
  }

  instrumentation::Instrumentation* instr = Runtime::Current()->GetInstrumentation();
  if (UNLIKELY(instr->HasMethodEntryListeners())) {
    instr->MethodEnterEvent(self, called);
    if (self->IsExceptionPending()) {
      return nullptr;
    }
  }

  // Skip calling `artJniMethodStart()` for @CriticalNative and @FastNative.
  if (LIKELY(normal_native)) {
    // 为synchronized代码执行同步操作
    if (called->IsSynchronized()) {
      ObjPtr<mirror::Object> lock = GetGenericJniSynchronizationObject(self, called);
      DCHECK(lock != nullptr);
      lock->MonitorEnter(self);
      if (self->IsExceptionPending()) {
        return nullptr;  // Report error.
      }
    }
    if (UNLIKELY(self->ReadFlag(ThreadFlag::kMonitorJniEntryExit))) {
      artJniMonitoredMethodStart(self);
    } else {
      artJniMethodStart(self);
    }
  }
  // ......
  // Skip pushing LRT frame for @CriticalNative.
  if (LIKELY(!critical_native)) {
    // Push local reference frame.
    JNIEnvExt* env = self->GetJniEnv();
    DCHECK(env != nullptr);
    uint32_t cookie = bit_cast<uint32_t>(env->PushLocalReferenceFrame());

    // Save the cookie on the stack.
    // 将cookie值保存到reverse area中  
    uint32_t* sp32 = reinterpret_cast<uint32_t*>(managed_sp);
    *(sp32 - 1) = cookie;
  }

  // Retrieve the stored native code.
  // Note that it may point to the lookup stub or trampoline.
  // FIXME: This is broken for @CriticalNative as the art_jni_dlsym_lookup_stub
  // does not handle that case. Calls from compiled stubs are also broken.
  // 获取native代码地址  
  void const* nativeCode = called->GetEntryPointFromJni();

  VLOG(third_party_jni) << "GenericJNI: "
                        << called->PrettyMethod()
                        << " -> "
                        << std::hex << reinterpret_cast<uintptr_t>(nativeCode);

  // Return native code.
  // 返回native code地址  
  return nativeCode;
}
```



接着我们再看看nativeCode是怎么获取的, 其实逻辑非常简单

```c++
  void* GetEntryPointFromJni() const {
    DCHECK(IsNative());
    return GetEntryPointFromJniPtrSize(kRuntimePointerSize);
  }

  ALWAYS_INLINE void* GetEntryPointFromJniPtrSize(PointerSize pointer_size) const {
    return GetDataPtrSize(pointer_size);
  }

  ALWAYS_INLINE void* GetDataPtrSize(PointerSize pointer_size) const {
    DCHECK(IsImagePointerSize(pointer_size));
    return GetNativePointer<void*>(DataOffset(pointer_size), pointer_size);
  }

  // 其实就是获取了artMethod data_属性的值。
  // 具体可见下方结构体
  static constexpr MemberOffset DataOffset(PointerSize pointer_size) {
    return MemberOffset(PtrSizedFieldsOffset(pointer_size) + OFFSETOF_MEMBER(
        PtrSizedFields, data_) / sizeof(void*) * static_cast<size_t>(pointer_size));
  }




class EXPORT ArtMethod final {
  // Must be the last fields in the method.
  struct PtrSizedFields {
    // Depending on the method type, the data is
    //   - native method: pointer to the JNI function registered to this method
    //                    or a function to resolve the JNI function,
    //   - resolution method: pointer to a function to resolve the method and
    //                        the JNI function for @CriticalNative.
    //   - conflict method: ImtConflictTable,
    //   - abstract/interface method: the single-implementation if any,
    //   - proxy method: the original interface method or constructor,
    //   - default conflict method: null
    //   - other methods: during AOT the code item offset, at runtime a pointer
    //                    to the code item.
    void* data_;

    // Method dispatch from quick compiled code invokes this pointer which may cause bridging into
    // the interpreter.
    void* entry_point_from_quick_compiled_code_;
  } ptr_sized_fields_;
       
}
```



而对于JNI方法data_的值会在JNI RegisterNative中设置

```c++
// 1. JNI Call
env->RegisterNatives(XX)

// 2. JNI实现
static jint RegisterNatives(JNIEnv* env,
                              jclass java_class,
                              const JNINativeMethod* methods,
                              jint method_count) {

    // ......
    const void* final_function_ptr = class_linker->RegisterNative(soa.Self(), m, fnPtr);
    // ......
    return JNI_OK;
  }

// 3. 设置method的data_变量值 
const void* ClassLinker::RegisterNative(
    Thread* self, ArtMethod* method, const void* native_method) {
 
    
    // ......
    method->SetEntryPointFromJni(new_native_method);
    
    // ......
    
}
```



#### artQuickGenericJniEndTrampoline

该方法主要是用于在JNI方法调用后，负责清理(handle scope, saved state) & unlocking。总的来说就是收尾

```c++

/*
 * Is called after the native JNI code. Responsible for cleanup (handle scope, saved state) and
 * unlocking.
 */
extern "C" uint64_t artQuickGenericJniEndTrampoline(Thread* self,
                                                    jvalue result,
                                                    uint64_t result_f) {
  // We're here just back from a native call. We don't have the shared mutator lock at this point
  // yet until we call GoToRunnable() later in GenericJniMethodEnd(). Accessing objects or doing
  // anything that requires a mutator lock before that would cause problems as GC may have the
  // exclusive mutator lock and may be moving objects, etc.
  ArtMethod** sp = self->GetManagedStack()->GetTopQuickFrame();
  DCHECK(self->GetManagedStack()->GetTopQuickFrameGenericJniTag());
  uint32_t* sp32 = reinterpret_cast<uint32_t*>(sp);
  ArtMethod* called = *sp;
  uint32_t cookie = *(sp32 - 1);
  return GenericJniMethodEnd(self, cookie, result, result_f, called);
}
```



主要就是如几个时期

1.针对于normal_native/fast_native方法进行收尾操作（主要是suspend check）

2.针对于synchronized方法进行解锁。

3.将返回值设置到ret中并返回

```c++
extern uint64_t GenericJniMethodEnd(Thread* self,
                                    uint32_t saved_local_ref_cookie,
                                    jvalue result,
                                    uint64_t result_f,
                                    ArtMethod* called)
    NO_THREAD_SAFETY_ANALYSIS {
  bool critical_native = called->IsCriticalNative();
  bool fast_native = called->IsFastNative();
  bool normal_native = !critical_native && !fast_native;

  // @CriticalNative does not do a state transition. @FastNative usually does not do a state
  // transition either but it performs a suspend check that may do state transitions.
  // 1.貌似是要做一个suspend check操作 
  if (LIKELY(normal_native)) {
    if (UNLIKELY(self->ReadFlag(ThreadFlag::kMonitorJniEntryExit))) {
      artJniMonitoredMethodEnd(self);
    } else {
      artJniMethodEnd(self);
    }
  } else if (fast_native) {
    // When we are in @FastNative, we are already Runnable.
    DCHECK(Locks::mutator_lock_->IsSharedHeld(self));
    // Only do a suspend check on the way out of JNI just like compiled stubs.
    self->CheckSuspend();
  }
    
  // We need the mutator lock (i.e., calling `artJniMethodEnd()`) before accessing
  // the shorty or the locked object.
  // 2.针对于synchronized方法进行解锁。
  if (called->IsSynchronized()) {
    DCHECK(normal_native) << "@FastNative/@CriticalNative and synchronize is not supported";
    ObjPtr<mirror::Object> lock = GetGenericJniSynchronizationObject(self, called);
    DCHECK(lock != nullptr);
    artJniUnlockObject(lock.Ptr(), self);
  }
    
  // 3.将返回值设置到ret中并返回
  char return_shorty_char = called->GetShorty()[0];
  uint64_t ret;
  if (return_shorty_char == 'L') {
    ret = reinterpret_cast<uint64_t>(
        UNLIKELY(self->IsExceptionPending()) ? nullptr : JniDecodeReferenceResult(result.l, self));
    PopLocalReferences(saved_local_ref_cookie, self);
  } else {
    if (LIKELY(!critical_native)) {
      PopLocalReferences(saved_local_ref_cookie, self);
    }
    switch (return_shorty_char) {
      case 'F': {
        if (kRuntimeISA == InstructionSet::kX86) {
          // Convert back the result to float.
          double d = bit_cast<double, uint64_t>(result_f);
          ret = bit_cast<uint32_t, float>(static_cast<float>(d));
        } else {
          ret = result_f;
        }
      }
      break;
      case 'D':
        ret = result_f;
        break;
      case 'Z':
        ret = result.z;
        break;
      case 'B':
        ret = result.b;
        break;
      case 'C':
        ret = result.c;
        break;
      case 'S':
        ret = result.s;
        break;
      case 'I':
        ret = result.i;
        break;
      case 'J':
        ret = result.j;
        break;
      case 'V':
        ret = 0;
        break;
      default:
        LOG(FATAL) << "Unexpected return shorty character " << return_shorty_char;
        UNREACHABLE();
    }
  }

  return ret;
}
```

#### 收尾

对应于如下过程

1.check是否有异常，若有则调用异常处理函数

2.清空栈指针

3.check是否有设置exit_hook如果有则调用

4.清空栈空间，恢复寄存器，返回程序执行

```asm
    // Pending exceptions possible.
    // 判断执行过程中是否有异常信息，如果有异常信息跳转到异常处理函数中去
    ldr x2, [xSELF, THREAD_EXCEPTION_OFFSET]
    cbnz x2, .Lexception_in_native

    // Tear down the alloca.
    // 清空frame空间占用
    mov sp, x28
	// check是否需要调用exit hook方法
    LOAD_RUNTIME_INSTANCE x1
    ldrb w1, [x1, #RUN_EXIT_HOOKS_OFFSET_FROM_RUNTIME_INSTANCE]
    CFI_REMEMBER_STATE
    cbnz w1, .Lcall_method_exit_hook
.Lcall_method_exit_hook_done:

    // Tear down the callee-save frame.
    .cfi_def_cfa_register sp
    // Restore callee-saves and LR as in `RESTORE_SAVE_REFS_AND_ARGS_FRAME`
    // but do not restore argument registers.
    // Note: Likewise, we could avoid restoring X20 in the case of Baker
    // read barriers, as it is overwritten by REFRESH_MARKING_REGISTER
    // later; but it's not worth handling this special case.
#if (FRAME_SIZE_SAVE_REFS_AND_ARGS != 224)
#error "FRAME_SIZE_SAVE_REFS_AND_ARGS(ARM64) size not as expected."
#endif
	// 恢复寄存器
    RESTORE_REG x20, 136
    RESTORE_TWO_REGS x21, x22, 144
    RESTORE_TWO_REGS x23, x24, 160
    RESTORE_TWO_REGS x25, x26, 176
    RESTORE_TWO_REGS x27, x28, 192
    RESTORE_TWO_REGS x29, xLR, 208
    // Remove the frame.
    DECREASE_FRAME FRAME_SIZE_SAVE_REFS_AND_ARGS

    REFRESH_MARKING_REGISTER

    // store into fpr, for when it's a fpr return...
    fmov d0, x0
    ret

.Lcall_method_exit_hook:
    CFI_RESTORE_STATE_AND_DEF_CFA x28, FRAME_SIZE_SAVE_REFS_AND_ARGS
    fmov d0, x0
    mov x4, FRAME_SIZE_SAVE_REFS_AND_ARGS
    bl art_quick_method_exit_hook
    b .Lcall_method_exit_hook_done

.Lexception_in_native:
    // Move to x1 then sp to please assembler.
    ldr x1, [xSELF, # THREAD_TOP_QUICK_FRAME_OFFSET]
    add sp, x1, #-1  // Remove the GenericJNI tag.
    bl art_deliver_pending_exception
```



- 异常处理

这块不是文章的重点，笔者就不继续往下分析了～感兴趣的同学可以接着往下看～

```asm
// check 当前执行流程是否有pending exception
// 如果有跳转到异常处理方法中
ldr x2, [xSELF, THREAD_EXCEPTION_OFFSET]
cbnz x2, .Lexception_in_native

// ......
.Lexception_in_native:
    // Move to x1 then sp to please assembler.
    ldr x1, [xSELF, # THREAD_TOP_QUICK_FRAME_OFFSET]
    add sp, x1, #-1  // Remove the GenericJNI tag.
    bl art_deliver_pending_exception
```

- 清空堆栈指针

```asm
SETUP_SAVE_REFS_AND_ARGS_FRAME_WITH_METHOD_IN_X0
// Save SP, so we can have static CFI info.
mov x28, sp


// 清空frame空间占用
// x28寄存器保存了刚调用方法时的sp指针值
mov sp, x28
```

- exit_hook

```asm
// check是否需要调用exit_hook
LOAD_RUNTIME_INSTANCE x1
ldrb w1, [x1, #RUN_EXIT_HOOKS_OFFSET_FROM_RUNTIME_INSTANCE]
CFI_REMEMBER_STATE
cbnz w1, .Lcall_method_exit_hook

.Lcall_method_exit_hook:
    CFI_RESTORE_STATE_AND_DEF_CFA x28, FRAME_SIZE_SAVE_REFS_AND_ARGS
    fmov d0, x0
    mov x4, FRAME_SIZE_SAVE_REFS_AND_ARGS
    // 通过art_quick_method_exit_hook调用artMethodExitHook
    bl art_quick_method_exit_hook
    b .Lcall_method_exit_hook_done
```

这里的逻辑也比较简单，就是调用了artMethodExitHook这个方法

```asm
ENTRY art_quick_method_exit_hook
    SETUP_SAVE_EVERYTHING_FRAME

    // frame_size is passed from JITed code in x4
    add x3, sp, #16                           // floating-point result ptr in kSaveEverything frame
    add x2, sp, #272                          // integer result ptr in kSaveEverything frame
    add x1, sp, #FRAME_SIZE_SAVE_EVERYTHING   // ArtMethod**
    mov x0, xSELF                             // Thread::Current
    bl  artMethodExitHook                     // (Thread*, ArtMethod**, gpr_res*, fpr_res*,
                                              // frame_size)

    // Normal return.
    RESTORE_SAVE_EVERYTHING_FRAME
    REFRESH_MARKING_REGISTER
    ret
END art_quick_method_exit_hook
```





## art_jni_dlsym_lookup_stub



### 入口设置



若JNI方法没有注册Native实现类，类加载的时候将设置如下跳板。

不太直观对吧？不太能理解这个跳板是用来干嘛的？简单介绍下吧～

JNI方法有两种声明方式，这里的跳板函数的主要作用就是用于解析第二种JNI方法声明的Native函数地址。

- 一种是通过JNI_Onload中调用RegisterNative

- 另外一种是将JNI的实现方法名称声明为Java_pkg_functionName这样的格式。

```c++
static void LinkCode(ClassLinker* class_linker,
                     ArtMethod* method,
                     const OatFile::OatClass* oat_class,
                     uint32_t class_def_method_index) REQUIRES_SHARED(Locks::mutator_lock_) {
                     

	// ......
	
	if (method->IsNative()) {
        // Set up the dlsym lookup stub. Do not go through `UnregisterNative()`
        // as the extra processing for @CriticalNative is not needed yet.
        // 依据Native方法是否被打上标记@dalvik.annotation.optimization.CriticalNative.
        // 如果标记了@CriticalNative  -> art_jni_dlsym_lookup_critical_stub
        // 如果没有标记 				-> art_jni_dlsym_lookup_stub
        method->SetEntryPointFromJni(
            method->IsCriticalNative() ? GetJniDlsymLookupCriticalStub() : GetJniDlsymLookupStub());
  	}
	
	// ......


}
```



跳板代码如下。只是overview看一下，后面会对流程做细分，拆分解析。总的来说，这个跳板的逻辑可以分为如下几个过程

1.开辟栈针保存寄存器

2.获取native方法地址

3.收尾/结束

```asm
ENTRY art_jni_dlsym_lookup_stub
	// 1. 开辟栈针保存寄存器，这里会保存x0~x7, d0~d7, x29,x30
	// 共20个寄存器的值
    // spill regs.
    SAVE_ALL_ARGS_INCREASE_FRAME 2 * 8
    stp   x29, x30, [sp, ALL_ARGS_SIZE]
    .cfi_rel_offset x29, ALL_ARGS_SIZE
    .cfi_rel_offset x30, ALL_ARGS_SIZE + 8
    add   x29, sp, ALL_ARGS_SIZE

	// 2.调用artFindNativeMethod/artFindNativeMethodRunnable方法获取JNI的实现方法地址
    mov x0, xSELF   // pass Thread::Current()
    // Call artFindNativeMethod() for normal native and artFindNativeMethodRunnable()
    // for @FastNative or @CriticalNative.
    ldr   xIP0, [x0, #THREAD_TOP_QUICK_FRAME_OFFSET]      // uintptr_t tagged_quick_frame
    bic   xIP0, xIP0, #TAGGED_JNI_SP_MASK                 // ArtMethod** sp
    ldr   xIP0, [xIP0]                                    // ArtMethod* method
    ldr   xIP0, [xIP0, #ART_METHOD_ACCESS_FLAGS_OFFSET]   // uint32_t access_flags
    mov   xIP1, #(ACCESS_FLAGS_METHOD_IS_FAST_NATIVE | ACCESS_FLAGS_METHOD_IS_CRITICAL_NATIVE)
    tst   xIP0, xIP1
    b.ne  .Llookup_stub_fast_or_critical_native
    bl    artFindNativeMethod
    b     .Llookup_stub_continue
    .Llookup_stub_fast_or_critical_native:
    bl    artFindNativeMethodRunnable
    
    // 3.收尾/结束
    // 恢复堆栈，调用#2获取的native 函数地址，执行native方法。
.Llookup_stub_continue:
    mov   x17, x0    // store result in scratch reg.

    // load spill regs.
    ldp   x29, x30, [sp, #ALL_ARGS_SIZE]
    .cfi_restore x29
    .cfi_restore x30
    RESTORE_ALL_ARGS_DECREASE_FRAME 2 * 8

    cbz   x17, 1f   // is method code null ?
    br    x17       // if non-null, tail call to method's code.

1:
    ret             // restore regs and return to caller to handle exception.
END art_jni_dlsym_lookup_stub

```





### 跳板分析



1.开辟栈针保存寄存器

2.获取native方法地址

3.收尾/结束



- 堆栈开辟

其实很简单就保存了x0~x7, d0~d7

```asm
 // spill regs.
 // 开辟堆栈空间，通过宏定义保存x0~x7, d0~d7
SAVE_ALL_ARGS_INCREASE_FRAME 2 * 8
 // 保存x29, x30寄存器 
stp   x29, x30, [sp, ALL_ARGS_SIZE]
.cfi_rel_offset x29, ALL_ARGS_SIZE
.cfi_rel_offset x30, ALL_ARGS_SIZE + 8
 // 设置基地址 
add   x29, sp, ALL_ARGS_SIZE


// 宏定义：
// 用于开辟堆栈空间，大小为128 + extra_space bytes 其中会保存 x0~x7, d0 ~ d7寄存器值
.macro SAVE_ALL_ARGS_INCREASE_FRAME extra_space
    // Save register args x0-x7, d0-d7 and return address.
    stp    x0, x1, [sp, #-(ALL_ARGS_SIZE + \extra_space)]!
    .cfi_adjust_cfa_offset (ALL_ARGS_SIZE + \extra_space)
    stp    x2, x3, [sp, #16]
    stp    x4, x5, [sp, #32]
    stp    x6, x7, [sp, #48]
    stp    d0, d1, [sp, #64]
    stp    d2, d3, [sp, #80]
    stp    d4, d5, [sp, #96]
    stp    d6, d7, [sp, #112]
.endm
```

- 调用方法获取jni native实现代码地址

对于普通的native方法调用artFindNativeMethod获取JNI地址，对于 @FastNative or @CriticalNative标记的方法调用artFindNativeMethodRunnable获取JNI地址。

```asm
mov x0, xSELF   // pass Thread::Current()
// Call artFindNativeMethod() for normal native and artFindNativeMethodRunnable()
// for @FastNative or @CriticalNative.
// 获取tagged_quick_frame指针地址
ldr   xIP0, [x0, #THREAD_TOP_QUICK_FRAME_OFFSET]      // uintptr_t tagged_quick_frame
// 将mask对应的位数清空
bic   xIP0, xIP0, #TAGGED_JNI_SP_MASK                 // ArtMethod** sp
// 获取artMethod对象
ldr   xIP0, [xIP0]                                    // ArtMethod* method
// 获取access_flags字段值
ldr   xIP0, [xIP0, #ART_METHOD_ACCESS_FLAGS_OFFSET]   // uint32_t access_flags
// check access_flags是否是CriticalNative or FastNative
mov   xIP1, #(ACCESS_FLAGS_METHOD_IS_FAST_NATIVE | ACCESS_FLAGS_METHOD_IS_CRITICAL_NATIVE)
tst   xIP0, xIP1
// 如果是CriticalNative or FastNative
b.ne  .Llookup_stub_fast_or_critical_native
// 如果不是CriticalNative or FastNative
bl    artFindNativeMethod
// 执行完毕跳转到后一步继续执行，进入收尾阶段
b     .Llookup_stub_continue
.Llookup_stub_fast_or_critical_native:
bl    artFindNativeMethodRunnable
```



```cpp
// Used by the JNI dlsym stub to find the native method to invoke if none is registered.
extern "C" const void* artFindNativeMethod(Thread* self) {
  DCHECK_EQ(self, Thread::Current());
  Locks::mutator_lock_->AssertNotHeld(self);  // We come here as Native.
  ScopedObjectAccess soa(self);
  return artFindNativeMethodRunnable(self);
}
```

核心逻辑可分为如下几个步骤

1.二次check当前的class是否已经绑定了native code，如果有直接返回，如果没有继续往下处理

2.根据symbol寻找native code地址

3.注册native code以便后续操作

```cpp
// Used by the JNI dlsym stub to find the native method to invoke if none is registered.
extern "C" const void* artFindNativeMethodRunnable(Thread* self)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  Locks::mutator_lock_->AssertSharedHeld(self);  // We come here as Runnable.
  uint32_t dex_pc;
  ArtMethod* method = self->GetCurrentMethod(&dex_pc);
  DCHECK(method != nullptr);
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();

  // 特殊case的兼容。从compiled managed code调用过来的，这里的method对象是caller不是callee需要单独做一下解析。
  if (!method->IsNative()) {
    // We're coming from compiled managed code and the `method` we see here is the caller.
    // Resolve target @CriticalNative method for a direct call from compiled managed code.
    uint32_t method_idx = GetInvokeStaticMethodIndex(method, dex_pc);
    ArtMethod* target_method = class_linker->ResolveMethod<ClassLinker::ResolveMode::kNoChecks>(
        self, method_idx, method, kStatic);
    if (target_method == nullptr) {
      self->AssertPendingException();
      return nullptr;
    }
    DCHECK(target_method->IsCriticalNative());
    // Note that the BSS also contains entries used for super calls. Given we
    // only deal with invokestatic in this code path, we don't need to adjust
    // the method index.
    MaybeUpdateBssMethodEntry(target_method,
                              MethodReference(method->GetDexFile(), method_idx),
                              GetCalleeSaveOuterMethod(self, CalleeSaveType::kSaveRefsAndArgs));

    // These calls do not have an explicit class initialization check, so do the check now.
    // (When going through the stub or GenericJNI, the check was already done.)
    DCHECK(target_method->NeedsClinitCheckBeforeCall());
    ObjPtr<mirror::Class> declaring_class = target_method->GetDeclaringClass();
    if (UNLIKELY(!declaring_class->IsVisiblyInitialized())) {
      StackHandleScope<1> hs(self);
      Handle<mirror::Class> h_class(hs.NewHandle(declaring_class));
      if (!class_linker->EnsureInitialized(self, h_class, true, true)) {
        DCHECK(self->IsExceptionPending()) << method->PrettyMethod();
        return nullptr;
      }
    }

    // Replace the runtime method on the stack with the target method.
    DCHECK(!self->GetManagedStack()->GetTopQuickFrameGenericJniTag());
    ArtMethod** sp = self->GetManagedStack()->GetTopQuickFrameKnownNotTagged();
    DCHECK(*sp == Runtime::Current()->GetCalleeSaveMethod(CalleeSaveType::kSaveRefsAndArgs));
    *sp = target_method;
    self->SetTopOfStackGenericJniTagged(sp);  // Fake GenericJNI frame.

    // Continue with the target method.
    method = target_method;
  }
  DCHECK(method == self->GetCurrentMethod(/*dex_pc=*/ nullptr));

  // Check whether we already have a registered native code.
  // For @CriticalNative it may not be stored in the ArtMethod as a JNI entrypoint if the class
  // was not visibly initialized yet. Do this check also for @FastNative and normal native for
  // consistency; though success would mean that another thread raced to do this lookup.
  // 1. check一下是否已经包含注册了native code
  const void* native_code = class_linker->GetRegisteredNative(self, method);
  if (native_code != nullptr) {
    return native_code;
  }

  // Lookup symbol address for method, on failure we'll return null with an exception set,
  // otherwise we return the address of the method we found.
  // 2. 寻找symbol地址，当失败的时候将返回null值，其他情况都会返回我们找到的native代码的地址  
  JavaVMExt* vm = down_cast<JNIEnvExt*>(self->GetJniEnv())->GetVm();
  std::string error_msg;
  native_code = vm->FindCodeForNativeMethod(method, &error_msg, /*can_suspend=*/ true);
  if (native_code == nullptr) {
    LOG(ERROR) << error_msg;
    self->ThrowNewException("Ljava/lang/UnsatisfiedLinkError;", error_msg.c_str());
    return nullptr;
  }

  // Register the code. This usually prevents future calls from coming to this function again.
  // We can still come here if the ClassLinker cannot set the entrypoint in the ArtMethod,
  // i.e. for @CriticalNative methods with the declaring class not visibly initialized.
  // 3. 注册一下。以便后续调用
  return class_linker->RegisterNative(self, method, native_code);
}
```

这里在对native code的寻找逻辑再做一下介绍, 此处寻找的native method code的名称是比较特殊的形如

`JNIEXPORT <返回类型> JNICALL Java_<包名>_<类名>_<方法名>(JNIEnv*, <原对象引用>，<参数1>..<参数n>)`

如下便是native code地址的寻找逻辑

```c++
void* JavaVMExt::FindCodeForNativeMethod(ArtMethod* m, std::string* error_msg, bool can_suspend) {
  // ......
  ObjPtr<mirror::Class> c = m->GetDeclaringClass();
  Thread* const self = Thread::Current();
  void* native_method = libraries_->FindNativeMethod(self, m, error_msg, can_suspend);
  // ......
  return native_method;
}

// See section 11.3 "Linking Native Methods" of the JNI spec.
// 寻找jni方法地址，
  void* FindNativeMethod(Thread* self, ArtMethod* m, std::string* detail, bool can_suspend)
      REQUIRES(!Locks::jni_libraries_lock_)
      REQUIRES_SHARED(Locks::mutator_lock_) {
      // 获取JNI的name
    std::string jni_short_name(m->JniShortName());
    std::string jni_long_name(m->JniLongName());
    const ObjPtr<mirror::ClassLoader> declaring_class_loader =
        m->GetDeclaringClass()->GetClassLoader();
    void* const declaring_class_loader_allocator =
        Runtime::Current()->GetClassLinker()->GetAllocatorForClassLoader(declaring_class_loader);
    CHECK(declaring_class_loader_allocator != nullptr);
    // TODO: Avoid calling GetShorty here to prevent dirtying dex pages?
    const char* shorty = m->GetShorty();
    void* native_code = nullptr;
    android::JNICallType jni_call_type =
        m->IsCriticalNative() ? android::kJNICallTypeCriticalNative : android::kJNICallTypeRegular;
    
    // .......
    // 通过jniName从so中寻找特定的符号  
    native_code =  FindNativeMethodInternal(self,
                                             declaring_class_loader_allocator,
                                             shorty,
                                             jni_short_name,
                                             jni_long_name,
                                             jni_call_type)
    if (native_code != nullptr) {
      return native_code;
    }
	// ......
    return nullptr;
  }

```

在上述的方法中，最终从so寻找的native函数的符号名称的拼接逻辑如下

```cpp
std::string ArtMethod::JniShortName() {
  return GetJniShortName(GetDeclaringClassDescriptor(), GetName());
}

std::string GetJniShortName(const std::string& class_descriptor, const std::string& method) {
  // Remove the leading 'L' and trailing ';'...
  std::string class_name(class_descriptor);
  CHECK_EQ(class_name[0], 'L') << class_name;
  CHECK_EQ(class_name[class_name.size() - 1], ';') << class_name;
  class_name.erase(0, 1);
  class_name.erase(class_name.size() - 1, 1);
  // 初始值为空字符串
  std::string short_name;
  // 添加Java_前缀
  short_name += "Java_";
  // 获取类名并将其'/'设置为'_'
  short_name += MangleForJni(class_name);
  // 添加间隔符
  short_name += "_";
  // 添加方法名称
  short_name += MangleForJni(method);
  // 最终方法名称为Java_pkg_functionName
  return short_name;
}
```



- 收尾/结束

```c++
.Llookup_stub_continue:
	// 这里的x0是上一步获取的实际的native代码地址
    mov   x17, x0    // store result in scratch reg.

    // load spill regs.
    // 恢复寄存器
    ldp   x29, x30, [sp, #ALL_ARGS_SIZE]
    .cfi_restore x29
    .cfi_restore x30
    RESTORE_ALL_ARGS_DECREASE_FRAME 2 * 8
	// 尝试调用函数
    // 如果native code地址不为空，通过 br跳转到函数的执行中去
    // 如果为空，直接返回。
    cbz   x17, 1f   // is method code null ?
    br    x17       // if non-null, tail call to method's code.

1:
    ret             // restore regs and return to caller to handle exception
```







## art_jni_dlsym_lookup_critical_stub

- 入口设置

见art_jni_dlsym_lookup_stub

- 跳板

1.参数上下文保存

2.堆帧管理

3.方法查找与调用

4.异常处理与栈恢复

5.尾调用优化

```c++
    /*
     * Jni dlsym lookup stub for @CriticalNative.
     */
ENTRY art_jni_dlsym_lookup_critical_stub
    // 1.参数上下文保存
    // The hidden arg holding the tagged method (bit 0 set means GenericJNI) is x15.
    // For Generic JNI we already have a managed frame, so we reuse the art_jni_dlsym_lookup_stub.
    tbnz  x15, #0, art_jni_dlsym_lookup_stub

    // Save args, the hidden arg and caller PC. No CFI needed for args and the hidden arg.
    SAVE_ALL_ARGS_INCREASE_FRAME 2 * 8
    stp   x15, lr, [sp, #ALL_ARGS_SIZE]
    .cfi_rel_offset lr, ALL_ARGS_SIZE + 8

    // Call artCriticalNativeFrameSize(method, caller_pc)
    mov   x0, x15  // x0 := method (from hidden arg)
    mov   x1, lr   // x1 := caller_pc
    bl    artCriticalNativeFrameSize

    // Move frame size to x14.
    mov   x14, x0

    // Restore args, the hidden arg and caller PC.
    ldp   x15, lr, [sp, #128]
    .cfi_restore lr
    RESTORE_ALL_ARGS_DECREASE_FRAME 2 * 8

    // Reserve space for a SaveRefsAndArgs managed frame, either for the actual runtime
    // method or for a GenericJNI frame which is similar but has a native method and a tag.
    INCREASE_FRAME FRAME_SIZE_SAVE_REFS_AND_ARGS

    // Calculate the base address of the managed frame.
    // 2.栈帧管理
    add   x13, sp, x14

    // Prepare the return address for managed stack walk of the SaveRefsAndArgs frame.
    // If we're coming from JNI stub with tail call, it is LR. If we're coming from
    // JNI stub that saved the return address, it will be the last value we copy below.
    // If we're coming directly from compiled code, it is LR, set further down.
    mov   xIP1, lr

    // Move the stack args if any.
    cbz   x14, .Lcritical_skip_copy_args
    mov   x12, sp
.Lcritical_copy_args_loop:
    ldp   xIP0, xIP1, [x12, #FRAME_SIZE_SAVE_REFS_AND_ARGS]
    subs  x14, x14, #16
    stp   xIP0, xIP1, [x12], #16
    bne   .Lcritical_copy_args_loop
.Lcritical_skip_copy_args:

    // Spill registers for the SaveRefsAndArgs frame above the stack args.
    // Note that the runtime shall not examine the args here, otherwise we would have to
    // move them in registers and stack to account for the difference between managed and
    // native ABIs. Do not update CFI while we hold the frame address in x13 and the values
    // in registers are unchanged.
    stp   d0, d1, [x13, #16]
    stp   d2, d3, [x13, #32]
    stp   d4, d5, [x13, #48]
    stp   d6, d7, [x13, #64]
    stp   x1, x2, [x13, #80]
    stp   x3, x4, [x13, #96]
    stp   x5, x6, [x13, #112]
    stp   x7, x20, [x13, #128]
    stp   x21, x22, [x13, #144]
    stp   x23, x24, [x13, #160]
    stp   x25, x26, [x13, #176]
    stp   x27, x28, [x13, #192]
    stp   x29, xIP1, [x13, #208]  // xIP1: Save return address for tail call from JNI stub.
    // (If there were any stack args, we're storing the value that's already there.
    // For direct calls from compiled managed code, we shall overwrite this below.)

    // Move the managed frame address to native callee-save register x29 and update CFI.
    mov   x29, x13
    // Skip args d0-d7, x1-x7
    CFI_EXPRESSION_BREG 20, 29, 136
    CFI_EXPRESSION_BREG 21, 29, 144
    CFI_EXPRESSION_BREG 22, 29, 152
    CFI_EXPRESSION_BREG 23, 29, 160
    CFI_EXPRESSION_BREG 24, 29, 168
    CFI_EXPRESSION_BREG 25, 29, 176
    CFI_EXPRESSION_BREG 26, 29, 184
    CFI_EXPRESSION_BREG 27, 29, 192
    CFI_EXPRESSION_BREG 28, 29, 200
    CFI_EXPRESSION_BREG 29, 29, 208
    // The saved return PC for managed stack walk is not necessarily our LR.

    // Save our return PC in the padding.
    str   lr, [x29, #__SIZEOF_POINTER__]
    CFI_EXPRESSION_BREG 30, 29, __SIZEOF_POINTER__

    ldr   wIP0, [x15, #ART_METHOD_ACCESS_FLAGS_OFFSET]  // Load access flags.
    add   x14, x29, #1            // Prepare managed SP tagged for a GenericJNI frame.
    tbnz  wIP0, #ACCESS_FLAGS_METHOD_IS_NATIVE_BIT, .Lcritical_skip_prepare_runtime_method

    // When coming from a compiled method, the return PC for managed stack walk is LR.
    // (When coming from a compiled stub, the correct return PC is already stored above.)
    str   lr, [x29, #(FRAME_SIZE_SAVE_REFS_AND_ARGS - __SIZEOF_POINTER__)]

    // Replace the target method with the SaveRefsAndArgs runtime method.
    LOAD_RUNTIME_INSTANCE x15
    ldr   x15, [x15, #RUNTIME_SAVE_REFS_AND_ARGS_METHOD_OFFSET]

    mov   x14, x29                // Prepare untagged managed SP for the runtime method.
// 3.方法查找与调用
.Lcritical_skip_prepare_runtime_method:
    // Store the method on the bottom of the managed frame.
    str   x15, [x29]

    // Place (maybe tagged) managed SP in Thread::Current()->top_quick_frame.
    str   x14, [xSELF, #THREAD_TOP_QUICK_FRAME_OFFSET]

    // Preserve the native arg register x0 in callee-save register x28 which was saved above.
    mov   x28, x0

    // Call artFindNativeMethodRunnable()
    // 方法查找
    mov   x0, xSELF   // pass Thread::Current()
    bl    artFindNativeMethodRunnable

    // Store result in scratch reg.
    mov   x13, x0

    // Restore the native arg register x0.
    mov   x0, x28

    // Restore our return PC.
    RESTORE_REG_BASE x29, lr, __SIZEOF_POINTER__

    // Remember the stack args size, negated because SP cannot be on the right-hand side in SUB.
    sub   x14, sp, x29

    // Restore the frame. We shall not need the method anymore.
    ldp   d0, d1, [x29, #16]
    ldp   d2, d3, [x29, #32]
    ldp   d4, d5, [x29, #48]
    ldp   d6, d7, [x29, #64]
    ldp   x1, x2, [x29, #80]
    ldp   x3, x4, [x29, #96]
    ldp   x5, x6, [x29, #112]
    ldp   x7, x20, [x29, #128]
    .cfi_restore x20
    RESTORE_TWO_REGS_BASE x29, x21, x22, 144
    RESTORE_TWO_REGS_BASE x29, x23, x24, 160
    RESTORE_TWO_REGS_BASE x29, x25, x26, 176
    RESTORE_TWO_REGS_BASE x29, x27, x28, 192
    RESTORE_REG_BASE x29, x29, 208

    REFRESH_MARKING_REGISTER

    // Check for exception before moving args back to keep the return PC for managed stack walk.
    CFI_REMEMBER_STATE
    cbz   x13, .Lcritical_deliver_exception

    // Move stack args to their original place.
    cbz   x14, .Lcritical_skip_copy_args_back
    sub   x12, sp, x14
.Lcritical_copy_args_back_loop:
    ldp   xIP0, xIP1, [x12, #-16]!
    adds  x14, x14, #16
    stp   xIP0, xIP1, [x12, #FRAME_SIZE_SAVE_REFS_AND_ARGS]
    bne   .Lcritical_copy_args_back_loop
.Lcritical_skip_copy_args_back:

    // Remove the frame reservation.
    DECREASE_FRAME FRAME_SIZE_SAVE_REFS_AND_ARGS

    // Do the tail call.
    // 4.tail call调用
    br    x13
// 5.异常处理
.Lcritical_deliver_exception:
    CFI_RESTORE_STATE_AND_DEF_CFA sp, FRAME_SIZE_SAVE_REFS_AND_ARGS
    // The exception delivery checks that xSELF was saved but the SaveRefsAndArgs
    // frame does not save it, so we cannot use the existing SaveRefsAndArgs frame.
    // That's why we checked for exception after restoring registers from it.
    // We need to build a SaveAllCalleeSaves frame instead. Args are irrelevant at this
    // point but keep the area allocated for stack args to keep CFA definition simple.
    DECREASE_FRAME FRAME_SIZE_SAVE_REFS_AND_ARGS - FRAME_SIZE_SAVE_ALL_CALLEE_SAVES

    // Calculate the base address of the managed frame.
    sub   x13, sp, x14

    // Spill registers for the SaveAllCalleeSaves frame above the stack args area. Do not update
    // CFI while we hold the frame address in x13 and the values in registers are unchanged.
    stp   d8, d9, [x13, #16]
    stp   d10, d11, [x13, #32]
    stp   d12, d13, [x13, #48]
    stp   d14, d15, [x13, #64]
    stp   x19, x20, [x13, #80]
    stp   x21, x22, [x13, #96]
    stp   x23, x24, [x13, #112]
    stp   x25, x26, [x13, #128]
    stp   x27, x28, [x13, #144]
    str   x29, [x13, #160]
    // Keep the caller PC for managed stack walk.

    // Move the managed frame address to native callee-save register x29 and update CFI.
    mov   x29, x13
    CFI_EXPRESSION_BREG 19, 29, 80
    CFI_EXPRESSION_BREG 20, 29, 88
    CFI_EXPRESSION_BREG 21, 29, 96
    CFI_EXPRESSION_BREG 22, 29, 104
    CFI_EXPRESSION_BREG 23, 29, 112
    CFI_EXPRESSION_BREG 24, 29, 120
    CFI_EXPRESSION_BREG 25, 29, 128
    CFI_EXPRESSION_BREG 26, 29, 136
    CFI_EXPRESSION_BREG 27, 29, 144
    CFI_EXPRESSION_BREG 28, 29, 152
    CFI_EXPRESSION_BREG 29, 29, 160
    // The saved return PC for managed stack walk is not necessarily our LR.

    // Save our return PC in the padding.
    str   lr, [x29, #__SIZEOF_POINTER__]
    CFI_EXPRESSION_BREG 30, 29, __SIZEOF_POINTER__

    // Store ArtMethod* Runtime::callee_save_methods_[kSaveAllCalleeSaves] to the managed frame.
    LOAD_RUNTIME_INSTANCE xIP0
    ldr   xIP0, [xIP0, #RUNTIME_SAVE_ALL_CALLEE_SAVES_METHOD_OFFSET]
    str   xIP0, [x29]

    // Place the managed frame SP in Thread::Current()->top_quick_frame.
    str   x29, [xSELF, #THREAD_TOP_QUICK_FRAME_OFFSET]

    DELIVER_PENDING_EXCEPTION_FRAME_READY
END art_jni_dlsym_lookup_critical_stub
```



### 参数与上下文处理



```c++
// The hidden arg holding the tagged method (bit 0 set means GenericJNI) is x15.
    // For Generic JNI we already have a managed frame, so we reuse the art_jni_dlsym_lookup_stub.
    tbnz  x15, #0, art_jni_dlsym_lookup_stub

    // Save args, the hidden arg and caller PC. No CFI needed for args and the hidden arg.
    SAVE_ALL_ARGS_INCREASE_FRAME 2 * 8
    stp   x15, lr, [sp, #ALL_ARGS_SIZE]
    .cfi_rel_offset lr, ALL_ARGS_SIZE + 8

    // Call artCriticalNativeFrameSize(method, caller_pc)
    mov   x0, x15  // x0 := method (from hidden arg)
    mov   x1, lr   // x1 := caller_pc
    bl    artCriticalNativeFrameSize

    // Move frame size to x14.
    mov   x14, x0

    // Restore args, the hidden arg and caller PC.
    ldp   x15, lr, [sp, #128]
    .cfi_restore lr
    RESTORE_ALL_ARGS_DECREASE_FRAME 2 * 8

    // Reserve space for a SaveRefsAndArgs managed frame, either for the actual runtime
    // method or for a GenericJNI frame which is similar but has a native method and a tag.
    INCREASE_FRAME FRAME_SIZE_SAVE_REFS_AND_ARGS

    // Calculate the base address of the managed frame.
    add   x13, sp, x14

    // Prepare the return address for managed stack walk of the SaveRefsAndArgs frame.
    // If we're coming from JNI stub with tail call, it is LR. If we're coming from
    // JNI stub that saved the return address, it will be the last value we copy below.
    // If we're coming directly from compiled code, it is LR, set further down.
    mov   xIP1, lr

    // Move the stack args if any.
    cbz   x14, .Lcritical_skip_copy_args
    mov   x12, sp
```





### 堆帧管理





### 方法查找与调用





### 异常处理与栈恢复





### 尾调用优化





## art_quick_imt_conflict_trampoline



- 基础概念

1. IMT - Interface Method Table 

`IMT conflicts`（接口方法表冲突）指的是这样一种场景：当一个类实现了多个接口，且这些接口中存在**方法签名完全相同**（即方法名、参数列表、返回值都相同）的方法时，ART 在解析这些接口方法时会出现 “冲突”—— 虚拟机 需要确定究竟该调用哪个接口的实现方法。

2. 为什么会出现 IMT 冲突？

在 Java 中，如果一个类实现的多个接口包含签名相同的方法，Java 语法要求这个类**必须显式重写该方法**以解决冲突（否则会编译报错）。但在 ART 内部实现中，虚拟机仍需要通过 IMT 机制处理这种多接口方法的映射关系，当映射出现歧义时就会触发 “IMT 冲突”。



- 设置位置



- art跳板



## art_invoke_obsolete_method_stub



- 基础概念

看着好像是为调试做准备的？用来脱糖的，推测可能jit可能会对method的做优化，导致DexCache & DexFile不一致？（我猜的嘿嘿）

``` c++
// Set to indicate that the ArtMethod is obsolete and has a different DexCache + DexFile from its
// declaring class. This flag may only be applied to methods.
static constexpr uint32_t kAccObsoleteMethod =        0x00040000;  // method (runtime)
```

- 调用路径

-> VMDebug_startMethodTracingFilename

 -> Trace::Start

  -> JitCodeCache::InvalidateAllCompiledCode

   -> ClassLinker::SetEntryPointsForObsoleteMethod

​    -> GetInvokeObsoleteMethodStub

- art跳板

```asm
ONE_ARG_RUNTIME_EXCEPTION art_invoke_obsolete_method_stub, artInvokeObsoleteMethod

    /*
     * Compiled code has requested that we deoptimize into the interpreter. The deoptimization
     * will long jump to the upcall with a special exception of -1.
     */
    .extern artDeoptimizeFromCompiledCode
ENTRY art_quick_deoptimize_from_compiled_code
    SETUP_SAVE_EVERYTHING_FRAME
    mov    x1, xSELF                      // Pass thread.
    bl     artDeoptimizeFromCompiledCode  // (DeoptimizationKind, Thread*)
    brk 0
END art_quick_deoptimize_from_compiled_code
```



## art_quick_proxy_invoke_handler

- 概念

1.proxy handler是干嘛的

主要是for代理方法的

-  调用

Proxy_generateProxy

 -> ClassLinker::CreateProxyClass

  -> ClassLinker::MarkClassInitialized

   -> ClassLinker::FixupStaticTrampolines

​    -> Instrumentation::GetCodeForInvoke

​     -> GetOptimizedCodeFor

​      -> GetQuickProxyInvokeHandler

- art跳板

```asm
ENTRY art_quick_proxy_invoke_handler
    SETUP_SAVE_REFS_AND_ARGS_FRAME_WITH_METHOD_IN_X0
    mov     x2, xSELF                   // pass Thread::Current
    mov     x3, sp                      // pass SP
    bl      artQuickProxyInvokeHandler  // (Method* proxy method, receiver, Thread*, SP)
    ldr     x2, [xSELF, THREAD_EXCEPTION_OFFSET]
    CFI_REMEMBER_STATE
    cbnz    x2, .Lexception_in_proxy    // success if no exception is pending
    RESTORE_SAVE_REFS_AND_ARGS_FRAME    // Restore frame
    REFRESH_MARKING_REGISTER
    fmov    d0, x0                      // Store result in d0 in case it was float or double
    ret                                 // return on success
.Lexception_in_proxy:
    CFI_RESTORE_STATE_AND_DEF_CFA sp, FRAME_SIZE_SAVE_REFS_AND_ARGS
    RESTORE_SAVE_REFS_AND_ARGS_FRAME
    DELIVER_PENDING_EXCEPTION
END art_quick_proxy_invoke_handler
```



## art_quick_deoptimize



// 暂时不知道是干嘛的，貌似好像也没有人调用

- art跳板

```asm
ENTRY art_quick_deoptimize_from_compiled_code
    SETUP_SAVE_EVERYTHING_FRAME
    mov    x1, xSELF                      // Pass thread.
    bl     artDeoptimizeFromCompiledCode  // (DeoptimizationKind, Thread*)
    brk 0
END art_quick_deoptimize_from_compiled_code
```







## null



```cpp
ArtMethod* Runtime::CreateCalleeSaveMethod() {
  auto* method = CreateRuntimeMethod(GetClassLinker(), GetLinearAlloc());
  PointerSize pointer_size = GetInstructionSetPointerSize(instruction_set_);
  method->SetEntryPointFromQuickCompiledCodePtrSize(nullptr, pointer_size);
  DCHECK_NE(instruction_set_, InstructionSet::kNone);
  DCHECK(method->IsRuntimeMethod());
  return method;
}
```



## quickCode





## 其他记录

- art跳板 列表

art/runtime/entrypoints/runtime_asm_entrypoints.h

- quick entrypoints 列表

art/runtime/entrypoints/quick/quick_entrypoints.h

- art设置entrypoint的方法

```shell
-> cgrep "SetEntryPointFromQuickCompiledCodePtrSize"
./art/dex2oat/linker/image_writer.cc:3530:    copy->SetEntryPointFromQuickCompiledCodePtrSize(quick_code, target_ptr_size_);
./art/dex2oat/linker/oat_writer.cc:1461:            method.SetEntryPointFromQuickCompiledCodePtrSize(
./art/dex2oat/linker/oat_writer.cc:1493:      resolved_method->SetEntryPointFromQuickCompiledCodePtrSize(
./art/dex2oat/linker/oat_writer.cc:1514:        method->SetEntryPointFromQuickCompiledCodePtrSize(code_ptr, pointer_size_);
./art/runtime/gc/space/image_space.cc:1409:            method.SetEntryPointFromQuickCompiledCodePtrSize(new_code, kPointerSize);
./art/runtime/art_method.cc:814:      SetEntryPointFromQuickCompiledCodePtrSize(
./art/runtime/art_method.cc:823:    SetEntryPointFromQuickCompiledCodePtrSize(GetQuickToInterpreterBridge(), image_pointer_size);
./art/runtime/class_linker.cc:2234:          method.SetEntryPointFromQuickCompiledCodePtrSize(GetQuickToInterpreterBridge(),
./art/runtime/class_linker.cc:3693:  method->SetEntryPointFromQuickCompiledCodePtrSize(
./art/runtime/art_method-inl.h:679:    SetEntryPointFromQuickCompiledCodePtrSize(new_code, pointer_size);
./art/runtime/art_method.h:764:    SetEntryPointFromQuickCompiledCodePtrSize(entry_point_from_quick_compiled_code,
./art/runtime/art_method.h:767:  ALWAYS_INLINE void SetEntryPointFromQuickCompiledCodePtrSize(
./art/runtime/runtime.cc:2730:    method->SetEntryPointFromQuickCompiledCodePtrSize(nullptr, pointer_size);
./art/runtime/runtime.cc:2751:    method->SetEntryPointFromQuickCompiledCodePtrSize(nullptr, pointer_size);
./art/runtime/runtime.cc:2763:  method->SetEntryPointFromQuickCompiledCodePtrSize(nullptr, pointer_size);
```

- 跳板判断

```c++
// art/runtime/instrumentation.cc
std::string Instrumentation::EntryPointString(const void* code) {
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
  jit::Jit* jit = Runtime::Current()->GetJit();
  if (class_linker->IsQuickToInterpreterBridge(code)) {
    return "interpreter";
  } else if (class_linker->IsQuickResolutionStub(code)) {
    return "resolution";
  } else if (jit != nullptr && jit->GetCodeCache()->ContainsPc(code)) {
    return "jit";
  } else if (code == GetInvokeObsoleteMethodStub()) {
    return "obsolete";
  } else if (code == interpreter::GetNterpEntryPoint()) {
    return "nterp";
  } else if (code == interpreter::GetNterpWithClinitEntryPoint()) {
    return "nterp with clinit";
  } else if (class_linker->IsQuickGenericJniStub(code)) {
    return "generic jni";
  } else if (Runtime::Current()->GetOatFileManager().ContainsPc(code)) {
    return "oat";
  } else if (OatQuickMethodHeader::IsStub(reinterpret_cast<const uint8_t*>(code)).value_or(false)) {
    return "stub";
  }
  return "unknown";
}
```





# QA





## Art 跳板什么时候会被写入



类加载链接阶段即LinkCode函数中写入





# 灵感（最终会删除）



## 关于函数传参



如果一个Java方法A调用了另外一个Java方法B，参数是如何由A传到B中去的呢？

invoke指令中会包含参数的寄存器(art寄存器)列表，这些寄存器内会保存art的参数，最终这些参数会通过ShadowFrame的方式传入到art内。并通过映射最终保存到另外。



## 启动解释器以后



在启动解释器以后不会再调用ArtMethod.Invoke方法执行下一个方法

而是通过使用：

interpreter::ArtInterpreterToInterpreterBridge -> 解释器到解释器

interpreter::ArtInterpreterToCompiledCodeBridge -> 解释器到jit/aot/jni



调用Invoke和调用ArtInterpreterToInterpreterBridge的区别在与是否需要调用一层跳板。

如果是Invoke那么就需要调用一层跳板，如果是ArtInterpreterToInterpreterBridge就不需要调用一层跳板。

解释器内部就会把环境准备好。（指的是ShadowFrame）



## 参数传递



1.最开始会在art_quick_invoke_static_stub中的INVOKE_STUB_CREATE_FRAME宏中会将参数保存到堆栈中

2.之后的artQuickToInterpreterBridge方法中的shadow_frame_builder.VisitArguments会调用从#1中的堆栈中获取参数值

# Refs



thanks for all～

[知乎-android jvm跳板函数分析](https://zhuanlan.zhihu.com/p/521498157)

[掘金-JNI Trampoline分析](https://juejin.cn/post/7249288285809524773)

[博客-YAHFA--ART环境下的Hook框架](https://rk700.github.io/2017/03/30/YAHFA-introduction/)

[博客-类加载解析](https://huanle19891345.github.io/en/android/art/2%E7%B1%BB%E5%8A%A0%E8%BD%BD/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%B1%82/)w
