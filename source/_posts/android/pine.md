---
title: Art Hook之Pine实现原理
tags:
  - android
  - art hook
cover: >-
  https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/pine-hook.drawio.png
date: 2025-12-21 23:57:37
---


# Art Hook之Pine实现原理



# 关于

本文主要是对Pine core实现原理进行分析，聚焦arm64平台 & art虚拟机


# 概览

- pine trampolines

pine hook有两种api

1.Pine.hook桥接到统一的桥接方法然后做方法的桥接执行

2.Pine. hookReplace桥接到自定义的桥接方法，自己管理方法的执行(感觉像是新加的功能，整体不是非常完善)

其中每一种api中依据实现原理还有一个分叉：

1.inline Mode：使用inline hook实现程序流程的拦截

2.replace Mode：使用方法执行入口替换的方式实现程序执行流程的拦截。

![pine-trampolines.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/pine-trampolines.drawio.png)

图pine trampoline列表



- pine hook 

![pine-hook.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/pine-hook.drawio.png)

图pine hook核心实现原理(tips：虽然提供了replace Mode & inline Mode，当前主要使用的逻辑是replace Mode，本着看都看了的心态对所有逻辑都进行类分析，读者可按需查看)

# 入口

执行逻辑如下：
1.前置判断过滤
2.初始化
3.全局回调——hook方法调用前
4.获取方法底层C++的ArtMethod地址
5.记录hook函数
6.处理方法hook
7.全局回调——hook方法调用后

拆分整合掉不重要的内容。最终我们知道Pine hook的核心流程：
- 初始化
- 方法记录
- 方法Hook
```java
public static MethodHook.Unhook hook(Member method, MethodHook callback) {  
    return hook(method, callback, true);  
}

public static MethodHook.Unhook hook(Member method, MethodHook callback, boolean canInitDeclaringClass) {
	if (PineConfig.debug)
		Log.d(TAG, "Hooking method " + method + " with callback " + callback);
	
	// 1.前置判断过滤
	if (method == null) throw new NullPointerException("method == null");
	if (callback == null) throw new NullPointerException("callback == null");

	int modifiers = method.getModifiers();
	if (method instanceof Method) {
		if (Modifier.isAbstract(modifiers)) // 不hook抽象方法
			throw new IllegalArgumentException("Cannot hook abstract methods: " + method);
		((Method) method).setAccessible(true);
	} else if (method instanceof Constructor) {
		// 不hook静态构造函数
		if (Modifier.isStatic(modifiers)) // TODO: We really cannot hook this?
			throw new IllegalArgumentException("Cannot hook class initializer: " + method);
		((Constructor<?>) method).setAccessible(true);
	} else {
		// 不hook除普通方法 & 构造方法以外的其他内容
		throw new IllegalArgumentException("Only methods and constructors can be hooked: " + method);
	}

	// 2.初始化
	ensureInitialized();

	HookListener hookListener = sHookListener;
	// 3.全局回调——hook方法调用前
	if (hookListener != null)
		hookListener.beforeHook(method, callback);

	// 4.获取方法底层C++的ArtMethod地址
	long artMethod = getArtMethod(method);
	HookRecord hookRecord;
	boolean newMethod = false;
	
	// 5.记录hook函数
	synchronized (sHookLock) {
		hookRecord = sHookRecords.get(artMethod);
		if (hookRecord == null) {
			newMethod = true;
			hookRecord = new HookRecord(method, artMethod);
			sHookRecords.put(artMethod, hookRecord);
		}
	}

	// 6.处理方法hook
	MethodHook.Unhook unhook = sHookHandler.handleHook(hookRecord, callback, modifiers,
			newMethod, canInitDeclaringClass);

	// 7.全局回调——hook方法调用后
	if (hookListener != null)
		hookListener.afterHook(method, unhook);

	return unhook;
}
```



# Hook核心逻辑分析


## 初始化

```java
// 1. 最外层调用
ensureInitialized();

// 2.内层初始化调用，去重，防止多线程异常。
public static void ensureInitialized() {  
    if (initialized) return;  
    synchronized (Pine.class) {  
        if (initialized) return;  
        initialize();  
        initialized = true;  
    }  
}
```


执行初始化逻辑，具体实现逻辑如下
1.调用init0初始化参数
2.调用initBridgeMethods初始化桥接方法
3.调用enableFastNative启用fastNative功能(为native方法准备的)
```java
private static void initialize() {
	int sdkLevel = PineConfig.sdkLevel;
	// 不支持android 4.4以下
	if (sdkLevel < Build.VERSION_CODES.KITKAT)
		throw new RuntimeException("Unsupported android sdk level " + sdkLevel);
	// 不支持dalvik虚拟机，只支持art虚拟机
	String vmVersion = System.getProperty("java.vm.version");
	if (vmVersion == null || !vmVersion.startsWith("2"))
		throw new RuntimeException("Only supports ART runtime");
	// 针对不同的系统版本选择hook模式
	// 8.0以前 INLINE_WITHOUT_JIT： 使用inlne hook进行art hook, 如果方法没被jit则fallback到entry_point替换（替换ArtMethod的入口）
	// 8.0以及以后 REPLACEMENT： entry_point替换。 
	hookMode = sdkLevel < Build.VERSION_CODES.O ? HookMode.INLINE_WITHOUT_JIT : HookMode.REPLACEMENT;

	try {
		// pine native so加载 
		LibLoader libLoader = PineConfig.libLoader;
		if (libLoader != null) libLoader.loadLib();
		// native初始化
		init0(sdkLevel, PineConfig.debug, PineConfig.debuggable, PineConfig.antiChecks,
				PineConfig.disableHiddenApiPolicy, PineConfig.disableHiddenApiPolicyForPlatformDomain);
		// 入口函数初始化
		initBridgeMethods();

		// fastNative初始化
		if (PineConfig.useFastNative && sdkLevel >= Build.VERSION_CODES.LOLLIPOP)
			enableFastNative();
	} catch (Exception e) {
		throw new RuntimeException("Pine init error", e);
	}
}
```


### init0

```c++

void Pine_init0(JNIEnv* env, jclass Pine, jint androidVersion, jboolean debug, jboolean debuggable,
        jboolean antiChecks, jboolean disableHiddenApiPolicy, jboolean disableHiddenApiPolicyForPlatformDomain) {
    if (debug == JNI_TRUE) LOGI("Pine native init...");
    // 参数值保存
    PineConfig::debug = static_cast<bool>(debug);
    PineConfig::debuggable = static_cast<bool>(debuggable);
    PineConfig::anti_checks = static_cast<bool>(antiChecks);
    // 获取TrampolineInstaller
    TrampolineInstaller::GetOrInitDefault(); // trigger TrampolineInstaller::default_ initialization
    Android::Init(env, androidVersion, disableHiddenApiPolicy, disableHiddenApiPolicyForPlatformDomain);
    {
        ScopedLocalClassRef Ruler(env, "top/canyie/pine/Ruler");
        auto m1 = art::ArtMethod::Require(env, Ruler.Get(), "m1", "(F)V", true);
        auto m2 = art::ArtMethod::Require(env, Ruler.Get(), "m2", "()V", true);

        uint32_t expected_access_flags;
        do {
            ScopedLocalClassRef Method(env, "java/lang/reflect/Method");
            jmethodID getAccessFlags = Method.FindMethodID("getAccessFlags", "()I");
            if (LIKELY(getAccessFlags != nullptr)) {
                ScopedLocalRef javaM1(env, env->ToReflectedMethod(
                        Ruler.Get(), m1->ToMethodID(), JNI_TRUE));
                expected_access_flags = static_cast<uint32_t>(env->CallIntMethod(
                        javaM1.Get(), getAccessFlags));

                if (LIKELY(!env->ExceptionCheck())) break;

                LOGW("Method.getAccessFlags threw exception unexpectedly, use default access flags.");
                env->ExceptionDescribe();
                env->ExceptionClear();
            } else {
                LOGW("Method.getAccessFlags not found, use default access flags.");
            }
            expected_access_flags = AccessFlags::kPrivate | AccessFlags::kStatic | AccessFlags::kNative;
        } while (false);

        if (androidVersion >= Android::kQ) {
            expected_access_flags |= AccessFlags::kPublicApi;
        }

        ScopedLocalClassRef I(env, "top/canyie/pine/Ruler$I");
        auto abstract_method = art::ArtMethod::Require(env, I.Get(), "m", "()V", false);
        art::ArtMethod::InitMembers(env, m1, m2, abstract_method, expected_access_flags);

        if (UNLIKELY(!art::ArtMethod::GetQuickToInterpreterBridge())) {
            // This is a workaround for art_quick_to_interpreter_bridge not found.
            // This case is almost impossible to enter
            // because its symbols are found almost always on all devices.
            // But if it happened... Try to get it with an abstract method (it is not compilable
            // and its entry is art_quick_to_interpreter_bridge)
            // Note: We DO NOT use platform's abstract methods
            // because their entry may not be interpreter entry.

            LOGE("art_quick_to_interpreter_bridge not found, try workaround");

            void* entry = abstract_method->GetEntryPointFromCompiledCode();
            LOGE("New art_quick_to_interpreter_bridge %p", entry);
            art::ArtMethod::SetQuickToInterpreterBridge(entry);
        }
    }

#define SET_JAVA_VALUE(name, sig, value) \
if (auto field = env->GetStaticFieldID(Pine, (name), (sig)); (sig)[0] == 'I') env->SetStaticIntField(Pine, field, (value)); \
else env->SetStaticLongField(Pine, field, (value));

    SET_JAVA_VALUE("arch", "I", kCurrentArch);
    SET_JAVA_VALUE("openElf", "J", reinterpret_cast<jlong>(PineOpenElf));
    SET_JAVA_VALUE("findElfSymbol", "J", reinterpret_cast<jlong>(PineGetElfSymbolAddress));
    SET_JAVA_VALUE("closeElf", "J", reinterpret_cast<jlong>(PineCloseElf));
    SET_JAVA_VALUE("getMethodDeclaringClass", "J", reinterpret_cast<jlong>(GetMethodDeclaringClass));
    SET_JAVA_VALUE("syncMethodEntry", "J", reinterpret_cast<jlong>(SyncMethodEntry));
    SET_JAVA_VALUE("suspendVM", "J", reinterpret_cast<jlong>(PineSuspendVM));
    SET_JAVA_VALUE("resumeVM", "J", reinterpret_cast<jlong>(PineResumeVM));
#undef SET_JAVA_VALUE
}
```


#### GetOrInitDefault

根据预定义的编译器宏判断要返回什么对象。(兼容不同的架构类型)
Note: 后续只针对于Arm64进行分析

```c++

TrampolineInstaller* TrampolineInstaller::default_ = nullptr;

TrampolineInstaller* TrampolineInstaller::GetOrInitDefault() {
	// 根据预定义的编译器宏判断要返回什么对象。(对其架构类型)
    if (default_ == nullptr) {
#ifdef __aarch64__
        default_ = new Arm64TrampolineInstaller;
#elif defined(__arm__)
        default_ = new Thumb2TrampolineInstaller;
#elif defined(__i386__)
        default_ = new X86TrampolineInstaller;
#endif
        default_->Init();
    }
    return default_;
}
```


初始化
```c++

namespace pine {
    class TrampolineInstaller {
	    // ......
    
	    void Init() {  
		    // 初始化Trampoline
		    InitTrampolines();  
		    // 初始化trampoline大小
		    kBridgeJumpTrampolineSize = SubAsSize(kMethodJumpTrampoline, kBridgeJumpTrampoline);  
		    kMethodJumpTrampolineSize = SubAsSize(kCallOriginTrampoline, kMethodJumpTrampoline);  
		    kCallOriginTrampolineSize = SubAsSize(kBackupTrampoline, kCallOriginTrampoline);  
		    kBackupTrampolineSize = SubAsSize(kTrampolinesEnd, kBackupTrampoline);  
		}
		
		// ......
	}
}
    
```

- InitTrampolines
主要初始化trampline的地址以及参数对于trampoline入口的偏移量
```c++
void Arm64TrampolineInstaller::InitTrampolines() {
	// directJump 
    kDirectJumpTrampoline = AS_VOID_PTR(pine_direct_jump_trampoline);
    kDirectJumpTrampolineEntryOffset = DirectJumpTrampolineOffset(
            AS_VOID_PTR(pine_direct_jump_trampoline_jump_entry));
的
	// bridgeJump
    kBridgeJumpTrampoline = AS_VOID_PTR(pine_bridge_jump_trampoline);
    kBridgeJumpTrampolineTargetMethodOffset = BridgeJumpTrampolineOffset(
            AS_VOID_PTR(pine_bridge_jump_trampoline_target_method));
    kBridgeJumpTrampolineExtrasOffset = BridgeJumpTrampolineOffset(
            AS_VOID_PTR(pine_bridge_jump_trampoline_extras));
    kBridgeJumpTrampolineBridgeMethodOffset = BridgeJumpTrampolineOffset(
            AS_VOID_PTR(pine_bridge_jump_trampoline_bridge_method));
    kBridgeJumpTrampolineBridgeEntryOffset = BridgeJumpTrampolineOffset(
            AS_VOID_PTR(pine_bridge_jump_trampoline_bridge_entry));
    kBridgeJumpTrampolineOriginCodeEntryOffset = BridgeJumpTrampolineOffset(
            AS_VOID_PTR(pine_bridge_jump_trampoline_call_origin_entry));

	// methodJump
    kMethodJumpTrampoline = AS_VOID_PTR(pine_method_jump_trampoline);
    kMethodJumpTrampolineDestMethodOffset = MethodJumpTrampolineOffset(
            AS_VOID_PTR(pine_method_jump_trampoline_dest_method));
    kMethodJumpTrampolineDestEntryOffset = MethodJumpTrampolineOffset(
            AS_VOID_PTR(pine_method_jump_trampoline_dest_entry));
	
	// callOrigin
    kCallOriginTrampoline = AS_VOID_PTR(pine_call_origin_trampoline);
    kCallOriginTrampolineOriginMethodOffset = CallOriginTrampolineOffset(
            AS_VOID_PTR(pine_call_origin_trampoline_origin_method));
    kCallOriginTrampolineOriginalEntryOffset = CallOriginTrampolineOffset(
            AS_VOID_PTR(pine_call_origin_trampoline_origin_code_entry));

	// backUpTrampoline
    kBackupTrampoline = AS_VOID_PTR(pine_backup_trampoline);
    kBackupTrampolineOriginMethodOffset = BackupTrampolineOffset(
            AS_VOID_PTR(pine_backup_trampoline_origin_method));
    kBackupTrampolineOverrideSpaceOffset = BackupTrampolineOffset(
            AS_VOID_PTR(pine_backup_trampoline_override_space));
    kBackupTrampolineRemainingCodeEntryOffset = BackupTrampolineOffset(
            AS_VOID_PTR(pine_backup_trampoline_remaining_code_entry));
	
	// trampoline结束标记
    kTrampolinesEnd = AS_VOID_PTR(pine_trampolines_end);

    kDirectJumpTrampolineSize = 16;
}
```


1.pine_direct_jump_trampoline
作用：用于代码跳转，一般作为最外层而跳板方法拦截原始方法的执行流程。
执行流程：方法内需要有一个参数输入pine_direct_jump_trampoline_jump_entry， 跳板会将其先加载到x17中，并执行无条件跳转。
```c++

#define FUNCTION(name) \
.data; \
.align 4; \
.global name; \
name:

#define VAR(name) \
.global name; \
name:

FUNCTION(pine_direct_jump_trampoline)  
LDVAR(x17, pine_direct_jump_trampoline_jump_entry)  
br x17  
VAR(pine_direct_jump_trampoline_jump_entry)  
.long 0  
.long 0
```

2.pine_bridge_jump_trampoline
作用：作为一个桥接方法，将方法执行流程桥接到其他的方法中去。
执行流程：先执行判断过滤掉没有被hook的方法，然后获取锁防止线程并发问题，最后拼接组装参数跳转到桥接方法中去。
```c++
FUNCTION(pine_bridge_jump_trampoline)
// 加载需要hook的artMethod，并对比是否一致
// 不一致直接调用jump_to_original，跳转到原始方法。
LDVAR(x17, pine_bridge_jump_trampoline_target_method)
cmp x0, x17
bne jump_to_original
// 当前方法就是需要hook的方法
LDVAR(x17, pine_bridge_jump_trampoline_extras)
// 获取锁
b acquire_lock

// 锁获取异常进行低功耗等待
lock_failed:
wfe // Wait other thread to release the lock

// 尝试获取锁
acquire_lock:
ldaxr w16, [x17]
cbz w16, lock_failed // lock_flag == 0 (has other thread holding the lock), fail.
stlxr w16, wzr, [x17] // try set lock_flag to 0
cbnz w16, lock_failed // failed, try again.

// Now we hold the lock!
// 锁获取成功
// 设置参数，准备调用hook方法
str x1, [x17, #4]
str x2, [x17, #12]
str x3, [x17, #20]
str d0, [x17, #28]
str d1, [x17, #36]
str d2, [x17, #44]
str d3, [x17, #52]
str d4, [x17, #60]
str d5, [x17, #68]
str d6, [x17, #76]
str d7, [x17, #84]
mov x1, x0 // first param = callee ArtMethod
mov x2, x17 // second param = extras (saved x1, x2, x3)
mov x3, sp // third param = sp
LDVAR(x0, pine_bridge_jump_trampoline_bridge_method)
LDVAR(x17, pine_bridge_jump_trampoline_bridge_entry)
br x17

// 跳转到origin
jump_to_original:
LDVAR(x17, pine_bridge_jump_trampoline_call_origin_entry)
br x17
// 参数列表
VAR(pine_bridge_jump_trampoline_target_method)
.long 0
.long 0
VAR(pine_bridge_jump_trampoline_extras)
.long 0
.long 0
VAR(pine_bridge_jump_trampoline_bridge_method)
.long 0
.long 0
VAR(pine_bridge_jump_trampoline_bridge_entry)
.long 0
.long 0
VAR(pine_bridge_jump_trampoline_call_origin_entry)
.long 0
.long 0
```
3.pine_method_jump_trampoline
作用：类似于pine_direct_jump_trampoline，是另外一种拦截代码执行流程的方式。(Pine目前提供了两种api做方法hook这个是实现的另外一种api,具体后面讲)
执行流程：修改ArtMethod对象即x0寄存器(Art在执行的过程中会将x0记录作为被调用方法)，加载hooker方法地址，并执行代码跳转。(一般是跳转到二级跳板。)
```c++
FUNCTION(pine_method_jump_trampoline)
LDVAR(x0, pine_method_jump_trampoline_dest_method)
LDVAR(x17, pine_method_jump_trampoline_dest_entry)
br x17
VAR(pine_method_jump_trampoline_dest_method)
.long 0
.long 0
VAR(pine_method_jump_trampoline_dest_entry)
.long 0
.long 0
```
4.pine_call_origin_trampoline
逻辑上和pine_method_jump_trampoline一模一样。不过目前来看是Thumb2在用这玩意。
```c++
FUNCTION(pine_call_origin_trampoline)
LDVAR(x0, pine_call_origin_trampoline_origin_method)
LDVAR(x17, pine_call_origin_trampoline_origin_code_entry)
br x17
VAR(pine_call_origin_trampoline_origin_method)
.long 0
.long 0
VAR(pine_call_origin_trampoline_origin_code_entry)
.long 0
.long 0
```
5.pine_backup_trampoline

作用：用于实现调用原始方法
执行逻辑：同pine_method_jump_trampoline & pine_call_origin_trampoline
```c++
FUNCTION(pine_backup_trampoline)  
LDVAR(x0, pine_backup_trampoline_origin_method)  
VAR(pine_backup_trampoline_override_space)  
.long 0 // 4 bytes (will be overwritten)  
.long 0 // 4 bytes (will be overwritten)  
.long 0 // 4 bytes (will be overwritten)  
.long 0 // 4 bytes (will be overwritten)  
nop // 4 bytes, may be overwritten for anti checks  
nop // 4 bytes, may be overwritten for anti checks  
LDVAR(x17, pine_backup_trampoline_remaining_code_entry)  
br x17  
VAR(pine_backup_trampoline_origin_method)  
.long 0  
.long 0  
VAR(pine_backup_trampoline_remaining_code_entry)  
.long 0  
.long 0
```
6.pine_trampolines_end
```c++
FUNCTION(pine_trampolines_end)
```

#### Android::init

主要还是一些art符号的寻找操作，方便后续直接使用

```c++
Android::Init(env, androidVersion, disableHiddenApiPolicy, disableHiddenApiPolicyForPlatformDomain);
```

```c++
void Android::Init(JNIEnv* env, int sdk_version, bool disable_hiddenapi_policy, bool disable_hiddenapi_policy_for_platform) {

	// 通过JNI获取java vm
    Android::version = sdk_version;
    if (UNLIKELY(env->GetJavaVM(&jvm_) != JNI_OK)) {
        LOGF("Cannot get java vm");
        env->FatalError("Cannot get java vm");
        abort();
    }

    {
	    // 加载art runtime so文件
        bool eng_build = false;
        ElfImage art_lib_handle("libart.so");
        if (UNLIKELY(!art_lib_handle.IsOpened())) {
            // Running on eng build ROMs?
            art_lib_handle.RelativeOpen("libartd.so", true, true);
            if (LIKELY(art_lib_handle.IsOpened())) {
                eng_build = true;
            } else {
                // Alibaba YunOS AOC runtime?
                constexpr const char* kLibAocPath = "/system/lib"
#ifdef __LP64__
                                                    "64"
#endif
                                                    "/libaoc.so";
                if (access(kLibAocPath, R_OK) == 0) {
                    art_lib_handle.Open(kLibAocPath, true, true);
                }
            }
        }

        if (Android::version >= Android::kR) {
			// 获取如下方法地址
        	// art::ScopedSuspendAll::ScopedSuspendAll(char const*, bool)
			// art::ScopedSuspendAll::~ScopedSuspendAll()
			// art::gc::ScopedGCCriticalSection::ScopedGCCriticalSection(art::Thread*, art::gc::GcCause, art::gc::CollectorType)
			// art::gc::ScopedGCCriticalSection::~ScopedGCCriticalSection()
        
            suspend_all = reinterpret_cast<void (*)(void*, const char*, bool)>(art_lib_handle.GetSymbolAddress(
                    "_ZN3art16ScopedSuspendAllC1EPKcb"));
            resume_all = reinterpret_cast<void (*)(void*)>(art_lib_handle.GetSymbolAddress(
                    "_ZN3art16ScopedSuspendAllD1Ev"));
            if (UNLIKELY(!suspend_all || !resume_all)) {
                LOGE("SuspendAll API is unavailable.");
                suspend_all = nullptr;
                resume_all = nullptr;
            } else {
                start_gc_critical_section = reinterpret_cast<void (*)(void*, void*, art::GcCause,
                        art::CollectorType)>(art_lib_handle.GetSymbolAddress(
                        "_ZN3art2gc23ScopedGCCriticalSectionC2EPNS_6ThreadENS0_7GcCauseENS0_13CollectorTypeE"));
                end_gc_critical_section = reinterpret_cast<void (*)(void*)>(art_lib_handle.GetSymbolAddress(
                        "_ZN3art2gc23ScopedGCCriticalSectionD2Ev"));
                if (UNLIKELY(!start_gc_critical_section || !end_gc_critical_section)) {
                    LOGE("GC critical section API is unavailable.");
                    start_gc_critical_section = nullptr;
                    end_gc_critical_section = nullptr;
                }
            }
        } else {
	        // art::Dbg::SuspendVM()
	        // art::Dbg::ResumeVM()
        
            suspend_vm = reinterpret_cast<void (*)()>(art_lib_handle.GetSymbolAddress(
                    "_ZN3art3Dbg9SuspendVMEv")); // art::Dbg::SuspendVM()
            resume_vm = reinterpret_cast<void (*)()>(art_lib_handle.GetSymbolAddress(
                    "_ZN3art3Dbg8ResumeVMEv")); // art::Dbg::ResumeVM()
            if (UNLIKELY(!suspend_vm || !resume_vm)) {
                LOGE("Suspend VM API is unavailable.");
                suspend_vm = nullptr;
                resume_vm = nullptr;
            }
        }

		// 反射禁用
        if (Android::version >= Android::kP)
            DisableHiddenApiPolicy(&art_lib_handle, disable_hiddenapi_policy, disable_hiddenapi_policy_for_platform);

        art::Thread::Init(&art_lib_handle);
        art::ArtMethod::Init(&art_lib_handle);
        // JIT API is not supported yet in Android R+
        if (UNLIKELY(sdk_version >= kN && sdk_version < kR)) {
            ElfImage jit_lib_handle(eng_build ? "libartd-compiler.so" : "libart-compiler.so", true, false);
            art::Jit::Init(&art_lib_handle, &jit_lib_handle);
        }

        InitMembersFromRuntime(jvm_, &art_lib_handle);
    }

    WellKnownClasses::Init(env);
}
```

- art::Thread::Init(&art_lib_handle)
```c++
void Thread::Init(const ElfImage* handle) {
    if (Android::version == Android::kL || Android::version == Android::kLMr1) {
        // This function is needed to create the backup method on Lollipop.
        // Below M, an ArtMethod is actually a instance of java.lang.reflect.ArtMethod, can't use malloc()
        // It should be immovable. On Kitkat, moving gc is unimplemented in art, so it can't be moved
        // but on Lollipop, this object may be moved by gc, so we need to ensure it is non-movable.
        alloc_non_movable = reinterpret_cast<void* (*)(void*, Thread*)>(handle->GetSymbolAddress(
                // art::mirror::Class::AllocNonMovableObject(art::Thread*)
                "_ZN3art6mirror5Class21AllocNonMovableObjectEPNS_6ThreadE"));
    }
    
    // 获取当前的线程对象

    current = reinterpret_cast<Thread* (*)()>(handle->GetSymbolAddress(
            "_ZN3art6Thread14CurrentFromGdbEv")); // art::Thread::CurrentFromGdb()

    if (UNLIKELY(!current && Android::version < Android::kN)) {
        current = reinterpret_cast<Thread* (*)()>(handle->GetSymbolAddress(
                "_ZN3art6Thread7CurrentEv")); // art::Thread::Current()
        if (UNLIKELY(!current)) {
            key_self = static_cast<pthread_key_t*>(handle->GetSymbolAddress(
                    "_ZN3art6Thread17pthread_key_self_E")); // art::Thread::pthread_key_self_
        }
    }

    new_local_ref = reinterpret_cast<jobject (*)(JNIEnv*, void*)>(handle->GetSymbolAddress(
            "_ZN3art9JNIEnvExt11NewLocalRefEPNS_6mirror6ObjectE")); // art::JNIEnvExt::NewLocalRef(art::mirror::Object *)

	// 创建局部变量

    if (UNLIKELY(!new_local_ref)) {
        LOGW("JNIEnvExt::NewLocalRef is unavailable, try JavaVMExt::AddWeakGlobalReference");
        const char* add_global_weak_ref_symbol;
        if (Android::version < Android::kM) {
            // art::JavaVMExt::AddWeakGlobalReference(art::Thread *, art::mirror::Object *)
            add_global_weak_ref_symbol = "_ZN3art9JavaVMExt22AddWeakGlobalReferenceEPNS_6ThreadEPNS_6mirror6ObjectE";
        } else if (Android::version < Android::kO) {
            // art::JavaVMExt::AddWeakGlobalRef(art::Thread *, art::mirror::Object *)
            add_global_weak_ref_symbol = "_ZN3art9JavaVMExt16AddWeakGlobalRefEPNS_6ThreadEPNS_6mirror6ObjectE";
        } else {
            // art::JavaVMExt::AddWeakGlobalRef(art::Thread *, art::ObjPtr<art::mirror::Object>)
            add_global_weak_ref_symbol = "_ZN3art9JavaVMExt16AddWeakGlobalRefEPNS_6ThreadENS_6ObjPtrINS_6mirror6ObjectEEE";
        }
        add_weak_global_ref = reinterpret_cast<jweak (*)(JavaVM*, Thread*, void*)>(
                handle->GetSymbolAddress(add_global_weak_ref_symbol));
    }
    
    // 将jobject转为art::mirror::Object

    decode_jobject = reinterpret_cast<void* (*)(Thread*, jobject)>(handle->GetSymbolAddress(
            "_ZNK3art6Thread13DecodeJObjectEP8_jobject", false)); // art::Thread::DecodeJObject(_jobject *)
}
```

- art::ArtMethod::Init(&art_lib_handle)
```c++
void ArtMethod::Init(const ElfImage* handle) {
	// 获取解释的入口:
    art_quick_to_interpreter_bridge = handle->GetSymbolAddress("art_quick_to_interpreter_bridge");
    art_quick_generic_jni_trampoline = handle->GetSymbolAddress("art_quick_generic_jni_trampoline");
    ExecuteNterpImpl = handle->GetSymbolAddress("ExecuteNterpImpl", false);

    // Alibaba YunOS AOC runtime?
    // 兼容阿里 yunos
    if (UNLIKELY(!art_quick_to_interpreter_bridge))
        art_quick_to_interpreter_bridge = handle->GetSymbolAddress("aoc_quick_to_interpreter_bridge");
    if (UNLIKELY(!art_quick_generic_jni_trampoline))
        art_quick_generic_jni_trampoline = handle->GetSymbolAddress("aoc_quick_generic_jni_trampoline");
	// <= android7.0兼容适配
    if (Android::version < Android::kN) {
        art_interpreter_to_compiled_code_bridge = handle->GetSymbolAddress(
                "artInterpreterToCompiledCodeBridge");
        art_interpreter_to_interpreter_bridge = handle->GetSymbolAddress(
                "artInterpreterToInterpreterBridge");
    }

	// 获取ArtMethod::CopyFrom方法
    const char* symbol_copy_from = nullptr;
    if (Android::version >= Android::kO) {
        // art::ArtMethod::CopyFrom(art::ArtMethod *, art::PointerSize)
        symbol_copy_from = "_ZN3art9ArtMethod8CopyFromEPS0_NS_11PointerSizeE";
    } else if (Android::version >= Android::kN) {
#ifdef __LP64__
        // art::ArtMethod::CopyFrom(art::ArtMethod *, unsigned long)
        symbol_copy_from = "_ZN3art9ArtMethod8CopyFromEPS0_m";
#else
        // art::ArtMethod::CopyFrom(art::ArtMethod *, unsigned int)
        symbol_copy_from = "_ZN3art9ArtMethod8CopyFromEPS0_j";
#endif
    } else if (Android::version >= Android::kM) {
#ifdef __LP64__
        // art::ArtMethod::CopyFrom(art::ArtMethod const *, unsigned long)
        symbol_copy_from = "_ZN3art9ArtMethod8CopyFromEPKS0_m";
#else
        // art::ArtMethod::CopyFrom(art::ArtMethod const *, unsigned int)
        symbol_copy_from = "_ZN3art9ArtMethod8CopyFromEPKS0_j";
#endif
    }

	// 将方法进行强行转化
    if (symbol_copy_from)
        copy_from = reinterpret_cast<void (*)(ArtMethod*, ArtMethod*, size_t)>(
                handle->GetSymbolAddress(symbol_copy_from));

    if (UNLIKELY(Android::version == Android::kO))
        throw_invocation_time_error = reinterpret_cast<void (*)(ArtMethod*)>(handle->GetSymbolAddress(
                "_ZN3art9ArtMethod24ThrowInvocationTimeErrorEv"));
}

```
- art::Jit::Init(&art_lib_handle, &jit_lib_handle)
```c++
void Jit::Init(const ElfImage* art_lib_handle, const ElfImage* jit_lib_handle) {

	// art::jit::Jit::jit_compiler_handle_
    global_compiler_ptr = static_cast<JitCompiler**>(art_lib_handle->GetSymbolAddress(
            "_ZN3art3jit3Jit20jit_compiler_handle_E"));

	// jit_load方法
    auto jit_load = reinterpret_cast<JitCompiler* (*)(bool*)>(jit_lib_handle->GetSymbolAddress(
            "jit_load"));

	// 创建jitCompiler
    if (LIKELY(jit_load)) {
        bool generate_debug_info = false;
        self_compiler = jit_load(&generate_debug_info);
    } else {
        LOGW("Failed to create new JitCompiler: jit_load not found");
    }

    // FIXME: jit_compile_method doesn't exist in Android R
    void* jit_compile_method = jit_lib_handle->GetSymbolAddress("jit_compile_method");

    if (Android::version >= Android::kQ) {
        Jit::jit_compile_method_q = reinterpret_cast<bool (*)(void*, void*, void*, bool, bool)>(jit_compile_method);
        // Android Q, ART may update CompilerOptions and the value we set will be overwritten.
        // the function pointer saved in art::jit::Jit::jit_update_options_ .
        Jit::jit_update_options_ptr = static_cast<void**>(art_lib_handle->GetSymbolAddress(
                "_ZN3art3jit3Jit19jit_update_options_E"));
    } else {
        Jit::jit_compile_method = reinterpret_cast<bool (*)(void*, void*, void*, bool)>(jit_compile_method);
    }

    // fields count from compiler_filter_ (not included) to inline_max_code_units_ (not included)
    // FIXME Offset for inline_max_code_units_ seems to be incorrect on my Pixel 3 (Android 10)...
    // FIXME Structure of CompilerOptions has changed in Android R.
    unsigned thresholds_count = Android::version >= Android::kO ? 5 : 6;

    CompilerOptions_inline_max_code_units = new Member<void, size_t>(
            sizeof(void*) + thresholds_count * sizeof(size_t));
}
```
- InitMembersFromRuntime
主要是ClassLinker和JitCodeCache
```c++

void Android::InitMembersFromRuntime(JavaVM* jvm, const ElfImage* handle) {  
    if (version < kQ) {  
        // ClassLinker is unnecessary before R.  
        // JIT was added in Android N but MoveObsoleteMethod was added in Android O        // and I didn't find a stable way to retrieve jit code cache until Q        // from Runtime object, so try to retrieve from ProfileSaver.        // TODO: Still clearing jit info on Android N but only for jit-compiled methods.  
        if (version >= kO) {  
            InitJitCodeCache(nullptr, 0, handle);  
        }  
        return;  
    }  
    void** instance_ptr = static_cast<void**>(handle->GetSymbolAddress("_ZN3art7Runtime9instance_E"));  
    void* runtime;  
    if (UNLIKELY(!instance_ptr || !(runtime = *instance_ptr))) {  
        LOGE("Unable to retrieve Runtime.");  
        return;  
    }  
  
    // If SmallIrtAllocator symbols can be found, then the ROM has merged commit "Initially allocate smaller local IRT"  
    // This commit added a pointer member between `class_linker_` and `java_vm_`. Need to calibrate offset here.    // https://android.googlesource.com/platform/art/+/4dcac3629ea5925e47b522073f3c49420e998911    // https://github.com/crdroidandroid/android_art/commit/aa7999027fa830d0419c9518ab56ceb7fcf6f7f1    // https://android.googlesource.com/platform/art/+/849d09a81907f16d8ccc6019b8baf86a304b730c    bool has_smaller_irt = version >= kT  
            || handle->HasSymbol("_ZN3art17SmallIrtAllocator10DeallocateEPNS_8IrtEntryE")  
            || handle->HasSymbol("_ZN3art3jni17SmallLrtAllocatorC2Ev");  
  
    std::vector<size_t> known_offsets = OffsetOfJavaVm(has_smaller_irt);  
    size_t jvm_offset = 0;  
    for (size_t offset : known_offsets) {  
        auto val = reinterpret_cast<std::unique_ptr<JavaVM>*>(  
                reinterpret_cast<uintptr_t>(runtime) + offset)->get();  
        if (val == jvm) {  
            jvm_offset = offset;  
            break;  
        }  
    }  
    if (UNLIKELY(!jvm_offset)) {  
        LOGW("JavaVM offset mismatches default offsets, trying a linear search");  
        int offset = Memory::FindOffset(runtime, jvm, 1024, 4);  
        if (UNLIKELY(offset == -1)) {  
            LOGE("Failed to find java vm from Runtime");  
            return;  
        }  
        jvm_offset = offset;  
        LOGW("Found JavaVM in Runtime at %zu", jvm_offset);  
    }  
    InitClassLinker(runtime, jvm_offset, handle, has_smaller_irt);  
    InitJitCodeCache(runtime, jvm_offset, handle);  
}
```
- WellKnownClasses::Init(env)
初始化类型
```c++
jclass WellKnownClasses::java_lang_reflect_ArtMethod = nullptr;  
jfieldID WellKnownClasses::java_lang_reflect_Executable_artMethod = nullptr;  
void WellKnownClasses::Init(JNIEnv* env) {  
    java_lang_reflect_ArtMethod = FindClass(env, "java/lang/reflect/ArtMethod");  
    if (UNLIKELY(Android::version >= Android::kR)) {  
        java_lang_reflect_Executable_artMethod = RequireNonStaticFieldID(env,  
                "java/lang/reflect/Executable", "artMethod", "J");  
    }  
}
```

#### calculate offset 

通过预定义的Ruler类定位artMethod各个字段的位置
```c++
{
        ScopedLocalClassRef Ruler(env, "top/canyie/pine/Ruler");
        auto m1 = art::ArtMethod::Require(env, Ruler.Get(), "m1", "(F)V", true);
        auto m2 = art::ArtMethod::Require(env, Ruler.Get(), "m2", "()V", true);

		// 通过对比实际定义的类，artMethod各个字段的偏移
        uint32_t expected_access_flags;
        do {
            ScopedLocalClassRef Method(env, "java/lang/reflect/Method");
            jmethodID getAccessFlags = Method.FindMethodID("getAccessFlags", "()I");
            if (LIKELY(getAccessFlags != nullptr)) {
                ScopedLocalRef javaM1(env, env->ToReflectedMethod(
                        Ruler.Get(), m1->ToMethodID(), JNI_TRUE));
                expected_access_flags = static_cast<uint32_t>(env->CallIntMethod(
                        javaM1.Get(), getAccessFlags));

                if (LIKELY(!env->ExceptionCheck())) break;

                LOGW("Method.getAccessFlags threw exception unexpectedly, use default access flags.");
                env->ExceptionDescribe();
                env->ExceptionClear();
            } else {
                LOGW("Method.getAccessFlags not found, use default access flags.");
            }
            expected_access_flags = AccessFlags::kPrivate | AccessFlags::kStatic | AccessFlags::kNative;
        } while (false);

        if (androidVersion >= Android::kQ) {
            expected_access_flags |= AccessFlags::kPublicApi;
        }

        ScopedLocalClassRef I(env, "top/canyie/pine/Ruler$I");
        auto abstract_method = art::ArtMethod::Require(env, I.Get(), "m", "()V", false);
        art::ArtMethod::InitMembers(env, m1, m2, abstract_method, expected_access_flags);
		
		// 没找到artMethod的art_quick_to_interpreter_bridge函数地址
		// 通过获取abstract方法的entry_point作为其值。
        if (UNLIKELY(!art::ArtMethod::GetQuickToInterpreterBridge())) {
            // This is a workaround for art_quick_to_interpreter_bridge not found.
            // This case is almost impossible to enter
            // because its symbols are found almost always on all devices.
            // But if it happened... Try to get it with an abstract method (it is not compilable
            // and its entry is art_quick_to_interpreter_bridge)
            // Note: We DO NOT use platform's abstract methods
            // because their entry may not be interpreter entry.

            LOGE("art_quick_to_interpreter_bridge not found, try workaround");

            void* entry = abstract_method->GetEntryPointFromCompiledCode();
            LOGE("New art_quick_to_interpreter_bridge %p", entry);
            art::ArtMethod::SetQuickToInterpreterBridge(entry);
        }
    }
```



#### setJavaValue

将上面获取的值设置到ArtMethod中

```c++
#define SET_JAVA_VALUE(name, sig, value) \
if (auto field = env->GetStaticFieldID(Pine, (name), (sig)); (sig)[0] == 'I') env->SetStaticIntField(Pine, field, (value)); \
else env->SetStaticLongField(Pine, field, (value));

    SET_JAVA_VALUE("arch", "I", kCurrentArch);
    SET_JAVA_VALUE("openElf", "J", reinterpret_cast<jlong>(PineOpenElf));
    SET_JAVA_VALUE("findElfSymbol", "J", reinterpret_cast<jlong>(PineGetElfSymbolAddress));
    SET_JAVA_VALUE("closeElf", "J", reinterpret_cast<jlong>(PineCloseElf));
    SET_JAVA_VALUE("getMethodDeclaringClass", "J", reinterpret_cast<jlong>(GetMethodDeclaringClass));
    SET_JAVA_VALUE("syncMethodEntry", "J", reinterpret_cast<jlong>(SyncMethodEntry));
    SET_JAVA_VALUE("suspendVM", "J", reinterpret_cast<jlong>(PineSuspendVM));
    SET_JAVA_VALUE("resumeVM", "J", reinterpret_cast<jlong>(PineResumeVM));
#undef SET_JAVA_VALUE
```


### initBridgeMethods

逻辑和Epic类似。

```java
private static void initBridgeMethods() {
        try {
            String entryClassName;
            Class<?>[] paramTypes;
			// 根据架构类型选择合适的桥接类
            if (arch == ARCH_ARM64) {
                entryClassName = "top.canyie.pine.entry.Arm64Entry";
                paramTypes = new Class<?>[] {long.class, long.class, long.class,
                        long.class, long.class, long.class, long.class};
            } else if (arch == ARCH_ARM) {
                entryClassName = "top.canyie.pine.entry.Arm32Entry";
                paramTypes = new Class<?>[] {int.class, int.class, int.class};
            } else if (arch == ARCH_X86) {
                entryClassName = "top.canyie.pine.entry.X86Entry";
                paramTypes = new Class<?>[] {int.class, int.class, int.class};
            } else throw new RuntimeException("Unexpected arch " + arch);

            // Use Class.forName() to ensure entry class is initialized.
            Class<?> entryClass = Class.forName(entryClassName, true, Pine.class.getClassLoader());

            String[] bridgeMethodNames = {
                    "voidBridge", "intBridge", "longBridge", "doubleBridge", "floatBridge",
                    "booleanBridge", "byteBridge", "charBridge", "shortBridge", "objectBridge"
            };
			// 将所有的桥接方法设置到map中方便后续使用。
            for (String bridgeMethodName : bridgeMethodNames) {
                Method bridge = entryClass.getDeclaredMethod(bridgeMethodName, paramTypes);
                bridge.setAccessible(true);
                sBridgeMethods.put(bridgeMethodName, bridge);
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to init bridge methods", e);
        }
    }
```


### enableFastNative


```java
// useFastNative是一个可配置的项， 主要用于提速jni方法调用
if (PineConfig.useFastNative && sdkLevel >= Build.VERSION_CODES.LOLLIPOP)
                enableFastNative();
```

实现非常简单，就是设置所有JNI方法的accessFlags添加上kFastNative的标签即可。
```c++
void Pine_enableFastNative(JNIEnv* env, jclass Pine) {  
    LOGI("Experimental feature FastNative is enabled.");  
    for (auto& method_info : gFastNativeMethods) {  
        auto method = art::ArtMethod::Require(env, Pine, method_info.name, method_info.signature, true);  
        assert(method != nullptr);  
        // 设置method AccessFlag
        method->SetFastNative();  
    }  
}
```



## 方法记录

```java

long artMethod = getArtMethod(method);
HookRecord hookRecord;
boolean newMethod = false;

synchronized (sHookLock) {
	hookRecord = sHookRecords.get(artMethod);
	if (hookRecord == null) {
		newMethod = true;
		hookRecord = new HookRecord(method, artMethod);
		sHookRecords.put(artMethod, hookRecord);
	}
}
```


## 方法Hook

```java
// hookRecord —— 方法记录(ArtMethod & Field)
// callback —— 回调Callback
// modifiers —— 方法的修饰符(int值)
// newMethod —— 是否是新创建的方法
// canInitDeclaringClass —— 是否运行初始化类
MethodHook.Unhook unhook = sHookHandler.handleHook(hookRecord, callback, modifiers,
			newMethod, canInitDeclaringClass);
```

具体实现，逻辑如下
1.如果之前没有hook过，需要先安装跳板
2.在跳板安装完成的情况下向hook点添加callback即可。

不难发现其中Hook的核心逻辑在hookNewMethod中
```java
   private static HookHandler sHookHandler = new HookHandler() {
        @Override
        public MethodHook.Unhook handleHook(HookRecord hookRecord, MethodHook hook, int modifiers,
                                            boolean newMethod, boolean canInitDeclaringClass) {
            // 只有首次hook的时候需要调用方法执行hook 
            if (newMethod)
                hookNewMethod(hookRecord, modifiers, canInitDeclaringClass);
                
            // 如果之前hook过直接将方法添加到hook record中    
			
			// 过滤异常case，如果hook callback为直接返回
            if (hook == null) {
                // This can only happen when the up handler pass null manually,
                // just return null and let the up to do remaining everything
                return null;
            }
            hookRecord.addCallback(hook);
            // 创建unhook包装给外部。
            // 另外提一嘴：逻辑比较简单，Unhook会调用HookHandler.handleUnhook
            // 而handleUnhook只是做了一个callback的移除。
            return hook.new Unhook(hookRecord);
        }

        @Override public void handleUnhook(HookRecord hookRecord, MethodHook hook) {
            hookRecord.removeCallback(hook);
        }
    };
```

逻辑如下：
1.调用makeClassesVisiblyInitialized
2.针对于INLINE模式进行一次jit
3.获取桥接方法
4.调用hook0进行hook并获取一个java.lang.reflect.Method(用于回调原始方法)
```java
static void hookNewMethod(HookRecord hookRecord, int modifiers, boolean canInitDeclaringClass) {
        Member method = hookRecord.target;
        final int mode = hookMode;
        // 此处有两种模式会进行inline hook
        // 1. INLINE_WITHOUT_JIT —— inline hook但是不进行jit预编译
        // 2. INLINE —— inline hook但是进行jit编译
        boolean isInlineHook = mode == HookMode.INLINE || mode == HookMode.INLINE_WITHOUT_JIT;

        long thread = currentArtThread0();
        
        // 如果hook方法是Static方法，调用以下static方法以便其能被正常解析。
        if ((hookRecord.isStatic = Modifier.isStatic(modifiers)) && canInitDeclaringClass) {
            resolve((Method) method);
            if (PineConfig.sdkLevel >= Build.VERSION_CODES.Q) {
                // Android R has a new class state called "visibly initialized",
                // and FixupStaticTrampolines will be called after class was initialized.
                // The entry point will be reset. Make this class be visibly initialized before hook
                // Note: this feature does not exist on official Android Q,
                // but some weird ROMs cherry-pick this commit to these Android Q ROMs
                // https://github.com/crdroidandroid/android_art/commit/ef76ced9d2856ac988377ad99288a357697c4fa2
                makeClassesVisiblyInitialized(thread);
            }
        }

        Class<?> declaring = method.getDeclaringClass();

        final boolean jni = Modifier.isNative(modifiers);
        final boolean proxy = Proxy.isProxyClass(declaring);

        // Only try compile target method when trying inline hook.
        // 处理需要inline hook的case
        if (isInlineHook) {
            // Cannot compile native or proxy methods.
            // 如果是INLINE 模式在hook强制进行一次jit编译
            if (!(jni || proxy)) {
                if (mode == HookMode.INLINE) {
                    boolean compiled = compile0(thread, method);
                    if (!compiled) {
                        Log.w(TAG, "Cannot compile the target method, force replacement mode.");
                        isInlineHook = false;
                    }
                }
            } else {
                isInlineHook = false;
            }
        }

        String bridgeMethodName;
        // FIXME: WARNING: The following code will cause parameter types and return type to be initialized!!!
        // 根据入参和返回值获取桥接方法
        if (method instanceof Method) {
            hookRecord.paramTypes = ((Method) method).getParameterTypes();
            Class<?> returnType = ((Method) method).getReturnType();
            bridgeMethodName = returnType.isPrimitive() ? returnType.getName() + "Bridge" : "objectBridge";
        } else {
            hookRecord.paramTypes = ((Constructor<?>) method).getParameterTypes();
            // Constructor is actually a method named <init> and its return type is void.
            bridgeMethodName = "voidBridge";
        }
		// 设置参数
        hookRecord.paramNumber = hookRecord.paramTypes.length;

        hookRecord.bridge = PineConfig.sdkLevel == Build.VERSION_CODES.M && arch == ARCH_ARM64
                ? Arm64MarshmallowEntry.getBridge(bridgeMethodName, hookRecord.paramNumber)
                : sBridgeMethods.get(bridgeMethodName);
        if (hookRecord.bridge == null)
            throw new AssertionError("Cannot find bridge method for " + method);

        Method backup = hook0(thread, declaring, hookRecord, method, hookRecord.bridge, isInlineHook,
                jni, proxy);

        if (backup == null)
            throw new RuntimeException("Failed to hook method " + method);

        backup.setAccessible(true);
        hookRecord.backup = backup;
    }
```

### makeClassesVisiblyInitialized

通过调用art MakeInitializedClassesVisiblyInitialized，设置Class的状态标记位

```c++
void Pine_makeClassesVisiblyInitialized(JNIEnv*, jclass, jlong thread) {  
    Android::MakeInitializedClassesVisiblyInitialized(reinterpret_cast<void*>(thread), true);  
}

static void MakeInitializedClassesVisiblyInitialized(void* thread, bool wait) {  
    // If symbol MakeInitializedClassesVisiblyInitialized not found,  
    // class_linker_ won't be initialized.    
    if (!class_linker_) {  
        return;  
    }  
    // art::ClassLinker::MakeInitializedClassesVisiblyInitialized(art::Thread*, bool)
    make_visibly_initialized_(class_linker_, thread, wait);  
}
```

### jit

触发执行JIT,但是只针对低版本生效，高版本没法用



Java触发位置
```java
boolean compiled = compile0(thread, method);
```

JNI入口位置
```c++
jboolean Pine_compile0(JNIEnv* env, jclass, jlong thread, jobject javaMethod) {  
    return static_cast<jboolean>(art::ArtMethod::FromReflectedMethod(env, javaMethod)->Compile(  
            reinterpret_cast<art::Thread*>(thread)));  
}
```
编译前置过滤
```c++
// art_method.h
bool Compile(Thread* thread) {  
	// 如果已经编译过了，直接返回true 
    if (LIKELY(IsCompiled())) return true;
    //  sdk < 24 不触发
    if (UNLIKELY(Android::version < Android::kN)) return false;
    // 如果Pine禁用了jit,直接返回  
    if (UNLIKELY(!PineConfig::jit_compilation_allowed)) return false;  
    // 如果类的accessFlag标记了不进行jit, 直接返回
    if (UNLIKELY(HasAccessFlags(kAccCompileDontBother))) return false;  
    return Jit::CompileMethod(thread, this);  
}
```

实际的jit方法
```c++
bool Jit::CompileMethod(Thread* thread, void* method) {  
	// sdk >= 30 直接返回
    if (LIKELY(Android::version >= Android::kR)) {  
        LOGW("JIT compilation is not supported in Android R yet");  
        return false;  
    }  
    // 获取jit compiler
    void* compiler = GetCompiler();  
    if (UNLIKELY(!compiler)) {  
        LOGE("No JitCompiler available for JIT compilation!");  
        return false;  
    }  
  
    bool result;  
  
    // JIT compilation will modify the state of the thread, so we backup and restore it after compilation.  
    // jit会修改线程的flag,调用前备份下
    int32_t origin_state_and_flags = thread->GetStateAndFlags();  
	// 调用jit方法
    if (jit_compile_method) {  
        result = jit_compile_method(compiler, method, thread, false/*osr*/);  
    } else if (jit_compile_method_q) {  
        result = jit_compile_method_q(compiler, method, thread, false/*baseline*/, false/*osr*/);  
    } else {  
        LOGE("Compile method failed: jit_compile_method not found");  
        return false;  
    }  
	// reset jit 
    thread->SetStateAndFlags(origin_state_and_flags);  
    return result;  
}
```
### getBridge

获取桥接方法，所有被Pine hook的Java方法都会被桥接到这个Brige方法中进行代码的分发执行。

```c++
String bridgeMethodName;  
// FIXME: WARNING: The following code will cause parameter types and return type to be initialized!!!  
if (method instanceof Method) {   // 普通方法
	// 记录方法参数、返回值、拼接桥接方法名称(根据返回值)
    hookRecord.paramTypes = ((Method) method).getParameterTypes();  
    Class<?> returnType = ((Method) method).getReturnType();  
    bridgeMethodName = returnType.isPrimitive() ? returnType.getName() + "Bridge" : "objectBridge";  
} else {   // 构造方法
    hookRecord.paramTypes = ((Constructor<?>) method).getParameterTypes();  
    // Constructor is actually a method named <init> and its return type is void.  
    bridgeMethodName = "voidBridge";  
}  
  
// 记录方法入参长度
hookRecord.paramNumber = hookRecord.paramTypes.length;  
// sdk == 23 & arm64架构： Arm64MarshmallowEntry.getBridge
// others： 通过桥接方法名称获取桥接方法。(初始化的时候会将所有桥接方法保存到sBridgeMethods中)
hookRecord.bridge = PineConfig.sdkLevel == Build.VERSION_CODES.M && arch == ARCH_ARM64  
        ? Arm64MarshmallowEntry.getBridge(bridgeMethodName, hookRecord.paramNumber)  
        : sBridgeMethods.get(bridgeMethodName);  
if (hookRecord.bridge == null)  
    throw new AssertionError("Cannot find bridge method for " + method);
```

### hook0



执行hook操作，核心逻辑如下：

1.字段初始化
2.创建backup方法
3.安装trampoline 并 初始化backup方法，以便后续调用
4.将origin方法的入口地址设置到Record中

```c++
// threadAddress 当前native thread对象
// declaring hookee方法的declaring class对象
// hookRecord HookRecord对象(用于保存hook记录)
// javaTarget hookee Method对象
// javaBridge hooker Method对象
// isInlineHook 是否是inline hook mode
// isJni hookee方法是否是JnI方法
// isProxy hookee的declaringClass是否为Proxy方法
jobject Pine_hook0(JNIEnv* env, jclass, jlong threadAddress, jclass declaring, jobject hookRecord,
                   jobject javaTarget, jobject javaBridge, jboolean isInlineHook, jboolean isJni,
                   jboolean isProxy) {
    // 字段初始化
    // 获取HookRecord中 trampoline字段的fieldId
    jfieldID HookRecord_trampoline = GetHookRecordTrampolineField(env, hookRecord);
    auto thread = reinterpret_cast<art::Thread*>(threadAddress);
    auto target = art::ArtMethod::FromReflectedMethod(env, javaTarget); // 获取hookee方法底层C++ 的 ArtMethod实例
    auto bridge = art::ArtMethod::FromReflectedMethod(env, javaBridge); // 获取hooker方法底层C++ 的 ArtMethod实例

    if (PineConfig::jit_compilation_allowed && PineConfig::auto_compile_bridge) {
        // The bridge method entry will be hardcoded in the trampoline, subsequent optimization
        // operations that require modification of the bridge method entry will not take effect.
        // Try to do JIT compilation first to get the best performance.
        bridge->Compile(thread);
    }

    bool is_inline_hook = JBOOL_TRUE(isInlineHook);
    const bool is_native = JBOOL_TRUE(isJni);
    const bool is_proxy = JBOOL_TRUE(isProxy);
    const bool is_native_or_proxy = is_native || is_proxy;

    TrampolineInstaller* trampoline_installer = TrampolineInstaller::GetDefault();

    if (is_inline_hook && (trampoline_installer->IsReplacementOnly() || !target->IsCompiled())) {
        is_inline_hook = false;
    }

    if (UNLIKELY(is_inline_hook && trampoline_installer->CannotSafeInlineHook(target))) {
        LOGW("Cannot safe inline hook the target method, force replacement mode.");
        is_inline_hook = false;
    }

    bool skip_first_few_bytes = PineConfig::anti_checks
            && is_inline_hook && trampoline_installer->CanSkipFirstFewBytes(target);

	// 创建backup方法
    art::ArtMethod* backup;
    if (WellKnownClasses::java_lang_reflect_ArtMethod) {
        // If ArtMethod has mirror class in java, we cannot use malloc to direct
        // allocate an instance because it must has a record in Runtime.

        backup = static_cast<art::ArtMethod*>(thread->AllocNonMovable(
                WellKnownClasses::java_lang_reflect_ArtMethod));
        if (UNLIKELY(!backup)) {
#if __ANDROID_API__ < __ANDROID_API_L__
            // On Android kitkat, moving gc is not supported in art. All objects are immovable.
            if (UNLIKELY(Android::version >= Android::kL)) {
#endif
                LOGE("Failed to allocate an immovable object for creating backup method.");
                env->ExceptionClear();
#if __ANDROID_API__ < __ANDROID_API_L__
            }
#endif

            jobject javaBackup = env->AllocObject(WellKnownClasses::java_lang_reflect_ArtMethod);
            if (UNLIKELY(env->ExceptionCheck())) {
                LOGE("Can't create the backup method!");
                return nullptr;
            }
            backup = static_cast<art::ArtMethod*>(thread->DecodeJObject(javaBackup));
        }
    } else {
        backup = art::ArtMethod::New();
        if (UNLIKELY(!backup)) {
            int local_errno = errno;
            LOGE("Cannot allocate backup ArtMethod, errno %d(%s)", errno, strerror(errno));
            if (local_errno == ENOMEM) {
                JNIHelper::Throw(env, "java/lang/OutOfMemoryError",
                                 "No memory for allocate backup method");
            } else {
                JNIHelper::Throw(env, "java/lang/RuntimeException",
                                 "hook failed: cannot allocate backup method");
            }
            return nullptr;
        }
    }

    void* new_entrypoint;
    char error_msg[288];
    // 安装trampoline
    {
        // ArtMethod objects are very important. Many threads depend on their values,
        // so we need to suspend other threads to avoid errors.
        ScopedSuspendVM suspend_vm(thread);

        void* call_origin = is_inline_hook
                            ? trampoline_installer->InstallInlineTrampoline(target, bridge, skip_first_few_bytes)
                            : trampoline_installer->InstallReplacementTrampoline(target, bridge);

        if (LIKELY(call_origin)) {
            backup->BackupFrom(target, call_origin, is_inline_hook, is_native, is_proxy);
            target->AfterHook(is_inline_hook, is_native_or_proxy);
            new_entrypoint = target->GetEntryPointFromCompiledCode();
        } else {
            snprintf(error_msg, sizeof(error_msg), "Failed to install %s trampoline on method %p: %s (%d).",
                     is_inline_hook ? "inline" : "replacement", target, strerror(errno), errno);
            if (errno == EACCES || errno == EPERM)
                strlcat(error_msg, " This is a security failure, check selinux policy, seccomp or capabilities. Earlier log may point out root cause.", sizeof(error_msg));
            LOGE("%s", error_msg);
            new_entrypoint = nullptr;
        }
    }

	// 将entry point设置到HookRecord中保存  
    if (LIKELY(new_entrypoint)) {
        env->SetLongField(hookRecord, HookRecord_trampoline, reinterpret_cast<jlong>(new_entrypoint));
        // 返回将c++底层的ArtMethod对象转化为 java.lang.reflect.Method
        return env->ToReflectedMethod(declaring, backup->ToMethodID(),
                                      static_cast<jboolean>(backup->IsStatic()));
    } else {
        JNIHelper::Throw(env, errno == EACCES || errno == EPERM ? "java/lang/SecurityException" : "java/lang/RuntimeException", error_msg);
        return nullptr;
    }
}


```

#### backup方法创建

总结一下就是：
1.如果有java.lang.reflect.ArtMethod类型，则使用它创建ArtMethod对象
	a. 5.0、5.1的系统版本通过AllocNonMovableObject创建ArtMethod对象(版本差异)
	b. 其他版本使用env->AllocObject创建ArtMethod对象
2.若无则使用malloc分配特定大小的内存。

```c++
art::ArtMethod* backup;
if (WellKnownClasses::java_lang_reflect_ArtMethod) {
	// 如果ArtMethod的java mirror类型能够获取到

	// If ArtMethod has mirror class in java, we cannot use malloc to direct
	// allocate an instance because it must has a record in Runtime.

	// 先尝试调用art::mirror::Class::AllocNonMovableObject new对象。(应该只针对于androd 5.0、5.1的版本)
	// 如果失败再尝试使用env->AllocObject开启新的内存空间
	backup = static_cast<art::ArtMethod*>(thread->AllocNonMovable(
			WellKnownClasses::java_lang_reflect_ArtMethod));
	if (UNLIKELY(!backup)) {
#if __ANDROID_API__ < __ANDROID_API_L__
		// On Android kitkat, moving gc is not supported in art. All objects are immovable.
		if (UNLIKELY(Android::version >= Android::kL)) {
#endif
			LOGE("Failed to allocate an immovable object for creating backup method.");
			env->ExceptionClear();
#if __ANDROID_API__ < __ANDROID_API_L__
		}
#endif

		jobject javaBackup = env->AllocObject(WellKnownClasses::java_lang_reflect_ArtMethod);
		if (UNLIKELY(env->ExceptionCheck())) {
			LOGE("Can't create the backup method!");
			return nullptr;
		}
		backup = static_cast<art::ArtMethod*>(thread->DecodeJObject(javaBackup));
	}
} else {
	// 如果ArtMethod的java mirror类型无法获取到
	// 直接使用malloc操作new一块大小相同的空间

	backup = art::ArtMethod::New();
	
	// 这要是还不行那就真没招了。直接抛异常了。
	if (UNLIKELY(!backup)) {
		int local_errno = errno;
		LOGE("Cannot allocate backup ArtMethod, errno %d(%s)", errno, strerror(errno));
		if (local_errno == ENOMEM) {
			JNIHelper::Throw(env, "java/lang/OutOfMemoryError",
							 "No memory for allocate backup method");
		} else {
			JNIHelper::Throw(env, "java/lang/RuntimeException",
							 "hook failed: cannot allocate backup method");
		}
		return nullptr;
	}
}
```


#### trampoline安装

```c++
void* new_entrypoint;
char error_msg[288];
// 安装trampoline
{
	// ArtMethod objects are very important. Many threads depend on their values,
	// so we need to suspend other threads to avoid errors.
	// 先挂起其他所有线程，修改ArtMethod是一个非常危险的行为
	ScopedSuspendVM suspend_vm(thread);
	// 针对于inline hook & entryPoint单独处理，并返回一个方法句柄用于调用原始方法。
	void* call_origin = is_inline_hook
						? trampoline_installer->InstallInlineTrampoline(target, bridge, skip_first_few_bytes)
						: trampoline_installer->InstallReplacementTrampoline(target, bridge);
	// 将hookee方法的内容拷贝到backup 方法中
	if (LIKELY(call_origin)) {
		backup->BackupFrom(target, call_origin, is_inline_hook, is_native, is_proxy);
		target->AfterHook(is_inline_hook, is_native_or_proxy);
		new_entrypoint = target->GetEntryPointFromCompiledCode();
	} else {
		snprintf(error_msg, sizeof(error_msg), "Failed to install %s trampoline on method %p: %s (%d).",
				 is_inline_hook ? "inline" : "replacement", target, strerror(errno), errno);
		if (errno == EACCES || errno == EPERM)
			strlcat(error_msg, " This is a security failure, check selinux policy, seccomp or capabilities. Earlier log may point out root cause.", sizeof(error_msg));
		LOGE("%s", error_msg);
		new_entrypoint = nullptr;
	}
}

if (LIKELY(new_entrypoint)) {
	env->SetLongField(hookRecord, HookRecord_trampoline, reinterpret_cast<jlong>(new_entrypoint));
	return env->ToReflectedMethod(declaring, backup->ToMethodID(),
								  static_cast<jboolean>(backup->IsStatic()));
} else {
	JNIHelper::Throw(env, errno == EACCES || errno == EPERM ? "java/lang/SecurityException" : "java/lang/RuntimeException", error_msg);
	return nullptr;
}

```


##### InstallInlineTrampoline


```c++
// target —— hookee方法的底层C++的ArtMethod对象
// replacement —— hooker方法的地处C++的ArtMethod对象
// skip_first_few_bytes  —— 是否跳过前几个字节(绕过检测)
void* TrampolineInstaller::InstallInlineTrampoline(art::ArtMethod* target, art::ArtMethod* bridge,
                                                   bool skip_first_few_bytes) {
	// 设置内存为rwx
    void* target_code_addr = target->GetCompiledCodeAddr();
    bool target_code_writable = Memory::Unprotect(target_code_addr);
    if (UNLIKELY(!target_code_writable)) {
        LOGE("Failed to make target code writable!");
        return nullptr;
    }
	// 创建backup trampoline
    size_t backup_size = kDirectJumpTrampolineSize;
    if (skip_first_few_bytes) backup_size += kSkipBytes;

    void* backup = Backup(target, backup_size);
    if (UNLIKELY(!backup)) return nullptr;
	// 创建bridge jump trampoline
    void* bridge_jump_trampoline = CreateBridgeJumpTrampoline(target, bridge, backup);
    if (UNLIKELY(!bridge_jump_trampoline)) return nullptr;
	// 创建填充direct jump trampoline
    {
        ScopedMemoryAccessProtection protection(target_code_addr, kDirectJumpTrampolineSize);
        if (skip_first_few_bytes) {
            FillWithNopImpl(target_code_addr, kSkipBytes);
            WriteDirectJumpTrampolineTo(AS_VOID_PTR(AS_PTR_NUM(target_code_addr) + kSkipBytes),
                    bridge_jump_trampoline);
        } else {
            WriteDirectJumpTrampolineTo(target_code_addr, bridge_jump_trampoline);
        }
    }

    if (PineConfig::debug)
        LOGD("InstallInlineTrampoline: target_code_addr %p backup %p bridge_jump %p",
                target_code_addr, backup, bridge_jump_trampoline);

    return backup;
}


```

- 内存保护
	就是将compile code地址的内存权限通过mprotect设置为rwx.
```c++ 
// 获取hookee方法compiled_code地址， 并去除内存保护(方便后续修改)
void* target_code_addr = target->GetCompiledCodeAddr();
bool target_code_writable = Memory::Unprotect(target_code_addr);
if (UNLIKELY(!target_code_writable)) {
	LOGE("Failed to make target code writable!");
	return nullptr;
}

// memory.h
// 调用mprotect将页面内存权限设置为rwx(可读、可写、可执行)
static inline bool Unprotect(void* ptr) {
	size_t alignment = (uintptr_t) ptr % page_size;
	void* aligned_ptr = (void*) ((uintptr_t) ptr - alignment);
	int result = mprotect(aligned_ptr, page_size, PROT_READ | PROT_WRITE | PROT_EXEC);
	if (UNLIKELY(result == -1)) {
		LOGE("mprotect failed for %p: %s (%d)", ptr, strerror(errno), errno);
		return false;
	}
	return true;
}
```
- 创建backup方法

```c++
size_t backup_size = kDirectJumpTrampolineSize;
if (skip_first_few_bytes) backup_size += kSkipBytes;

void* backup = Backup(target, backup_size);
if (UNLIKELY(!backup)) return nullptr;
```


```c++
void* TrampolineInstaller::Backup(art::ArtMethod* target, size_t size) {  
	// 调用mmap分配内存页
    void* mem = Memory::AllocUnprotected(kBackupTrampolineSize);  
    if (UNLIKELY(!mem)) {  
        LOGE("Failed to allocate executable memory for backup!");  
        return nullptr;  
    }  
    // 将backup Trampoline拷贝到新分配的内存中(Arm64下kBackupTrampolineSize为16)
    memcpy(mem, kBackupTrampoline, kBackupTrampolineSize);  
    uintptr_t addr = reinterpret_cast<uintptr_t>(mem);  
	// 填充backup Trampoline的pine_backup_trampoline_origin_method参数
    auto origin_out = reinterpret_cast<art::ArtMethod**>(addr +  
                                                         kBackupTrampolineOriginMethodOffset);  
    *origin_out = target;  
	// 将原始方法的前4字节进行备份 (Arm64平台下)
    void* target_addr = target->GetEntryPointFromCompiledCode();  
    memcpy(AS_VOID_PTR(addr + kBackupTrampolineOverrideSpaceOffset), target_addr, size);  
  
	// 填充pine_backup_trampoline_remaining_code_entry即剩余的指令首地址。
    if (LIKELY(target->GetCompiledCodeSize() != size)) {  
        // has remaining code  
        void** remaining_out = reinterpret_cast<void**>(addr +  
                                                        kBackupTrampolineRemainingCodeEntryOffset);  
        *remaining_out = AS_VOID_PTR(reinterpret_cast<uintptr_t>(target_addr) + size);  
    }  
  
    // 缓存刷新
    Memory::FlushCache(mem, kBackupTrampolineSize);  
    return mem;  
}
```
backUpTrampoline
执行逻辑：先执行加载原始方法地址到x0中，然后执行预留空间的汇编，最后通过br无条件跳转到甚于的指令中去。
```c++
FUNCTION(pine_backup_trampoline)  
// 将pine_backup_trampoline_origin_method也就是跳转方法
LDVAR(x0, pine_backup_trampoline_origin_method)  
// 预留空间
VAR(pine_backup_trampoline_override_space)  
.long 0 // 4 bytes (will be overwritten)  
.long 0 // 4 bytes (will be overwritten)  
.long 0 // 4 bytes (will be overwritten)  
.long 0 // 4 bytes (will be overwritten)  
nop // 4 bytes, may be overwritten for anti checks  
nop // 4 bytes, may be overwritten for anti checks  
// 甚于的代码地址。
LDVAR(x17, pine_backup_trampoline_remaining_code_entry)  
br x17  
VAR(pine_backup_trampoline_origin_method)  
.long 0  
.long 0  
VAR(pine_backup_trampoline_remaining_code_entry)  
.long 0  
.long 0  
  
FUNCTION(pine_trampolines_end)
```
- 创建methodJumpTrampoline
```c++
void* bridge_jump_trampoline = CreateBridgeJumpTrampoline(target, bridge, backup);
if (UNLIKELY(!bridge_jump_trampoline)) return nullptr;
```


```c++
void*
TrampolineInstaller::CreateBridgeJumpTrampoline(art::ArtMethod* target, art::ArtMethod* bridge,
                                                void* origin_code_entry) {
    // 分配rwx内存区域用于装载trampoline                                           
    void* mem = Memory::AllocUnprotected(kBridgeJumpTrampolineSize);
    if (UNLIKELY(!mem)) {
        LOGE("Failed to allocate bridge jump trampoline!");
        return nullptr;
    }
    // 将pine_bridge_jump_trampoline方法拷贝到新开辟的内存中
    memcpy(mem, kBridgeJumpTrampoline, kBridgeJumpTrampolineSize);
    uintptr_t addr = reinterpret_cast<uintptr_t>(mem);
	// 填充跳板pine_bridge_jump_trampoline_target_method字段
    auto target_out = reinterpret_cast<art::ArtMethod**>(addr +
                                                         kBridgeJumpTrampolineTargetMethodOffset);
    *target_out = target;
	// 填充跳板pine_bridge_jump_trampoline_extras字段
    auto extras_out = reinterpret_cast<Extras**>(addr + kBridgeJumpTrampolineExtrasOffset);
    *extras_out = new Extras;
	// 填充跳板pine_bridge_jump_trampoline_bridge_method字段
    auto bridge_out = reinterpret_cast<art::ArtMethod**>(addr +
                                                         kBridgeJumpTrampolineBridgeMethodOffset);
    *bridge_out = bridge;
	// 填充跳板pine_bridge_jump_trampoline_bridge_entry字段
    auto bridge_entry_out = reinterpret_cast<void**>(addr + kBridgeJumpTrampolineBridgeEntryOffset);
    *bridge_entry_out = bridge->GetEntryPointFromCompiledCode();
	// 填充跳板pine_bridge_jump_trampoline_call_origin_entry字段
    auto origin_entry_out = reinterpret_cast<void**>(addr +
                                                     kBridgeJumpTrampolineOriginCodeEntryOffset);
    *origin_entry_out = origin_code_entry;
	// 刷新缓存
    Memory::FlushCache(mem, kBridgeJumpTrampolineSize);

    return mem;
}
```

先过滤不需要hook的方法，跳转执行原始方法pine_bridge_jump_trampoline_call_origin_entry，然后再获取锁，若获取失败则进入低功耗等待状态。获取则构造组装参数并调用hook方法。
```c++
FUNCTION(pine_bridge_jump_trampoline)
// 挑选出需要hook的方法
// (不需要hook的方法会跳转到jump_to_original)
LDVAR(x17, pine_bridge_jump_trampoline_target_method)
cmp x0, x17
bne jump_to_original

// 加载pine_bridge_jump_trampoline_extras到x17
LDVAR(x17, pine_bridge_jump_trampoline_extras)
// 获取锁
b acquire_lock

// 锁获取失败
lock_failed:
wfe // Wait other thread to release the lock

// 获取锁
acquire_lock:
ldaxr w16, [x17]
cbz w16, lock_failed // lock_flag == 0 (has other thread holding the lock), fail.
stlxr w16, wzr, [x17] // try set lock_flag to 0
cbnz w16, lock_failed // failed, try again.

// Now we hold the lock!
// 获取锁成功，将x1~x3, d0~d7设置到[x17 + 4,x17 + 92]
str x1, [x17, #4]
str x2, [x17, #12]
str x3, [x17, #20]
str d0, [x17, #28]
str d1, [x17, #36]
str d2, [x17, #44]
str d3, [x17, #52]
str d4, [x17, #60]
str d5, [x17, #68]
str d6, [x17, #76]
str d7, [x17, #84]
// 设置参数调用hook方法
// 第一个参数：原始方法x0
// 第二个参数：extra参数
// 第三个参数：sp寄存器
mov x1, x0 // first param = callee ArtMethod
mov x2, x17 // second param = extras (saved x1, x2, x3)
mov x3, sp // third param = sp
LDVAR(x0, pine_bridge_jump_trampoline_bridge_method) // 修改x0的取值
LDVAR(x17, pine_bridge_jump_trampoline_bridge_entry)
br x17

// 跳转到origin方法中
jump_to_original:
LDVAR(x17, pine_bridge_jump_trampoline_call_origin_entry)
br x17
VAR(pine_bridge_jump_trampoline_target_method) // hookee方法
.long 0
.long 0
VAR(pine_bridge_jump_trampoline_extras) // 额外的参数列表
.long 0
.long 0
VAR(pine_bridge_jump_trampoline_bridge_method) // 
.long 0
.long 0
VAR(pine_bridge_jump_trampoline_bridge_entry)
.long 0
.long 0
VAR(pine_bridge_jump_trampoline_call_origin_entry)
.long 0
.long 0
```

- 填充jumpTrampoline

```c++
// 填充jumpTrampoline
{
    // 内存保护
	ScopedMemoryAccessProtection protection(target_code_addr, kDirectJumpTrampolineSize);
	// skip前面的字节码
	if (skip_first_few_bytes) {
		// arm64平台下会将前8个字节设置为两个nop指令。
		FillWithNopImpl(target_code_addr, kSkipBytes); // kSkipBytes arm64平台默认为8
		WriteDirectJumpTrampolineTo(AS_VOID_PTR(AS_PTR_NUM(target_code_addr) + kSkipBytes),
									method_jump_trampoline);
	} else {
		WriteDirectJumpTrampolineTo(target_code_addr, method_jump_trampoline);
	}
}

if (PineConfig::debug)
	LOGD("InstallInlineTrampoline: target_code_addr %p backup %p jump_trampoline %p",
		 target_code_addr, backup, method_jump_trampoline);

return backup;
```

directJumpTrampoline填充逻辑
```c++
void TrampolineInstaller::WriteDirectJumpTrampolineTo(void* mem, void* jump_to) {
	// 将pine_direct_jump_trampoline拷贝到mem中
    memcpy(mem, kDirectJumpTrampoline, kDirectJumpTrampolineSize);
    // 填充pine_direct_jump_trampoline_jump_entry
    void* to_out = AS_VOID_PTR(reinterpret_cast<uintptr_t>(mem) + kDirectJumpTrampolineEntryOffset);
    memcpy(to_out, &jump_to, PTR_SIZE);
    // 刷新缓存
    Memory::FlushCache(mem, kDirectJumpTrampolineSize);
}

```
directTrampoline汇编代码
```c++
FUNCTION(pine_direct_jump_trampoline)  
LDVAR(x17, pine_direct_jump_trampoline_jump_entry)  
br x17  
VAR(pine_direct_jump_trampoline_jump_entry)  
.long 0  
.long 0
```


##### InstallReplacementTrampoline

整体逻辑如下：
1.创建bridge jump trampoline
2.将hookee方法entry point替换为#1创建的trampoline中
```c++
void*  
TrampolineInstaller::InstallReplacementTrampoline(art::ArtMethod* target, art::ArtMethod* bridge) {  
	// #1 创建bridge jump trampoline
    void* origin_code_entry = target->GetEntryPointFromCompiledCode();  
    void* bridge_jump_trampoline = CreateBridgeJumpTrampoline(target, bridge, origin_code_entry);  
    if (UNLIKELY(!bridge_jump_trampoline)) return nullptr;  
  
	// #2 入口替换
	// 这里注释是想说明：如果是不使用入口替换，比如替换原始代码的字节序列，修改x0值后，如果正好触发了堆栈回溯可能会有出现崩溃。
	// 最后选择了入口替换的方案
    // Unknown bug:  
    // After setting the r0 register to the original method, if the original method needs to be    
    // traced back to the call stack (such as an exception), the thread will become a zombie thread    
    // and there will be no response. Just set origin code entry and don't create call_origin_trampoline    
    // to set r0 register to avoid it.  
    // void *call_origin_trampoline = CreateCallOriginTrampoline(target, origin_code_entry);    
    // if (UNLIKELY(!call_origin_trampoline)) return nullptr;  
    target->SetEntryPointFromCompiledCode(bridge_jump_trampoline);  
    // return call_origin_trampoline;  
  
    if (PineConfig::debug)  
        LOGD("InstallReplacementTrampoline: origin %p origin_entry %p bridge_jump %p",  
                target, origin_code_entry, bridge_jump_trampoline);  
	// #3 返回原始的代码地址
    return origin_code_entry;  
}
```



# HookReplace逻辑分析

类似前面的Pine.hook API,只是在API的使用上提供了很大的灵活性。支持自己指定backup方法


## hookReplace

```java
// Pine.java
// hookRecord - HookRecord记录(一个对象保存了hookee方法的Method对象 & ArtMethod对象)
// replacement - hooker方法的Method对象
// backup - 用于call origin的方法
public static Method hookReplace(HookRecord hookRecord, Method replacement, Method backup,  
                                 boolean canInitDeclaringClass) {  
    // 1. 保存hook record信息
    Member method = hookRecord.target;  
    long artMethod = getArtMethod(method);  
    synchronized (sHookRecords) {  
        if (sHookRecords.containsKey(artMethod))  
            throw new IllegalStateException("Attempting to re-hook " + method);  
        sHookRecords.put(artMethod, hookRecord);  
    }  
    int modifiers = method.getModifiers();  
    final int mode = hookMode;  
    boolean isInlineHook = mode != HookMode.REPLACEMENT;  
  
    long thread = currentArtThread0();  
    // 2.  hook static方法前进行方法初始化(可选)
    if ((hookRecord.isStatic = Modifier.isStatic(modifiers)) && canInitDeclaringClass) {  
        resolve((Method) method);  
        if (PineConfig.sdkLevel >= Build.VERSION_CODES.Q) {  
            // Android R has a new class state called "visibly initialized",  
            // and FixupStaticTrampolines will be called after class was initialized.            
            // The entry point will be reset. Make this class be visibly initialized before hook            
            // Note: this feature does not exist on official Android Q,            
            // but some weird ROMs cherry-pick this commit to these Android Q ROMs            
            // https://github.com/crdroidandroid/android_art/commit/ef76ced9d2856ac988377ad99288a357697c4fa2            
            makeClassesVisiblyInitialized(thread);  
        }  
    }  
	  
    Class<?> declaring = method.getDeclaringClass();  
  
    final boolean jni = Modifier.isNative(modifiers);  
    final boolean proxy = Proxy.isProxyClass(declaring);  
  
    // Only try compile target method when trying inline hook.  
    // 3. 针对于inline hook触发一次jit过程
    if (isInlineHook) {  
        // Cannot compile native or proxy methods.  
        if (!(jni || proxy)) {  
            if (mode == HookMode.INLINE) {  
                boolean compiled = compile0(thread, method);  
                if (!compiled) {  
                    Log.w(TAG, "Cannot compile the target method, force replacement mode.");  
                    isInlineHook = false;  
                }  
            }  
        } else {  
            isInlineHook = false;  
        }  
    }  
  
    hookRecord.bridge = replacement;  
    hookRecord.skipUpdateDeclaringClass = true;  
	// 4. 调用hookReplace执行hook  
    backup = hookReplace0(thread, declaring, hookRecord, method, replacement, backup,  
            isInlineHook, jni, proxy);  
  
    if (backup == null)  
        throw new RuntimeException("Failed to hook method " + method);  
  
    backup.setAccessible(true);  
    return hookRecord.backup = backup;  
}
```


### 保存

```java
// 1. 保存hook record信息
Member method = hookRecord.target;  
long artMethod = getArtMethod(method);  
synchronized (sHookRecords) {  
	if (sHookRecords.containsKey(artMethod))  
		throw new IllegalStateException("Attempting to re-hook " + method);  
	sHookRecords.put(artMethod, hookRecord);  
}  
int modifiers = method.getModifiers();  
final int mode = hookMode;  
boolean isInlineHook = mode != HookMode.REPLACEMENT;  

long thread = currentArtThread0(); 
```


### 初始化static方法


在canInitDeclaringClass设置为true的情况下，并且方法类型是static的情况下，通过调用static方法对static进行初始化。

```java
// 2.  hook static方法前进行方法初始化(可选)
if ((hookRecord.isStatic = Modifier.isStatic(modifiers)) && canInitDeclaringClass) {  
	resolve((Method) method);  
	if (PineConfig.sdkLevel >= Build.VERSION_CODES.Q) {  
		// Android R has a new class state called "visibly initialized",  
		// and FixupStaticTrampolines will be called after class was initialized.            
		// The entry point will be reset. Make this class be visibly initialized before hook            
		// Note: this feature does not exist on official Android Q,            
		// but some weird ROMs cherry-pick this commit to these Android Q ROMs            
		// https://github.com/crdroidandroid/android_art/commit/ef76ced9d2856ac988377ad99288a357697c4fa2            
		makeClassesVisiblyInitialized(thread);  
	}  
}  
```


### 触发jit

```java
Class<?> declaring = method.getDeclaringClass();  
  
final boolean jni = Modifier.isNative(modifiers);  
final boolean proxy = Proxy.isProxyClass(declaring);  

// Only try compile target method when trying inline hook.  
// 3. 针对于inline hook触发一次jit过程
if (isInlineHook) {  
	// Cannot compile native or proxy methods.  
	if (!(jni || proxy)) {  
		if (mode == HookMode.INLINE) {  
			boolean compiled = compile0(thread, method);  
			if (!compiled) {  
				Log.w(TAG, "Cannot compile the target method, force replacement mode.");  
				isInlineHook = false;  
			}  
		}  
	} else {  
		isInlineHook = false;  
	}  
}  
```


###  hookReplace0

```java
hookRecord.bridge = replacement;  
hookRecord.skipUpdateDeclaringClass = true;  
// 4. 调用hookReplace执行hook  
backup = hookReplace0(thread, declaring, hookRecord, method, replacement, backup,  
		isInlineHook, jni, proxy);  

if (backup == null)  
	throw new RuntimeException("Failed to hook method " + method);  

backup.setAccessible(true);  
return hookRecord.backup = backup;  
```

核心是在JNI方法中
```java
// threadAddress - 当前线程的ArtMethod对象
// declaring - hookee方法对应的java类
// hookRecord - HookRecord方法
// javaTarget - Hookee方法对象
// javaReplacement - Hooker方法对象
// javaBackup - 用于回调的原始方法
// isInlineHook - 是否需要进行inline hook
// isJni - 是否是JNI类型的方法
// isProxy - 是否是代理方法
jobject Pine_hookReplace(JNIEnv* env, jclass, jlong threadAddress, jclass declaring, jobject hookRecord,  
                         jobject javaTarget, jobject javaReplacement, jobject javaBackup,  
                         jboolean isInlineHook, jboolean isJni, jboolean isProxy) {  
    // 参数初始化
    jfieldID HookRecord_trampoline = GetHookRecordTrampolineField(env, hookRecord);  
    auto thread = reinterpret_cast<art::Thread*>(threadAddress);  
    auto target = art::ArtMethod::FromReflectedMethod(env, javaTarget);  
    auto replacement = art::ArtMethod::FromReflectedMethod(env, javaReplacement);  
    auto backup = art::ArtMethod::FromReflectedMethod(env, javaBackup);  
  
    bool is_inline_hook = JBOOL_TRUE(isInlineHook);  
    const bool is_native = JBOOL_TRUE(isJni);  
    const bool is_proxy = JBOOL_TRUE(isProxy);  
    const bool is_native_or_proxy = is_native || is_proxy;  
  
    TrampolineInstaller* trampoline_installer = TrampolineInstaller::GetDefault();  
	// 判断是否需要采用inline hook
    if (is_inline_hook && (trampoline_installer->IsReplacementOnly() || !target->IsCompiled())) {  
        is_inline_hook = false;  
    }  
  
    if (UNLIKELY(is_inline_hook && trampoline_installer->CannotSafeInlineHook(target))) {  
        LOGW("Cannot safe inline hook the target method, force replacement mode.");  
        is_inline_hook = false;  
    }  
	// 是否需要跳过检测
    bool skip_first_few_bytes = PineConfig::anti_checks && is_inline_hook  
            && trampoline_installer->CanSkipFirstFewBytes(target);  
    // 开始执行hook 
    void* new_entrypoint;  
    char error_msg[288];  
    {  
        // ArtMethod objects are very important. Many threads depend on their values,  
        // so we need to suspend other threads to avoid errors.        ScopedSuspendVM suspend_vm(thread);  
  
        void* call_origin = is_inline_hook  
                            ? trampoline_installer->InstallDirectJumpInlineTrampoline(target, replacement, skip_first_few_bytes)  
                            : trampoline_installer->InstallDirectJumpReplacementTrampoline(target, replacement);  
  
        if (LIKELY(call_origin)) {  
            backup->BackupFrom(target, call_origin, is_inline_hook, is_native, is_proxy);  
            target->AfterHook(is_inline_hook, is_native_or_proxy);  
            new_entrypoint = target->GetEntryPointFromCompiledCode();  
        } else {  
            snprintf(error_msg, sizeof(error_msg), "Failed to install %s trampoline on method %p: %s (%d).",  
                     is_inline_hook ? "inline" : "replacement", target, strerror(errno), errno);  
            if (errno == EACCES || errno == EPERM)  
                strlcat(error_msg, " This is a security failure, check selinux policy, seccomp or capabilities. Earlier log may point out root cause.", sizeof(error_msg));  
            LOGE("%s", error_msg);  
            new_entrypoint = nullptr;  
        }  
    }  
    
    // 返回用于调用原始方法的方法指针(其实也就是java.lang.reflect.Method)
    if (LIKELY(new_entrypoint)) {  
        env->SetLongField(hookRecord, HookRecord_trampoline, reinterpret_cast<jlong>(new_entrypoint));  
        return env->ToReflectedMethod(declaring, backup->ToMethodID(),  
                                      static_cast<jboolean>(backup->IsStatic()));  
    } else {  
        JNIHelper::Throw(env, errno == EACCES || errno == EPERM ? "java/lang/SecurityException" : "java/lang/RuntimeException", error_msg);  
        return nullptr;  
    }  
}
```


#### InstallDirectJumpInlineTrampoline 


可以发现这个方法和InstallInlineTrampoline的方法相似度非常高，他们的区别在于二级跳板。
InstallInlineTrampoline的二级跳板是CreateBridgeJumpTrampoline方法创建的bridge jump trampoline然而
InstallDirectJumpInlineTrampoline的二级跳板是CreateMethodJumpTrampoline方法创建的method jump trampoline
```java
void* TrampolineInstaller::InstallDirectJumpInlineTrampoline(art::ArtMethod* target,  
                                                             art::ArtMethod* replacement,  
                                                             bool skip_first_few_bytes) {  
                                                             
    // 关闭内存保护
    void* target_code_addr = target->GetCompiledCodeAddr();  
    bool target_code_writable = Memory::Unprotect(target_code_addr);  
    if (UNLIKELY(!target_code_writable)) {  
        LOGE("Failed to make target code writable!");  
        return nullptr;  
    }  
	
    size_t backup_size = kDirectJumpTrampolineSize;  
    if (skip_first_few_bytes) backup_size += kSkipBytes;  
    // 创建backup方法
    void* backup = Backup(target, backup_size);  
    if (UNLIKELY(!backup)) return nullptr;  
	// 创建method jump trampoline
    void* method_jump_trampoline = CreateMethodJumpTrampoline(replacement);  
    if (UNLIKELY(!method_jump_trampoline)) return nullptr;  
  
    {   // 内存保护
        ScopedMemoryAccessProtection protection(target_code_addr, kDirectJumpTrampolineSize);  
        if (skip_first_few_bytes) { // 需要考虑绕过检测
            // 先填充nop指令
            FillWithNopImpl(target_code_addr, kSkipBytes);
            // 然后填充jump trampoline  
            WriteDirectJumpTrampolineTo(AS_VOID_PTR(AS_PTR_NUM(target_code_addr) + kSkipBytes),  
                                        method_jump_trampoline);  
        } else { // 不需要考虑绕过检测  
			// 直接填充jump trampoline
            WriteDirectJumpTrampolineTo(target_code_addr, method_jump_trampoline);  
        }  
    }  
  
    if (PineConfig::debug)  
        LOGD("InstallInlineTrampoline: target_code_addr %p backup %p jump_trampoline %p",  
             target_code_addr, backup, method_jump_trampoline);  
  
    return backup;  
}
```


method jump trampoline
修改x0 & x17并执行代码跳转操作，整体逻辑感觉比起bridge jump trampoline粗糙太多了。
```c++
FUNCTION(pine_method_jump_trampoline)  
LDVAR(x0, pine_method_jump_trampoline_dest_method)  
LDVAR(x17, pine_method_jump_trampoline_dest_entry)  
br x17  
VAR(pine_method_jump_trampoline_dest_method)  
.long 0  
.long 0  
VAR(pine_method_jump_trampoline_dest_entry)  
.long 0  
.long 0
```

#### InstallDirectJumpReplacementTrampoline


逻辑同InstallReplacementTrampoline，只是trampoline换成了CreateMethodJumpTrampoline
```java
void*  
TrampolineInstaller::InstallDirectJumpReplacementTrampoline(art::ArtMethod* target, art::ArtMethod* replacement) {  
    void* origin_code_entry = target->GetEntryPointFromCompiledCode();  
    void* trampoline = CreateMethodJumpTrampoline(replacement);  
    if (UNLIKELY(!trampoline)) return nullptr;  
    target->SetEntryPointFromCompiledCode(trampoline);  
  
    if (PineConfig::debug)  
        LOGD("InstallDirectJumpReplacementTrampoline: origin %p origin_entry %p jump_to %p",  
             target, origin_code_entry, trampoline);  
  
    return origin_code_entry;  
}
```


# 方法执行桥接分析

注本文主要分析Arm64平台下的桥接逻辑


## 桥接方法入口

在分析方法桥接之前，我们需要知晓桥接方法是什么
```java
// 桥接方法获取
hookRecord.bridge = PineConfig.sdkLevel == Build.VERSION_CODES.M && arch == ARCH_ARM64  
        ? Arm64MarshmallowEntry.getBridge(bridgeMethodName, hookRecord.paramNumber)  
        : sBridgeMethods.get(bridgeMethodName);
        
// 根据返回值 & 字段数目获取桥接方法。
public static Method getBridge(String bridgeName, int paramNumber) {  
	// 限定参数不能像小于2 
    if (paramNumber < 2) paramNumber = 2;  
    Class<?>[] bridgeParamTypes = new Class<?>[paramNumber];  
    // 填充params所有的字段
    for (int i = 0;i < paramNumber;i++) {  
        bridgeParamTypes[i] = long.class;  
    } 
    // 通过反射获取特定的方法 
    try {  
        Method bridge = Arm64MarshmallowEntry.class.getDeclaredMethod(bridgeName, bridgeParamTypes);  
        bridge.setAccessible(true);  
        return bridge;  
    } catch (NoSuchMethodException e) {  
        throw new IllegalArgumentException(e);  
    }  
}
```


假如方法返回类型是void,那么他对应的桥接方法有如下可能(基于方法参数个数)
```java

// 方法参数为0,1,2
static void voidBridge(long artMethod, long extras) throws Throwable {  
    voidBridge(artMethod, extras, 0);  
}  
  
// 方法参数为3
static void voidBridge(long artMethod, long extras, long sp) throws Throwable {  
    voidBridge(artMethod, extras, sp, 0);  
}  
  
// 方法参数为4 
static void voidBridge(long artMethod, long extras, long sp,  
                       long x4) throws Throwable {  
    voidBridge(artMethod, extras, sp, x4, 0);  
}  

// 方法参数为5
static void voidBridge(long artMethod, long extras, long sp,  
                       long x4, long x5) throws Throwable {  
    voidBridge(artMethod, extras, sp, x4, x5, 0);  
}  
// 方法参数为6  
static void voidBridge(long artMethod, long extras, long sp,  
                       long x4, long x5, long x6) throws Throwable {  
    voidBridge(artMethod, extras, sp, x4, x5, x6, 0);  
}  
// 方法参数为7  
static void voidBridge(long artMethod, long extras, long sp,  
                       long x4, long x5, long x6, long x7) throws Throwable {  
    Arm64Entry.voidBridge(artMethod, extras, sp, x4, x5, x6, x7);  
}
```


## Arm64Entry


### handleBridge


该方法的主要逻辑可以分为两大步骤，其实不难发现，主要是做的方法的拼接逻辑，核hookee方法的分发执行在Pine.handleCall中。
1.参数组装
	a.克隆extra参数
	b.解析寄存器 & 堆栈参数
	c.解析this参数
	d.将参数转为object数组(方便后续反射调用)
2.处理方法调用
	即调用Pine.handleCall并将#1处的所有参数传入，对hookee方法进行分发执行。

接下来会逐一讲解上述流程。

```java
public final class Arm64Entry {

// ......

// 方案执行的第一步
static void voidBridge(long artMethod, long extras, long sp,  
                               long x4, long x5, long x6, long x7) throws Throwable {  
    handleBridge(artMethod, extras, sp, x4, x5, x6, x7);  
}

private static Object handleBridge(long artMethod, long originExtras, long sp,
								   long x4, long x5, long x6, long x7) throws Throwable {
	// #1 克隆extra参数
	// Clone the extras and unlock to minimize the time we hold the lock
	long extras = Pine.cloneExtras(originExtras);
	Pine.log("handleBridge: artMethod=%#x originExtras=%#x extras=%#x sp=%#x", artMethod, originExtras, extras, sp);
	// #2 解析寄存器 & 堆栈参数
	Pine.HookRecord hookRecord = Pine.getHookRecord(artMethod);
	ThreeTuple<long[], long[], double[]> threeTuple = getArgs(hookRecord, extras, sp, x4, x5, x6, x7);
	long[] coreRegisters = threeTuple.a;
	long[] stack = threeTuple.b;
	double[] fpRegisters = threeTuple.c;
	
	Object receiver;
	Object[] args;

	int crIndex = 0, stackIndex = 0, fprIndex = 0;
	long thread = Pine.currentArtThread0();
    // #3 解析this参数
	if (hookRecord.isStatic) {
		receiver = null;
	} else {
		receiver = Pine.getObject(thread, coreRegisters[0]);
		crIndex = 1;
		stackIndex = 1;
	}
	// #4 将参数转为object数组(方便后续反射调用)
	if (hookRecord.paramNumber > 0) {
		args = new Object[hookRecord.paramNumber];
		for (int i = 0; i < hookRecord.paramNumber; i++) {
			Class<?> paramType = hookRecord.paramTypes[i];
			Object value;
			if (paramType == double.class) {
				if (fprIndex < fpRegisters.length)
					value = fpRegisters[fprIndex++];
				else
					value = Double.longBitsToDouble(stack[stackIndex]);
			} else if (paramType == float.class) {
				long asLong;
				if (fprIndex < fpRegisters.length)
					asLong = Double.doubleToLongBits(fpRegisters[fprIndex++]);
				else
					asLong = stack[stackIndex];
				value = Float.intBitsToFloat((int) (asLong & INT_BITS));
			} else {
				long asLong;
				if (crIndex < coreRegisters.length)
					asLong = coreRegisters[crIndex++];
				else
					asLong = stack[stackIndex];

				if (paramType.isPrimitive()) {
					if (paramType == int.class) {
						value = (int) (asLong & INT_BITS);
					} else if (paramType == long.class) {
						value = asLong;
					} else if (paramType == boolean.class) {
						value = (asLong & INT_BITS) != 0;
					} else if (paramType == short.class) {
						value = (short) (asLong & SHORT_BITS);
					} else if (paramType == char.class) {
						value = (char) (asLong & SHORT_BITS);
					} else if (paramType == byte.class) {
						value = (byte) (asLong & BYTE_BITS);
					} else {
						throw new AssertionError("Unknown primitive type: " + paramType);
					}
				} else {
					// In art, object address is actually 32 bits
					value = Pine.getObject(thread, asLong & INT_BITS);
				}
			}
			args[i] = value;
			stackIndex++;
		}
	} else {
		args = Pine.EMPTY_OBJECT_ARRAY;
	}
	// #5 调用handleCall执行
	return Pine.handleCall(hookRecord, receiver, args);
}

// ......

}
```


- 克隆extra参数
```java
long extras = Pine.cloneExtras(originExtras);
```
实际实现逻辑是通过JNI实现的
```c++
jlong Pine_cloneExtras(JNIEnv*, jclass, jlong extras) {  
    return reinterpret_cast<jlong>(reinterpret_cast<Extras*>(extras)->CloneAndUnlock());  
}

Extras* CloneAndUnlock() {  
	// malloc分配新的内存
    Extras* cloned = static_cast<Extras*>(malloc(sizeof(Extras)));  
    // 拷贝内存
    memcpy(cloned, this, sizeof(Extras));  
    // 释放锁
    ReleaseLock();  
    return cloned;  
}

// 释放锁
void ReleaseLock() {  
#if defined(__aarch64__) || defined(__arm__) // Not supported spinlock on x86 platform  
		CHECK(lock_flag == 0, "Unexpected lock_flag %d", lock_flag);  
		dmb(); // Ensure all previous accesses are observed before the lock is released.  
		lock_flag = 1;  
		dsb(); // Ensure completion of the store that cleared the lock before sending the event.  
		sev(); // Wake up the thread that is waiting for the lock.  
#endif  
	}

```

Note：这一步可能你会觉得很不合理，主要有两个问题，一为什么需要拷贝，二为什么需要释放锁。
其一拷贝是为了防止线程竞争，即不对公共的字段做修改。
其二释放锁是由于jump trampoline为了防止多线程竞争会加锁，在获取到锁以后才会执行到这里(具体逻辑可以看前面跳板分析的逻辑)，因此为了防止死锁，需要释放锁。（此处的锁应该就是为了方式extra被修改的）
- 解析寄存器 & 堆栈参数
```java
// 通过artMethod对象获取HookRecord
Pine.HookRecord hookRecord = Pine.getHookRecord(artMethod);  
// 解析hook record中的记录
ThreeTuple<long[], long[], double[]> threeTuple = getArgs(hookRecord, extras, sp, x4, x5, x6, x7);  
long[] coreRegisters = threeTuple.a;  
long[] stack = threeTuple.b;  
double[] fpRegisters = threeTuple.c;
```

```java
private static ThreeTuple<long[], long[], double[]> getArgs(Pine.HookRecord hookRecord, long extras, long sp,  
                                                            long x4, long x5, long x6, long x7) {  
    int crLength = 0;  
    int stackLength = 0;  
    int fprLength = 0;  
    boolean[] typeWides;  
  
    if (hookRecord.paramTypesCache == null) {  
        int paramTotal = hookRecord.paramNumber;  
        if (!hookRecord.isStatic) {  
            crLength = 1;  
            stackLength = 1;  
            paramTotal++;  
        }  
        if (paramTotal != 0) {  
            typeWides = new boolean[paramTotal];  
            if (!hookRecord.isStatic) {  
                typeWides[0] = false; // "this" object is a reference which is always 32-bit  
            }  
            for (int i = 0;i < hookRecord.paramNumber;i++) {  
                Class<?> paramType = hookRecord.paramTypes[i];  
                boolean fp;  
                boolean wide;  
                if (paramType == double.class) {  
                    fp = true;  
                    wide = true;  
                } else if (paramType == float.class) {  
                    fp = true;  
                    wide = false;  
                } else if (paramType == long.class) {  
                    fp = false;  
                    wide = true;  
                } else {  
                    fp = false;  
                    wide = false;  
                }  
  
                if (fp) { // floating point  
                    if (fprLength < FPR_SIZE)  
                        fprLength++;  
                } else {  
                    if (crLength < CR_SIZE)  
                        crLength++;  
                }  
                stackLength += wide ? 8 : 4;  
  
                if (hookRecord.isStatic)  
                    typeWides[i] = wide;  
                else  
                    typeWides[i + 1] = wide;  
            }  
        } else {  
            typeWides = EMPTY_BOOLEAN_ARRAY;  
        }  
  
        // Expose paramTypesCache after cache initialized to prevent possible race conditions  
        ParamTypesCache cache = new ParamTypesCache();  
        cache.crLength = crLength;  
        cache.stackLength = stackLength;  
        cache.fprLength = fprLength;  
        cache.typeWides = typeWides.clone();  
        hookRecord.paramTypesCache = cache;  
    } else {  
        ParamTypesCache cache = (ParamTypesCache) hookRecord.paramTypesCache;  
        crLength = cache.crLength;  
        stackLength = cache.stackLength;  
        fprLength = cache.fprLength;  
  
        // Do not use original typeWides array as it may still be used by other threads  
        typeWides = cache.typeWides.clone();  
    }  
  
    // This can happen when we are running on Android 6.0. Avoid reading any value from stack  
    // to avoid segmentation faults. This is safe because this only happens when the target    // method have few parameters, in which case we can get all arguments from core registers    
    if (sp == 0)  
        stackLength = 0;  
  
    long[] coreRegisters = crLength != 0 ? new long[crLength] : EMPTY_LONG_ARRAY;  
    long[] stack = stackLength != 0 ? new long[stackLength] : EMPTY_LONG_ARRAY;  
    double[] fpRegisters = fprLength != 0 ? new double[fprLength] : EMPTY_DOUBLE_ARRAY;  
    // 从extras中解析x1~x3 & d0 ~ d7寄存器 & 堆栈中的参数, 并设置到typeWides、coreRegisters、fpRegisters、stack中去。
    // typeWides 每一个参数的类型
    // coreRegisters 通用寄存器x1 ~ x7 保存了7个通用类型的参数
    // fpRegisters 浮点寄存器d0 ~ d7 可以用于保存8个浮点参数
    // stack 多于的参数通过堆栈传参。
    Pine.getArgsArm64(extras, sp, typeWides, coreRegisters, stack, fpRegisters);  
  
    do {  
        // x1-x3 are restored in Pine.getArgs64  
        // 将x4 ~ x7 写入coreRegisters
        if (crLength < 4) break;  
        coreRegisters[3] = x4;  
        if (crLength == 4) break;  
        coreRegisters[4] = x5;  
        if (crLength == 5) break;  
        coreRegisters[5] = x6;  
        if (crLength == 6) break;  
        coreRegisters[6] = x7;  
    } while(false);  
	// 解析coreRegisters(通用寄存器)，stack(堆栈)，fpRegisters(浮点寄存器)
    return new ThreeTuple<>(coreRegisters, stack, fpRegisters);  
}
```

- 解析this参数
```java
// static方法没有this参数
if (hookRecord.isStatic) {  
    receiver = null;  
} else {  
	// 非static方法需要将一个参数转为java Object类型。(目前传参的时候只有java类型)
    receiver = Pine.getObject(thread, coreRegisters[0]);  
    crIndex = 1;  
    stackIndex = 1;  
}
```
而将一个C++指针转为jobject对象可以通过AddLocalRef方法实现
```java
jobject Pine_getObject0(JNIEnv* env, jclass, jlong thread, jlong address) {  
    return reinterpret_cast<art::Thread*>(thread)->AddLocalRef(env, reinterpret_cast<Object*>(address));  
}

// 具体实现有如下几个步骤
// 1. 通过调用art::JavaVMExt::AddWeakGlobalRef创建一个WeakGlobalRef
// 2. 创建localRef
// 3. 释放WeakGlobalRef
// 4. 返回localRef
jobject AddLocalRef(JNIEnv* env, Object* obj) {
	if (UNLIKELY(obj->IsForwardingAddress())) {
		// Bug #3: Invalid state during hashcode ForwardingAddress
		// Caused by gc moved the object?
		// The object has moved to new address, forwarding to it.
		Object* forwarding = obj->GetForwardingAddress();
		LOGW("Detected forwarding address object (origin %p, monitor %u, forwarding to %p)",
				obj, obj->GetMonitor(), forwarding);
		CHECK(forwarding != nullptr, "Forwarding to nullptr");
		// FIXME: Will this check fail under normal circumstances?
		CHECK_EQ(obj->GetClass(), forwarding->GetClass(),
				"Forwarding object type mismatch (origin %p, forwarding %p)", obj->GetClass(), forwarding->GetClass());
		obj = forwarding;
	}
	if (LIKELY(new_local_ref)) {
		return new_local_ref(env, obj);
	}
	jweak global_weak_ref = add_weak_global_ref(Android::jvm_, this, obj);
	jobject local_ref = env->NewLocalRef(global_weak_ref);
	env->DeleteWeakGlobalRef(global_weak_ref);
	return local_ref;
}
```
- 将参数转为object数组(方便后续反射调用)
```java
// 就是一个大的for循环，将以前几步解析出来的long数组转化为实际的java对象类型。

if (hookRecord.paramNumber > 0) {
	args = new Object[hookRecord.paramNumber];
	for (int i = 0; i < hookRecord.paramNumber; i++) {
		Class<?> paramType = hookRecord.paramTypes[i];
		Object value;
		if (paramType == double.class) {
			if (fprIndex < fpRegisters.length)
				value = fpRegisters[fprIndex++];
			else
				value = Double.longBitsToDouble(stack[stackIndex]);
		} else if (paramType == float.class) {
			long asLong;
			if (fprIndex < fpRegisters.length)
				asLong = Double.doubleToLongBits(fpRegisters[fprIndex++]);
			else
				asLong = stack[stackIndex];
			value = Float.intBitsToFloat((int) (asLong & INT_BITS));
		} else {
			long asLong;
			if (crIndex < coreRegisters.length)
				asLong = coreRegisters[crIndex++];
			else
				asLong = stack[stackIndex];

			if (paramType.isPrimitive()) {
				if (paramType == int.class) {
					value = (int) (asLong & INT_BITS);
				} else if (paramType == long.class) {
					value = asLong;
				} else if (paramType == boolean.class) {
					value = (asLong & INT_BITS) != 0;
				} else if (paramType == short.class) {
					value = (short) (asLong & SHORT_BITS);
				} else if (paramType == char.class) {
					value = (char) (asLong & SHORT_BITS);
				} else if (paramType == byte.class) {
					value = (byte) (asLong & BYTE_BITS);
				} else {
					throw new AssertionError("Unknown primitive type: " + paramType);
				}
			} else {
				// In art, object address is actually 32 bits
				value = Pine.getObject(thread, asLong & INT_BITS);
			}
		}
		args[i] = value;
		stackIndex++;
	}
} else {
	args = Pine.EMPTY_OBJECT_ARRAY;
}
```

### Pine.handleCall

方法逻辑如下：
1.处理原始方法调用前回调
2.原始方法调用
3.处理原始方法调用后回调
可以发现和epic其实是很类似的。
```java
public static Object handleCall(HookRecord hookRecord, Object thisObject, Object[] args)  
        throws Throwable {  
    // WARNING: DO NOT print thisObject or args, else the toString() method will be called on it  
    // At this time the object may not "ready"    if (PineConfig.debug)  
        /*Log.d(TAG, "handleCall: target=" + hookRecord.target + " thisObject=" +  
                thisObject + " args=" + Arrays.toString(args));*/        Log.d(TAG, "handleCall for method " + hookRecord.target);  
  
    if (PineConfig.disableHooks || hookRecord.emptyCallbacks()) {  
        try {  
            return callBackupMethod(hookRecord, thisObject, args);  
        } catch (InvocationTargetException e) {  
            throw e.getTargetException();  
        }  
    }  
	// 获取用于共享的context,后续回调需要传入  
    CallFrame callFrame = new CallFrame(hookRecord, thisObject, args);  
    // 获取callbacks后续需要逐一回调
    MethodHook[] callbacks = hookRecord.getCallbacks();  
  
    // call before callbacks  
    // #1 处理方法执行前回调
    int beforeIdx = 0;  
    do {  
        MethodHook callback = callbacks[beforeIdx];  
        try {  
            callback.beforeCall(callFrame);  
        } catch (Throwable e) {  
            Log.e(TAG, "Unexpected exception occurred when calling " + callback.getClass().getName() + ".beforeCall()", e);  
            // reset result (ignoring what the unexpectedly exiting callback did)  
            callFrame.resetResult();  
            continue;  
        }  
        // 如果hook方法被拦截，记录停止执行的callback index,直接break，
        if (callFrame.returnEarly) {  
            // skip remaining "before" callbacks and corresponding "after" callbacks  
            beforeIdx++;  
            break;  
        }  
    } while (++beforeIdx < callbacks.length);  
  
    // call original method if not requested otherwise 
    // #2 调用原始方法(在方法没有被拦截的情况下) 
    if (!callFrame.returnEarly) {  
        try {  
            callFrame.setResult(callFrame.invokeOriginalMethod());  
        } catch (InvocationTargetException e) {  
            callFrame.setThrowable(e.getTargetException());  
        }  
    }  
  
    // call after callbacks  
    // #3 处理原始方法执行完成后
    int afterIdx = beforeIdx - 1;  
    do {  
        MethodHook callback = callbacks[afterIdx];  
        Object lastResult = callFrame.getResult();  
        Throwable lastThrowable = callFrame.getThrowable();  
        try {  
            callback.afterCall(callFrame);  
        } catch (Throwable e) {  
            Log.e(TAG, "Unexpected exception occurred when calling " + callback.getClass().getName() + ".afterCall()", e);  
  
            // reset to last result (ignoring what the unexpectedly exiting callback did)  
            if (lastThrowable == null)  
                callFrame.setResult(lastResult);  
            else  
                callFrame.setThrowable(lastThrowable);  
        }  
    } while (--afterIdx >= 0);  // 确保成对调用，只有调用过before的方法才会执行after
  
    // return  
    // 最后返回代码的执行结果
    if (callFrame.hasThrowable())  
        throw callFrame.getThrowable();  
    else  
        return callFrame.getResult();  
}
```

上述过程中#1,#3都是一个简单的for循环调用callback,唯一需要分析的就是原始方法的调用逻辑了。
```java
// CallFrame
public Object invokeOriginalMethod() throws InvocationTargetException, IllegalAccessException {  
    return callBackupMethod(hookRecord, thisObject, args);  
}

static Object callBackupMethod(HookRecord hookRecord, Object thisObject, Object[] args) throws InvocationTargetException, IllegalAccessException {  
    // java.lang.Class object is movable and may cause crash when invoke backup method,  
    // native entry of JNI method may be changed by RegisterNatives and UnregisterNatives,    
    // so we need to update them when invoke backup method.    Member origin = hookRecord.target;  
    Method backup = hookRecord.backup;  
    Class<?> declaring = origin.getDeclaringClass();  
    // 在调用方法前需要更新下back method的一些信息
    syncMethodInfo(origin, backup, hookRecord.skipUpdateDeclaringClass);  
    // 直接反射调用
    // FIXME: GC happens here (you can add Runtime.getRuntime().gc() to test) will crash backup calling  
    Object result = backup.invoke(thisObject, args);  
    // Explicit use declaring_class object to ensure it has reference on stack  
    // and avoid being moved by gc. (invalid for now)    declaring.getClass();  
    return result;  
}
```

同步更新逻辑如下，就是把原始方法和backup方法的declaringClass取出来对比如果不一致就执行更新，除了declaringClass还会对EntryPointFromJni做类似的对比操作。
(可能是担心gc or 其他原因更新了原始方法？)
```c++
void Pine_syncMethodInfo(JNIEnv* env, jclass, jobject javaOrigin, jobject javaBackup, jboolean skipDeclaringClass) {  
    auto origin = art::ArtMethod::FromReflectedMethod(env, javaOrigin);  
    auto backup = art::ArtMethod::FromReflectedMethod(env, javaBackup);  
  
    // An ArtMethod is actually an instance of java class "java.lang.reflect.ArtMethod" on pre M  
    // declaring_class is a reference field so the runtime itself will update it if moved by GC    if (skipDeclaringClass == JNI_FALSE && Android::version >= Android::kM) {  
        uint32_t declaring_class = origin->GetDeclaringClass();  
        if (declaring_class != backup->GetDeclaringClass()) {  
            LOGI("GC moved declaring class of method %p, also update in backup %p", origin, backup);  
            backup->SetDeclaringClass(declaring_class);  
        }  
    }  
  
    // JNI method entry might be changed by RegisterNatives or UnregisterNatives  
    // Use backup to check native as we may add kNative to access flags of origin (Android 8.0+ with debuggable mode)    if (backup->IsNative()) {  
        void* previous = backup->GetEntryPointFromJni();  
        void* current = origin->GetEntryPointFromJni();  
        if (current != previous) {  
            LOGI("Native entry of method %p was changed, also update in backup %p", origin, backup);  
            backup->SetEntryPointFromJni(current);  
        }  
    }  
}
```


# 其他


## PendingHook

Note：这一逻辑感觉、可能、maybe正处理开发中，

static方法是懒加载的，如果方法没有初始化好那么是无法执行hook的，因此hook static方法前需要确保方法已经初始化好了，最简单的方法就是手动调用一次。
但是除了手动调用以外还有一种稍微麻烦一点的方式可以兼容适配这种情况——等初始化好了再执行hook


#### enableDelayHook

但是“等待初始化完成了再执行hook“这个能力并不是默认就启用的，需要调用api才能开启，api如下：

```java
PineEnhances.enableDelayHook()
```

点进去看下实现原理,发现非常简单就是一个字段设置为了true
```java
public static boolean enableDelayHook() {  
	// 确保so被加载了 
    ensureInited();  
    if (!PendingHookHandler.canWork()) {  
        Log.e(TAG, "PendingHookHandler not working");  
        return false;  
    }  
    // PendingHookHandler.install会将默认的handler设置为PendingHookHandler
    // setEnabled就只是简单的设置一个变量
    PendingHookHandler.install().setEnabled(true);  
    return true;  
}
```

然后我们看看PendingHookHandler的hook处理方法——handleHook方法
```java
public class PendingHookHandler implements Pine.HookHandler, ClassInitMonitor.Callback {

	// ......
	public MethodHook.Unhook handleHook(Pine.HookRecord hookRecord, MethodHook hook, int modifiers,  
	                                    boolean newMethod, boolean canInitDeclaringClass) {  
	                                    
	    // 判断是否启用延后执行（核心点就是看类是否已经初始化完全）
	    boolean skipInit = hook != null && shouldDelay(hookRecord.target, newMethod, modifiers);  
	    // 将hook保存到JNI中
	    if (newMethod) recordMethodHooked(hookRecord.artMethod, PREVENT_ENTRY_UPDATE, PREVENT_ENTRY_UPDATE);  
	    
	    // replaceMent Mode
	    if (Pine.getHookMode() == Pine.HookMode.REPLACEMENT) {  
	        // Here we always need to record hooked methods even if they don't need to be delayed  
	        // because we manually have shut the debug switch down, we need to skip ShouldUseInterpreterEntrypoint        
	        // WARNING: Do not log the target method here, as it may trigger        
	        // initialization of parameters and return type   
	        // 如果是skipInit为true此次不会执行hook     
	        MethodHook.Unhook u = realHandler.handleHook(hookRecord, hook, modifiers, newMethod,  
	                !skipInit && canInitDeclaringClass);  
	        // 更新下记录
	        if (newMethod) recordMethodHooked(hookRecord.artMethod, hookRecord.trampoline,  
	                Pine.getArtMethod(hookRecord.backup));  
	        // 返回        
	        return u;  
	    }  
	  
	    // 其他模式(INLINE Mode)
	    
	    // 处理skipInit的case
	    if (skipInit) {  
		    // 添加一条hook记录
	        Class<?> declaring = hookRecord.target.getDeclaringClass();  
	        synchronized (pendingMap) {  
	            Set<Pine.HookRecord> pendingHooks = pendingMap.get(declaring);  
	            if (pendingHooks == null) {  
	                pendingHooks = new HashSet<>(1, 1f);  
	                pendingMap.put(declaring, pendingHooks);  
	                // 调用ClassInitMonitor监控类完成初始化完成
	                ClassInitMonitor.care(declaring);  
	            }  
	            pendingHooks.add(hookRecord);  
	        }  
	        hookRecord.addCallback(hook);  
	        return hook.new Unhook(hookRecord);  
	    }  
	    // not skip模式下先直接进行初始化。
	    MethodHook.Unhook u = realHandler.handleHook(hookRecord, hook, modifiers, newMethod, canInitDeclaringClass);  
	    if (newMethod) recordMethodHooked(hookRecord.artMethod, hookRecord.trampoline,  
	            Pine.getArtMethod(hookRecord.backup));  
	    return u;  
	}
	// ......

}
```


这里有一点很有意思，inline mode的情况下类没有完全初始化好，是没法执行hook的，因此需要延后执行realHandler.handleHook，延后到什么时候执行呢？很容易理解那就是类初始化完全，怎么实现的呢？答案就在ClassInitMonitor中。
#### ClassInitMonitor

既然是等待初始化，我怎么知道什么时候是初始化好了呢？因此这时候我们就需要监控类的初始化并进行回调。
怎么实现类初始化的监控呢？答案就在ClassInitMonitor中

```java
public class ClassInitMonitor {

	static {
        try {
	        // 确保Pine初始化好。 
            Pine.ensureInitialized();
            // 初始化ClassInitMonitor
            canWork = PineEnhances.initClassInitMonitor(PineConfig.sdkLevel, Pine.openElf,
                    Pine.findElfSymbol, Pine.closeElf, Pine.getMethodDeclaringClass,
                    Pine.syncMethodEntry, Pine.suspendVM, Pine.resumeVM);
        } catch (Throwable e) {
            PineEnhances.logE("Error in initClassInitMonitor", e);
        }
    }


}
```


##### initClassInitMonitor

```c++
PineEnhances.initClassInitMonitor
```


PineEnhances_initClassInitMonitor内部会做很多的方法hook，并且由于需要兼容不同版本的原因，有很多分叉。
- sdk >= 24
art::ClassLinker::ShouldUseInterpreterEntrypoint(art::ArtMethod*, void const*)
如果失败 -> art::instrumentation::Instrumentation::UpdateMethodsCodeImpl(art::ArtMethod*, void const*) 
如果失败 -> art::instrumentation::Instrumentation::UpdateMethodsCode(art::ArtMethod*, void const*)
- else 
art::instrumentation::Instrumentation::UpdateMethodsCode(art::ArtMethod*, void const*)

- sdk >= 33
art::instrumentation::Instrumentation::InitializeMethodsCode(art::ArtMethod*, void const*)

- sdk >= 29
art::ClassLinker::MarkClassInitialized(art::Thread*, art::Handle<art::mirror::Class>)
art::ClassLinker::FixupStaticTrampolines(art::Thread*, art::ObjPtr<art::mirror::Class>)



不过主要hook如下几类方法，我们逐一进行讲解
UpdateMethodsCode - 控制方法代码更新逻辑，用于绕过JIT编译或修改方法实现。
MarkClassInitialized - 拦截类初始化完成标记，实现自定义初始化监控。
FixupStaticTrampolines - 修改静态代码修补逻辑，防止类初始化被篡改检测。
```c++
jboolean PineEnhances_initClassInitMonitor(JNIEnv* env, jclass PineEnhances, jint sdk_level,  
                                           jlong openElf, jlong findElfSymbol, jlong closeElf,  
                                           jlong getMethodDeclaringClass, jlong syncMethodEntry,  
                                           jlong suspendVM, jlong resumeVM) {  
     // 获取onClassInit 
     onClassInit_ = env->GetStaticMethodID(PineEnhances, "onClassInit", "(J)V");  
     if (!onClassInit_) {  
         LOGE("Unable to find onClassInit");  
         return JNI_FALSE;  
     }  
     // 将jclass创建global reference
     PineEnhances_ = static_cast<jclass>(env->NewGlobalRef(PineEnhances));  
     if (!PineEnhances_) {  
         LOGE("Unable to make new global ref");  
         return JNI_FALSE;  
     }  
     // 变量初始化
     auto OpenElf = reinterpret_cast<void* (*)(const char*)>(openElf);  
     FindElfSymbol = reinterpret_cast<void* (*)(void*, const char*, bool)>(findElfSymbol);  
     auto CloseElf = reinterpret_cast<void (*)(void*)>(closeElf);  
     GetMethodDeclaringClass = reinterpret_cast<void* (*)(ArtMethod)>(getMethodDeclaringClass);  
     SyncMethodEntry = reinterpret_cast<void (*)(ArtMethod, ArtMethod, const void*)>(syncMethodEntry);  
     auto SuspendVM = reinterpret_cast<void* (*)(JNIEnv*)>(suspendVM);  
     auto ResumeVM = reinterpret_cast<void (*)(void*)>(resumeVM);  
	 // 获取art 核心库名称
     auto vm_library = GetRuntimeLibraryName(env);  
     void* handle = OpenElf(vm_library.data());  
	 // 获取art::mirror::Class::GetClassDef()方法
     GetClassDef = reinterpret_cast<ClassDef (*)(void*)>(FindElfSymbol(handle,  
             "_ZN3art6mirror5Class11GetClassDefEv", true));  
     if (!GetClassDef) {  
         LOGE("Cannot find symbol art::Class::GetClassDef");  
         return JNI_FALSE;  
     }  
	 // 挂起所有线程 
     void* cookie = SuspendVM(env);  
     bool hooked = false;  
     // 宏定义
#define HOOK_FUNC(name) hooked |= HookFunc(name, (void*) replace_##name , (void**) &backup_##name)  
#define HOOK_SYMBOL(name, symbol, required) hooked |= HookSymbol(handle, symbol, (void*) replace_##name , (void**) &backup_##name , required)  
  
     // Before 7.0, it is NeedsInterpreter which is inlined so cannot be hooked  
     // But the logic which forces code to be executed by interpreter is added in Android 8.0     
     // And we have hooked UpdateMethodsCode to avoid entry updating, so it should be safe to skip  
     
	 // hook Entry Point更新回调
	 
     if (sdk_level >= __ANDROID_API_N__) {  
         // art::ClassLinker::ShouldUseInterpreterEntrypoint(art::ArtMethod*, void const*)
         HOOK_SYMBOL(ShouldUseInterpreterEntrypoint, "_ZN3art11ClassLinker30ShouldUseInterpreterEntrypointEPNS_9ArtMethodEPKv", false);  
         if (!hooked) {  
             // Android Tiramisu?  
             // art::interpreter::ShouldStayInSwitchInterpreter(art::ArtMethod*)
             HOOK_SYMBOL(ShouldStayInSwitchInterpreter, "_ZN3art11interpreter29ShouldStayInSwitchInterpreterEPNS_9ArtMethodE", false);  
         }  
         if (!hooked) {  
             LOGW("Failed to hook ShouldUseInterpreterEntrypoint/ShouldStayInSwitchInterpreter. Hook may fail on debuggable builds.");  
         }  
         hooked = false;  
		 // art::instrumentation::Instrumentation::UpdateMethodsCodeImpl(art::ArtMethod*, void const*) 
         HOOK_SYMBOL(UpdateMethodsCodeImpl, "_ZN3art15instrumentation15Instrumentation21UpdateMethodsCodeImplEPNS_9ArtMethodEPKv", false);  
         if (!hooked) {  
             // UpdateMethodsCodeImpl is inlined, try fallback to UpdateMethodsCode  
             // We cannot always hook UpdateMethodsCode, as it may be too small to be overridden when UpdateMethodsCodeImpl not inlined      
             // art::instrumentation::Instrumentation::UpdateMethodsCode(art::ArtMethod*, void const*)       
             HOOK_SYMBOL(UpdateMethodsCode, "_ZN3art15instrumentation15Instrumentation17UpdateMethodsCodeEPNS_9ArtMethodEPKv", true);  
         }  
     }  
     else  
         // art::instrumentation::Instrumentation::UpdateMethodsCode(art::ArtMethod*, void const*)
         HOOK_SYMBOL(UpdateMethodsCode, "_ZN3art15instrumentation15Instrumentation17UpdateMethodsCodeEPNS_9ArtMethodEPKv", true);      
     if (sdk_level >= __ANDROID_API_T__) {
	     // art::instrumentation::Instrumentation::InitializeMethodsCode(art::ArtMethod*, void const*)
         HOOK_SYMBOL(InitializeMethodsCode, "_ZN3art15instrumentation15Instrumentation21InitializeMethodsCodeEPNS_9ArtMethodEPKv", true);  
     }  
  
     if (sdk_level >= __ANDROID_API_Q__) {  
         // Note: kVisiblyInitialized is not implemented in Android Q,  
         // but we found some ROMs "indicates that is Q", but uses R's art (has "visibly initialized" state)         
         // https://github.com/crdroidandroid/android_art/commit/ef76ced9d2856ac988377ad99288a357697c4fa2         
         void* MarkClassInitialized = FindElfSymbol(handle, "_ZN3art11ClassLinker20MarkClassInitializedEPNS_6ThreadENS_6HandleINS_6mirror5ClassEEE", sdk_level >= __ANDROID_API_R__);  
         if (MarkClassInitialized) {  
             HOOK_FUNC(MarkClassInitialized);  
             HOOK_SYMBOL(FixupStaticTrampolinesWithThread, "_ZN3art11ClassLinker22FixupStaticTrampolinesEPNS_6ThreadENS_6ObjPtrINS_6mirror5ClassEEE", false);  
         }  
     }  
  
     HOOK_SYMBOL(FixupStaticTrampolines, "_ZN3art11ClassLinker22FixupStaticTrampolinesENS_6ObjPtrINS_6mirror5ClassEEE", false);  
     if (!hooked) {  
         // Before Android 8.0, it uses raw mirror::Class* without ObjPtr<>  
         HOOK_SYMBOL(FixupStaticTrampolines, "_ZN3art11ClassLinker22FixupStaticTrampolinesEPNS_6mirror5ClassE", true);  
     }  
#undef HOOK_SYMBOL  

	 // 恢复所有线程
     ResumeVM(cookie);
     // 关闭ELF(用于hook)  
     CloseElf(handle);  
	 // 
     if (!hooked) {  
         LOGE("Cannot hook MarkClassInitialized or FixupStaticTrampolines");  
         return JNI_FALSE;  
     }  
    return JNI_TRUE;  
}
```


- 宏
在介绍上述方法hook逻辑之前，我们针对部分宏做下介绍（后面会用到）
```c++

// 1. HOOK_FUNC & HOOK_SYMBOL宏
// 这两宏是用于做hook的，把hook方法做了下包装，方便后续使用。
// 可以发现这两个宏里面包含两个拼接的变量
// replace_##name (Hook方法，用于替换原始方法)
// backup_##name (原始方法的地址, 类型是一个函数指针)
#define HOOK_FUNC(name) hooked |= HookFunc(name, (void*) replace_##name , (void**) &backup_##name)  
#define HOOK_SYMBOL(name, symbol, required) hooked |= HookSymbol(handle, symbol, (void*) replace_##name , (void**) &backup_##name , required)


// 2. HOOK_ENTRY宏
// HOOK_ENTRY会生成上面提到的replace_##name 、backup_##name 接下来我们看下这俩方法是如何生成的
#define HOOK_ENTRY(name, return_type, ...) \ 
static return_type (*backup_##name)(__VA_ARGS__) = nullptr;\  // 定义backup_##name方法，类型为函数指针
return_type replace_##name (__VA_ARGS__) // 定义replace_##name的函数声明，方法体会在使用处声明。


// 3. HOOK_ENTRY宏使用案例
// 我们看个实例
HOOK_ENTRY(UpdateMethodsCode, void, void* thiz, ArtMethod method, const void* quick_code) {  
    if (PreUpdateMethodsCode(thiz, method, quick_code)) return;  
    backup_UpdateMethodsCode(thiz, method, quick_code);  
}

// 4. HOOK_ENTRY宏展开
static void (*backup_UpdateMethodsCode)(void* thiz, ArtMethod method, const void* quick_code) = null;
void replace_UpdateMethodsCode(void* thiz, ArtMethod method, const void* quick_code) {
	 if (PreUpdateMethodsCode(thiz, method, quick_code)) return;  
     backup_UpdateMethodsCode(thiz, method, quick_code);  
}


```
- UpdateMethodsCode
用于拦截方法入口更新。（主要是针对于pending hook的情况下）
```c++
HOOK_ENTRY(UpdateMethodsCode, void, void* thiz, ArtMethod method, const void* quick_code) {  
    // 如果是更新方法
    if (PreUpdateMethodsCode(thiz, method, quick_code)) return;  
    backup_UpdateMethodsCode(thiz, method, quick_code);  
}

bool PreUpdateMethodsCode(void* thiz, ArtMethod& method, const void*& quick_code) {  
    instrumentation_ = thiz;  
    // 如果方法已经被hook了
    if (IsMethodHooked(method)) {  
        auto backup = GetMethodBackup(method);  
        if (backup) {   // 如果有backup 修改method
            // Redirect entry update to backup  
            method = backup;  
        } else {   // 没有backup 保存下入口地址
            ScopedLock lk(pending_entries_mutex_);  
            pending_entries_[method] = quick_code;
            // 返回true  拦截方法入口，防止方法入口被更新。
            return true;  
        }  
    }  
    return false;  
}
```
- MarkClassInitialized
接收到类初始化完成的时机并将其回调到PineEnhances.onClassInit
```c++
HOOK_ENTRY(MarkClassInitialized, void*, void* thiz, void* self, uint32_t* cls_ptr) {  
    void* result = backup_MarkClassInitialized(thiz, self, cls_ptr);  
    if (cls_ptr) MaybeClassInit(reinterpret_cast<void*>(*cls_ptr));  
    return result;  
}

void MaybeClassInit(void* ptr) {  
    if (ptr == nullptr) return;  
    // 调用art::mirror::Class::GetClassDef() 
    auto class_def = GetClassDef(ptr);  
    bool call_monitor;  
    // 确保class def非null
    if (class_def) {  
        {  
            // 将当前类下挂载的所有的HookRecord取出来
            // 并将backup的entryPoint设置为target.quick_compile_code_
            // 同时target设置为hook用的trampoline地址。
            std::shared_lock<std::shared_mutex> lk(hook_records_mutex_);  
            auto i = hook_records_.find(class_def);  
            if (i != hook_records_.end()) {  
                for (const HookRecord& record : i->second) {  
                    SyncMethodEntry(record.target, record.backup, record.entrypoint);  
                }  
            }  
        }  
        // 从care list 中移除当前class_def
        ScopedLock lk(cared_classes_mutex_);  
        call_monitor = cared_classes_.erase(class_def) != 0;  
    } else {  
        call_monitor = care_no_class_def_;  
    }  
    // 上述流程失效则直接返回，不做回调
    if (!call_monitor) return;  
    // 回调PineEnhances.onClassInit方法
    JNIEnv* env = CurrentEnv();  
    env->CallStaticVoidMethod(PineEnhances_, onClassInit_, reinterpret_cast<jlong>(ptr));  
    if (env->ExceptionCheck()) {  
        LOGE("Unexpected exception threw in onClassInit");  
        env->ExceptionClear();  
    }  
}
```
- FixupStaticTrampolines
首先会调用原始方法，然后会类似MarkClassInitialized一样调用MaybeClassInit通知给PineEnhances.onClassInit
Note: 类初始化以后art会尝试通过调用FixupStaticTrampolines来修复static方法的入口
```c++
HOOK_ENTRY(FixupStaticTrampolines, void, void* thiz, void* cls) {  
    backup_FixupStaticTrampolines(thiz, cls);  
    MaybeClassInit(cls);  
}
```


到此我们以及介绍完了ClassInitMonitor的初始化逻辑——通过监听FixupStaticTrampolines/MarkClassInitialized方法的执行，从而实现类加载的监听，最终回调到PineEnhances.onClassInit(onClassInit中会通过执行HookHandler.handleHook触发当前类中所有延后的hook操作)



##### care


```java
public class ClassInitMonitor {

	// ......

	public static void care(Class<?> cls) {
        if (cls == null) throw new NullPointerException("cls == null");
        // 先将其decode为JObject然后转成一个地址再调用JNI方法
        PineEnhances.careClassInit(Primitives.getAddress(cls));
    }
    
    // ......

}
```

简单看一下PineEnhances_careClassInit的JNI逻辑，其实很简单，就是向cared_classes_插入一个列表。
原因是ClassInitMonitor.initClassInitMonitor会监控所有类“Class初始化完成“行为，插入cared_classes_后ClassInitMonitor才知道需要关注并回调哪些类，防止性能损耗。
```c++
void PineEnhances_careClassInit(JNIEnv*, jclass, jlong address) {  
    void* ptr = reinterpret_cast<void*>(address);  
    auto class_def = GetClassDef(ptr);  
    if (class_def == nullptr) {  
        // This class have no class def. That's mostly impossible, these classes (like proxy classes)  
        // should be initialized before. But if it happens...        
        LOGW("Class %p have no class def, this should not happen, please check the root cause", ptr);  
        care_no_class_def_ = true;  
        return;  
    }  
    ScopedLock lk(cared_classes_mutex_);  
    cared_classes_.insert(class_def);  
}
```



到此PendingHook分析完了～





# 最后



膜拜canyie大佬