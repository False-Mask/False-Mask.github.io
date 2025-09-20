---
title: android-art-trampoline
tags:
cover:
---



# Adroid Art Tranpoline





# pre



## Invoke

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



## 一层跳板



### art_quick_invoke_static_stub



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



```asm
.macro INVOKE_STUB_CREATE_FRAME
SAVE_SIZE=8*8   // x4, x5, <padding>, x19, x20, x21, FP, LR saved.
    SAVE_TWO_REGS_INCREASE_FRAME x4, x5, SAVE_SIZE
    SAVE_REG      x19,      24
    SAVE_TWO_REGS x20, x21, 32
    SAVE_TWO_REGS xFP, xLR, 48

    mov xFP, sp                            // Use xFP for frame pointer, as it's callee-saved.
    .cfi_def_cfa_register xFP

    add x10, x2, #(__SIZEOF_POINTER__ + 0xf) // Reserve space for ArtMethod*, arguments and
    and x10, x10, # ~0xf                   // round up for 16-byte stack alignment.
    sub sp, sp, x10                        // Adjust SP for ArtMethod*, args and alignment padding.

    mov xSELF, x3                          // Move thread pointer into SELF register.

    // Copy arguments into stack frame.
    // Use simple copy routine for now.
    // 4 bytes per slot.
    // X1 - source address
    // W2 - args length
    // X9 - destination address.
    // W10 - temporary
    add x9, sp, #8                         // Destination address is bottom of stack + null.

    // Copy parameters into the stack. Use numeric label as this is a macro and Clang's assembler
    // does not have unique-id variables.
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





### art_quick_invoke_stub



# art跳板



## art_quick_to_interpreter_bridge 

art到interpreter的执行跳板函数

1.开辟栈帧

2.调用artQuickToInterpreterBridge方法，并通过寄存器传参

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

5.将被调用方存入堆栈中, 将

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



#### ManagedStack



### EnterInterpreterFromEntryPoint

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



### Execute



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



### ExecuteSwitch



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





### ExecuteSwitchImpl



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



### ExecuteSwitchImplAsm



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



### ExecuteSwitchImplCpp



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



## 



# QA





## Art 跳板什么时候会被写入



类加载链接阶段即LinkCode函数中写入



# Refs



[知乎-android jvm跳板函数分析](https://zhuanlan.zhihu.com/p/521498157)

