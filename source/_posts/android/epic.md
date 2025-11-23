---
title: android art hook之epic实现原理
tags:
  - art hook
  - android
cover: 'https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/epic.drawio.png'
date: 2025-11-23 17:28:30
---


# android art hook之epic实现原理

![epic.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/epic.drawio.png)

图：Epic Art Hook实现原理简图



本文主要是对epic在arm64平台下art虚拟机下的hook的实现逻辑进行分析和学习。

再此感谢weishu大佬～

# 分析起点

DexposedBridge#hookMethod是Epic hook 的入口，不过当前方法是一个包装方法，主要是封装了部分API,hook的核心逻辑在Epic.hookMethod中。DexposedBridge#hookMethod方法逻辑如下：
1.前置检查，只支持普通方法 & 构造函数
2.将Method和XC_MethodHook做绑定存入到Map<Member, CopyOnWriteSortedSet<XC_MethodHook>> hookedMethodCallbacks中去。

3.调用Epic.hookMethod开启hook逻辑

```java
// DexposedBridge
public static XC_MethodHook.Unhook hookMethod(Member hookMethod, XC_MethodHook callback) {
		// 1.前置检查，只支持方法类型 
		if (!(hookMethod instanceof Method) && !(hookMethod instanceof Constructor<?>)) {
			throw new IllegalArgumentException("only methods and constructors can be hooked");
		}
		
		// 记录每一个函数的hookee & hooker并将对应关系保存到hookedMethodCallbacks中
		boolean newMethod = false;
		CopyOnWriteSortedSet<XC_MethodHook> callbacks;
		synchronized (hookedMethodCallbacks) {
			callbacks = hookedMethodCallbacks.get(hookMethod);
			if (callbacks == null) {
				callbacks = new CopyOnWriteSortedSet<XC_MethodHook>();
				hookedMethodCallbacks.put(hookMethod, callbacks);
				newMethod = true;
			}
		}

		Logger.w(TAG, "hook: " + hookMethod + ", newMethod ? " + newMethod);

		callbacks.add(callback);
		
		// 由于一个hookee可能对应多个callbacks,当是首次hook的时候触发hook逻辑
		if (newMethod) {
		    // 判断是否是art虚拟机
			if (Runtime.isArt()) { // art虚拟机的hook逻辑
                
                // 3.开启hook逻辑
				if (hookMethod instanceof Method) {
					Epic.hookMethod(((Method) hookMethod));
				} else {
					Epic.hookMethod(((Constructor) hookMethod));
				}
			} else { // 非art虚拟机的hook逻辑 
				Class<?> declaringClass = hookMethod.getDeclaringClass();
				int slot = getIntField(hookMethod, "slot");

				Class<?>[] parameterTypes;
				Class<?> returnType;
				if (hookMethod instanceof Method) {
					parameterTypes = ((Method) hookMethod).getParameterTypes();
					returnType = ((Method) hookMethod).getReturnType();
				} else {
					parameterTypes = ((Constructor<?>) hookMethod).getParameterTypes();
					returnType = null;
				}

				AdditionalHookInfo additionalInfo = new AdditionalHookInfo(callbacks, parameterTypes, returnType);
				hookMethodNative(hookMethod, declaringClass, slot, additionalInfo);
			}
		}
		return callback.new Unhook(hookMethod);
	}
```



# Hook逻辑


```java
// ArtMethod.java
public static boolean hookMethod(Method origin) {
    // 创建artMethod 
	ArtMethod artOrigin = ArtMethod.of(origin);
	// 调用方法执行hook
	return hookMethod(artOrigin);
}
```

## ArtMethod创建

> 此ArtMethod非彼ArtMethod.
> 此处的ArtMethod是epic内部的一个类型。


```java
// ArtMethod.java
public static ArtMethod of(Method method) {
	return new ArtMethod(method, -1);
}

public static ArtMethod of(Method method, long address) {
	return new ArtMethod(method, address);
}

public static ArtMethod of(Constructor constructor) {
	return new ArtMethod(constructor);
}
```

of方法最终都会调用到这两个private的构造，设置constructor/method对象、通过EpicNative获取art底层ArtMethod对象并保存到address字段中
```java
private ArtMethod(Constructor constructor) {
	if (constructor == null) {
		throw new IllegalArgumentException("constructor can not be null");
	}
	this.constructor = constructor;
	init();
}

private ArtMethod(Method method, long address) {  
    if (method == null) {  
        throw new IllegalArgumentException("method can not be null");  
    }  
    this.method = method;  
    if (address != -1) {  
        this.address = address;  
    } else {  
        init();  
    }  
}

private void init() {
	if (constructor != null) {
		address = EpicNative.getMethodAddress(constructor);
	} else {
		address = EpicNative.getMethodAddress(method);
	}
}
```

## 执行hook

1.创建MethodInfo并保存
2.确保static方法对于的类被加载了
3.触发jit确保origin & target被jit过了
4.backup Method
5.createTrampoline
6.install trampoline

```java
// Epic.java
private static boolean hookMethod(ArtMethod artOrigin) {
		// 1.保存method的基础信息
        MethodInfo methodInfo = new MethodInfo();
        methodInfo.isStatic = Modifier.isStatic(artOrigin.getModifiers());
        final Class<?>[] parameterTypes = artOrigin.getParameterTypes();
        if (parameterTypes != null) {
            methodInfo.paramNumber = parameterTypes.length;
            methodInfo.paramTypes = parameterTypes;
        } else {
            methodInfo.paramNumber = 0;
            methodInfo.paramTypes = new Class<?>[0];
        }
        methodInfo.returnType = artOrigin.getReturnType();
        methodInfo.method = artOrigin;
        originSigs.put(artOrigin.getAddress(), methodInfo);

        if (!artOrigin.isAccessible()) {
            artOrigin.setAccessible(true);
        }
    	// 2.确保static方法初始化
        // 由于static方法是懒加载的，因此需要确保static方法被调用过。
        // 方法也很粗暴针对于static,强制执行一次调用。
        artOrigin.ensureResolved();

        long originEntry = artOrigin.getEntryPointFromQuickCompiledCode();
    
    	// 3.触发方法的jit
        // check 方法的entryPoint是否是解释器入口
        // 确保方法被jit过(不hook解释器)
        if (originEntry == ArtMethod.getQuickToInterpreterBridge()) { // hook方法入口为解释器。
            Logger.i(TAG, "this method is not compiled, compile it now. current entry: 0x" + Long.toHexString(originEntry));
            // 触发一次jit
            boolean ret = artOrigin.compile();
            if (ret) {
                originEntry = artOrigin.getEntryPointFromQuickCompiledCode();
                Logger.i(TAG, "compile method success, new entry: 0x" + Long.toHexString(originEntry));
            } else {
                Logger.e(TAG, "compile method failed...");
                return false;
                // return hookInterpreterBridge(artOrigin);
            }
        }
    	// 4. backup method
		// 将原ArtMethod拷贝一份
        ArtMethod backupMethod = artOrigin.backup();
		
        Logger.i(TAG, "backup method address:" + Debug.addrHex(backupMethod.getAddress()));
        Logger.i(TAG, "backup method entry :" + Debug.addrHex(backupMethod.getEntryPointFromQuickCompiledCode()));
		// 建立修改前后的ArtMethod的联系。
        ArtMethod backupList = getBackMethod(artOrigin);
        if (backupList == null) {
            setBackMethod(artOrigin, backupMethod);
        }

        final long key = originEntry;
        final EntryLock lock = EntryLock.obtain(originEntry);
        //noinspection SynchronizationOnLocalVariableOrMethodParameter
        // 加锁防止并发
        synchronized (lock) {
            // 创建shellcode
            if (!scripts.containsKey(key)) {
                scripts.put(key, new Trampoline(ShellCode, originEntry));
            }
            Trampoline trampoline = scripts.get(key);
            // install shellcode.
            // 5.createTrampoline
            // 6.install trampoline
            // Note: trampoline.install包含trampoline的创建以及install操作。
            // 具体分析可见下方章节
            boolean ret = trampoline.install(artOrigin);
            // Logger.d(TAG, "hook Method result:" + ret);
            return ret;
        }
    }
```



### MethodInfo


这一步骤主要中创建了一个MethodInfo对象并对字段进行了初始化，如methodInfo/isStatic/paramNumber/paramTypes/returnType/method
最终初始化完成以后将MethodInfo对象存储到originSigs这么一个map里面去，其中key为虚拟ArtMethod的地址，value为MethodInfo.（很好理解，方便hook过程需要获取这部分信息）
```java
// 保存method的基础信息
MethodInfo methodInfo = new MethodInfo();
methodInfo.isStatic = Modifier.isStatic(artOrigin.getModifiers());
final Class<?>[] parameterTypes = artOrigin.getParameterTypes();
if (parameterTypes != null) {
	methodInfo.paramNumber = parameterTypes.length;
	methodInfo.paramTypes = parameterTypes;
} else {
	methodInfo.paramNumber = 0;
	methodInfo.paramTypes = new Class<?>[0];
}
methodInfo.returnType = artOrigin.getReturnType();
methodInfo.method = artOrigin;
originSigs.put(artOrigin.getAddress(), methodInfo);

// 确保原始的可以进行反射调用。
if (!artOrigin.isAccessible()) {
	artOrigin.setAccessible(true);
}
```

### ensureResolved


如果hook的方法是static方法则需要考虑方法初始化的问题。因为static方法是懒加载的，因此若方法一直没有被调用那么一直不会初始化。
因此epic在hook的过程中为了确保static方法一定是初始化完成的，会手动调用一次原始方法。
```java
// Epic.hookMethod
artOrigin.ensureResolved();

// ArtMethod.java
public void ensureResolved() {
	if (!Modifier.isStatic(getModifiers())) {
		Logger.d(TAG, "not static, ignore.");
		return;
	}
	
	// 为确保static方法已经初始化手动调用方法

	try {
		invoke(null);
		Logger.d(TAG, "ensure resolved");
	} catch (Exception ignored) {
		// we should never make a successful call.
	} finally {
		EpicNative.MakeInitializedClassVisibilyInitialized();
	}
}

```


### jit原始方法

此过程会确保被hook的原始方法已经jit过了。
```java

// 获取虚拟机底层的compile_code
long originEntry = artOrigin.getEntryPointFromQuickCompiledCode();
// check 方法的entryPoint是否是解释器入口
// 确保方法被jit过(不hook解释器)
if (originEntry == ArtMethod.getQuickToInterpreterBridge()) { // hook方法入口为解释器。
	Logger.i(TAG, "this method is not compiled, compile it now. current entry: 0x" + Long.toHexString(originEntry));
	// 触发一次jit
	boolean ret = artOrigin.compile();
	if (ret) {
		originEntry = artOrigin.getEntryPointFromQuickCompiledCode();
		Logger.i(TAG, "compile method success, new entry: 0x" + Long.toHexString(originEntry));
	} else {
		// 如果jit失败，直接跳过hook
		Logger.e(TAG, "compile method failed...");
		return false;
		// return hookInterpreterBridge(artOrigin);
	}
}
// 将原ArtMethod拷贝一份
ArtMethod backupMethod = artOrigin.backup();
```

- ***为什么需要触发jit呢？***
java方法有三类执行方式jit/aot/解释执行。具体如下
1.jit/aot执行模式
	这两种执行模式类似，都是是将dalvik字节码编译成了对应平台的汇编代码(编译的时机不一样，aot是运行前，jit是运行后针对于热点代码进行编译)，并替换了entry_point_from_quick_compiled_code_的地址（也就是说他可以不依赖dex文件执行，不需要从dex中取指令）
2.解释执行
	解释执行是利用解释器，从dex文件中取出一条条dalvik code并将其解释为一条条汇编指令。他没有替换entry_point_from_quick_compiled_code_，而且所有解释执行的方法的entry_point_from_quick_compiled_code_都是共享的。
由于**解释执行是共享的compile_code**因此，我们如果对某个解释执行的方法进行hook就需要对整个解释器方法进行hook，这可能会**波及很多我们压根不关注的方法，导致性能损耗**。因此我们需要将其转为jit/aot方法，这样我们就能缩小波及范围，降低性能损耗。

- ***epic是如何触发jit的呢？***
我们先看Java代码，Java代码会一层层调用到一个JNI方法

```java
artOrigin.compile();

// ArtMethod.java
 public boolean compile() {
	if (constructor != null) {
		return EpicNative.compileMethod(constructor);
	} else {
		return EpicNative.compileMethod(method);
	}
}

// EpicNative.java
public static boolean compileMethod(Member method) {
	final long nativePeer = XposedHelpers.getLongField(Thread.currentThread(), "nativePeer");
	return compileMethod(method, nativePeer);
}

public static native boolean compileMethod(Member method, long self);

```

JNI逻辑有点长，主要原因是需要适配不同的系统版本
```c++

jboolean epic_compile(JNIEnv *env, jclass, jobject method, jlong self) {
    LOGV("self from native peer: %p, from register: %p", reinterpret_cast<void*>(self), __self());
    jlong art_method = (jlong) env->FromReflectedMethod(method);
    if (art_method % 2 == 1) {
        art_method = reinterpret_cast<jlong>(JniIdManager_DecodeMethodId_(ArtHelper::getJniIdManager(), art_method));
    }
    bool ret;
    if (api_level >= 30) {
        void* current_region = JitCodeCache_GetCurrentRegion(ArtHelper::getJitCodeCache());
        if (api_level >= 31) {
            ret = ((JIT_COMPILE_METHOD4)jit_compile_method_)(jit_compiler_handle_, reinterpret_cast<void*>(self),
                                                             reinterpret_cast<void*>(current_region),
                                                             reinterpret_cast<void*>(art_method), 1);
        } else {
            ret = ((JIT_COMPILE_METHOD3)jit_compile_method_)(jit_compiler_handle_, reinterpret_cast<void*>(self),
                                                             reinterpret_cast<void*>(current_region),
                                                             reinterpret_cast<void*>(art_method), false, false);
        }
    } else if (api_level >= 29) {
        ret = ((JIT_COMPILE_METHOD2) jit_compile_method_)(jit_compiler_handle_,
                                                          reinterpret_cast<void *>(art_method),
                                                          reinterpret_cast<void *>(self), false, false);
    } else {
        ret = ((JIT_COMPILE_METHOD1) jit_compile_method_)(jit_compiler_handle_,
                                                          reinterpret_cast<void *>(art_method),
                                                          reinterpret_cast<void *>(self), false);
    }
    return (jboolean)ret;
}
```

\>=Android 11系统版本情况下，调用art::jit::JitCompiler::CompileMethod就行了 
```c++

jit_compile_method_ = (bool (*)(void *, void *, void *, bool)) dlsym_ex(jit_lib, "jit_compile_method");
// .....

if (api_level >= 30) {
	// Android R would not directly return ArtMethod address but an internal id
	// ......
	if (api_level >= 31) {
		// Android S CompileMethod accepts a CompilationKind enum instead of two booleans
		// source: https://android.googlesource.com/platform/art/+/refs/heads/android12-release/compiler/jit/jit_compiler.cc
		jit_compile_method_ = (bool (*)(void *, void *, void *, bool)) dlsym_ex(jit_lib, "_ZN3art3jit11JitCompiler13CompileMethodEPNS_6ThreadEPNS0_15JitMemoryRegionEPNS_9ArtMethodENS_15CompilationKindE");
	} else {
		jit_compile_method_ = (bool (*)(void *, void *, void *, bool)) dlsym_ex(jit_lib, "_ZN3art3jit11JitCompiler13CompileMethodEPNS_6ThreadEPNS0_15JitMemoryRegionEPNS_9ArtMethodEbb");
	}
}
```

### backUpMethod

```java
ArtMethod backupMethod = artOrigin.backup();

Logger.i(TAG, "backup method address:" + Debug.addrHex(backupMethod.getAddress()));
Logger.i(TAG, "backup method entry :" + Debug.addrHex(backupMethod.getEntryPointFromQuickCompiledCode()));

ArtMethod backupList = getBackMethod(artOrigin);
if (backupList == null) {
	setBackMethod(artOrigin, backupMethod);
}
```

- ArtMethod.backup()
深拷贝一个ArtMethod方法，为什么是深拷贝呢，因为原始方法在hook以后会被修改，在hook代码执行以后难免会有执行origin方法的需要，因此epic会对方法进行一次深拷贝(包含方法jit/aot汇编指令)
```java
public ArtMethod backup() {
        try {
            // Before Oreo, it is: java.lang.reflect.AbstractMethod
            // After Oreo, it is: java.lang.reflect.Executable
            Class<?> abstractMethodClass = Method.class.getSuperclass();

            Object executable = this.getExecutable();
            ArtMethod artMethod;
            if (Build.VERSION.SDK_INT < 23) {
                Class<?> artMethodClass = Class.forName("java.lang.reflect.ArtMethod");
                //Get the original artMethod field
                Field artMethodField = abstractMethodClass.getDeclaredField("artMethod");
                if (!artMethodField.isAccessible()) {
                    artMethodField.setAccessible(true);
                }
                Object srcArtMethod = artMethodField.get(executable);

                Constructor<?> constructor = artMethodClass.getDeclaredConstructor();
                constructor.setAccessible(true);
                Object destArtMethod = constructor.newInstance();

                //Fill the fields to the new method we created
                for (Field field : artMethodClass.getDeclaredFields()) {
                    if (!field.isAccessible()) {
                        field.setAccessible(true);
                    }
                    field.set(destArtMethod, field.get(srcArtMethod));
                }
                Method newMethod = Method.class.getConstructor(artMethodClass).newInstance(destArtMethod);
                newMethod.setAccessible(true);
                artMethod = ArtMethod.of(newMethod);

                artMethod.setEntryPointFromQuickCompiledCode(getEntryPointFromQuickCompiledCode());
                artMethod.setEntryPointFromJni(getEntryPointFromJni());
            } else {
                Constructor<Method> constructor = Method.class.getDeclaredConstructor();
                // we can't use constructor.setAccessible(true); because Google does not like it
                // AccessibleObject.setAccessible(new AccessibleObject[]{constructor}, true);
                Field override = AccessibleObject.class.getDeclaredField(
                        Build.VERSION.SDK_INT == Build.VERSION_CODES.M ? "flag" : "override");
                override.setAccessible(true);
                override.set(constructor, true);

                Method m = constructor.newInstance();
                m.setAccessible(true);
                for (Field field : abstractMethodClass.getDeclaredFields()) {
                    field.setAccessible(true);
                    field.set(m, field.get(executable));
                }
                Field artMethodField = abstractMethodClass.getDeclaredField("artMethod");
                artMethodField.setAccessible(true);
                int artMethodSize = getArtMethodSize();
                long memoryAddress = EpicNative.map(artMethodSize);

                byte[] data = EpicNative.get(address, artMethodSize);
                EpicNative.put(data, memoryAddress);
                artMethodField.set(m, memoryAddress);
                // From Android R, getting method address may involve the jni_id_manager which uses
                // ids mapping instead of directly returning the method address. During resolving the
                // id->address mapping, it will assume the art method to be from the "methods_" array
                // in class. However this address may be out of the range of the methods array. Thus
                // it will cause a crash during using the method offset to resolve method array.
                artMethod = ArtMethod.of(m, memoryAddress);
            }
            artMethod.makePrivate();
            artMethod.setAccessible(true);
            artMethod.origin = this; // save origin method.
            return artMethod;


        } catch (Throwable e) {
            Log.e(TAG, "backup method error:", e);
            throw new IllegalStateException("Cannot create backup method from :: " + getExecutable(), e);
        }
    }
```

- 保存
将新复制的方法保存到一个map中去，方便后续调用原始方法。
```java
ArtMethod backupList = getBackMethod(artOrigin);
if (backupList == null) {
	setBackMethod(artOrigin, backupMethod);
}

private static final Map<String, ArtMethod> backupMethodsMapping = new ConcurrentHashMap<>();

public synchronized static ArtMethod getBackMethod(ArtMethod origin) {  
    String identifier = origin.getIdentifier();  
    return backupMethodsMapping.get(identifier);  
}  
  
public static synchronized void setBackMethod(ArtMethod origin, ArtMethod backup) {  
    String identifier = origin.getIdentifier();  
    backupMethodsMapping.put(identifier, backup);  
}
```
### createTrampoline

到这为止，hook方法的逻辑还剩余：
1.加锁，防止多线程竞争为后续创建并添加trampoline做准备
2.创建Trampoline对象
3.安装trampoline
	a.创建跳板函数
	b.将跳板函数安装到虚拟机quick_compile_code中

> Note: 为了防止混淆，这里说明下，createTrampoline主要分析上述#2, #3.a 逻辑
```java
// #1 加锁，防止多线程竞争为后续创建并添加trampoline做准备
final long key = originEntry;
final EntryLock lock = EntryLock.obtain(originEntry);
//noinspection SynchronizationOnLocalVariableOrMethodParameter
synchronized (lock) {
	if (!scripts.containsKey(key)) {
		// #2 创建Trampoline对象
		scripts.put(key, new Trampoline(ShellCode, originEntry));
	}
	Trampoline trampoline = scripts.get(key);
	// #3 安装trampoline
	boolean ret = trampoline.install(artOrigin);
	// Logger.d(TAG, "hook Method result:" + ret);
	return ret;
}

```

####  Trampoline对象创建
我们来看看构造方法里面做了什么
```java
// shellCode - 静态对象
// entryPoint - 原始方法的entry_point_from_quick_compiled_code_地址。
Trampoline(ShellCode shellCode, long entryPoint) {
	this.shellCode = shellCode;
	// toMem实现是直接返回entryPoint，方法等价于  this.jumpToAddress = entryPoint;
	this.jumpToAddress = shellCode.toMem(entryPoint);
	// 复制原始方法的entry_point_from_quick_compiled_code_前4个指令做备份
	// (因为后面这四个指令会被覆盖为一个jump指令，跳转到hook方法。后文一级跳板会做详细介绍)
	this.originalCode = EpicNative.get(jumpToAddress, shellCode.sizeOfDirectJump());
}
```
***此处有一个迷雾，shellCode是什么？***
ShellCode是为了实现不同指令级架构的兼容，为不同的指令生成不同的跳板指令。(我们关心的一级跳板 & 二级跳板都藏在这个ShellCode实现类里面)
```java
// Epic.java 

private static ShellCode ShellCode;

static {
	boolean isArm = true; // TODO: 17/11/21 TODO
	int apiLevel = Build.VERSION.SDK_INT;
	boolean thumb2 = true;
	if (isArm) {
		// 构造方法均为默认构造(就不贴了)
		if (Runtime.is64Bit()) {
			ShellCode = new Arm64();
		} else if (Runtime.isThumb2()) {
			ShellCode = new Thumb2();
		} else {
			thumb2 = false;
			ShellCode = new Thumb2();
			Logger.w(TAG, "ARM32, not support now.");
		}
	}
	if (ShellCode == null) {
		throw new RuntimeException("Do not support this ARCH now!! API LEVEL:" + apiLevel + " thumb2 ? : " + thumb2);
	}
	Logger.i(TAG, "Using: " + ShellCode.getName());
}
```

#### 跳板函数创建

跳板函数的创建在如下方法调用中
```java
boolean ret = trampoline.install(artOrigin);
```
Trampoline.install 包含有两个过程（但是我们只针对于跳板创建进行分析，跳板的安装过程会在后面installTrampoline进行分析）
- 创建跳板
- 安装跳板
```java

// Trampoline
public boolean install(ArtMethod originMethod){
	boolean modified = segments.add(originMethod);
	if (!modified) {
		// Already hooked, ignore
		Logger.d(TAG, originMethod + " is already hooked, return.");
		return true;
	}
	// #1 创建跳板
	byte[] page = create();
	EpicNative.put(page, getTrampolineAddress());

	// #2 安装跳板 
	int quickCompiledCodeSize = Epic.getQuickCompiledCodeSize(originMethod);
	int sizeOfDirectJump = shellCode.sizeOfDirectJump();
	if (quickCompiledCodeSize < sizeOfDirectJump) {
		Logger.w(TAG, originMethod.toGenericString() + " quickCompiledCodeSize: " + quickCompiledCodeSize);
		originMethod.setEntryPointFromQuickCompiledCode(getTrampolinePc());
		return true;
	}
	// 这里是绝对不能改EntryPoint的，碰到GC就挂(GC暂停线程的时候，遍历所有线程堆栈，如果被hook的方法在堆栈上，那就GG)
	// source.setEntryPointFromQuickCompiledCode(script.getTrampolinePc());
	return activate();
}

```

Trampoline的创建对应于方法create()，准确来说create()方法创建的是二级跳板方法。
具体逻辑如下：
1.为所有segments生成一个跳板
2.叠加一个跳板用于跳转到原始方法。
```java

private byte[] create() {
	Logger.d(TAG, "create trampoline." + segments);
	byte[] mainPage = new byte[getSize()];

	int offset = 0;
	// #1 遍历所有的segments为每个片段单独生成一段trampoline
	for (ArtMethod method : segments) {
		// #1 创建 trampoline 
		byte[] bridgeJump = createTrampoline(method);
		int length = bridgeJump.length;
		System.arraycopy(bridgeJump, 0, mainPage, offset, length);
		offset += length;
	}
	// 调用原始方法
	// #2 创建call origin trampoline
	byte[] callOriginal = shellCode.createCallOrigin(jumpToAddress, originalCode);
	System.arraycopy(callOriginal, 0, mainPage, offset, callOriginal.length);

	return mainPage;
}

```

Note: 对于上述跳板创建逻辑，在多嘴一句。
\#1是为同一个quick_compile_code被多个java函数共享设计的 (如果方法体的内容是一样，即使是不同方法art在编译的时候可能会将方法设置为同一个quick_compile_code)
\#2是为了调用原始方法(方法不在hook列表需要返回原始方法的调用逻辑)

##### segments跳板

大家可能对segements是什么感到陌生，这里做下介绍。
- segments是什么？
我们先回顾一个事情，前面提到了两个代码完全相同的方法，最终quick_compile_code可能是同一个。但是我们实际做hook的时候可能只是想hook某个特定的方法。
比如A、B两个方法的quick_compile_code是一样的，但是我最终只想hook方法A。怎么办呢？这时候我们就需要分类了，segments就是用来做分类的，segments存储了某个特定的quick_compile_code中需要hook的方法列表。
假如quick_compile_code1会被100个方法用到a1,a2,...a100,但是我只想hook a1和a2，那么segments就存储a1,a2这两个。
- segments跳板是什么？
我们理解了segments的概念，就不难理解segments跳板的概念了。由于我们**一个quick_compile_code上需要挂载多个hook函数**，而且这些hook函数会被存放到segments中，那么我们方法实际被调用到的时候肯定是**需要逐一询问每一个hook函数是否执行**。
而segments跳板做的就是这个事情，有点类似于**责任链**，逐一询问segments中的函数是否需要发生hook,如果需要hook,就直接拦截原始方法，跳转到特定的hook函数中，如果不需要hook则直接跳过并询问下一个hook函数，如果所有的hook函数都不hook, 那么最后就调用一个**跳板函数返回到原始方法中执行**。（每一个方法末尾都有一个跳板函数，具体可见“原始跳板”处的分析）

createTrampoline方法是为特定的某个函数创建跳板的片段，外部createTrampoline的调用处是一个for循环。而这个片段的跳板方法最终会通过调用createBridgeJump创建。
```java
private byte[] createTrampoline(ArtMethod source){
	// 获取方法的基础信息 
	final Epic.MethodInfo methodInfo = Epic.getMethodInfo(source.getAddress());
	final Class<?> returnType = methodInfo.returnType;

	// 获取桥接方法
	Method bridgeMethod = Runtime.is64Bit() ? Entry64.getBridgeMethod(returnType)
			: Entry.getBridgeMethod(returnType);
	// 获取ArtMethod
	final ArtMethod target = ArtMethod.of(bridgeMethod);
	// 获取ArtMethod的address
	long targetAddress = target.getAddress();
	// 获取地址
	long targetEntry = target.getEntryPointFromQuickCompiledCode();
	// 获取art ArtMethod字段位置
	long sourceAddress = source.getAddress();
	// 分配内存
	long structAddress = EpicNative.malloc(4);
	
	Logger.d(TAG, "targetAddress:"+ Debug.longHex(targetAddress));
	Logger.d(TAG, "sourceAddress:"+ Debug.longHex(sourceAddress));
	Logger.d(TAG, "targetEntry:"+ Debug.longHex(targetEntry));
	Logger.d(TAG, "structAddress:"+ Debug.longHex(structAddress));
	// 创建jump bridge
	// targetAddress - 桥接方法底层C++ ArtMethod的地址
	// targetEntry   - 桥接方法的compile_code地址
	// sourceAddress - 原方法底层C++ ArtMethod的地址
	// structAddress - 结构体地址
	return shellCode.createBridgeJump(targetAddress, targetEntry, sourceAddress, structAddress);
}

```


创建跳板方法
```java
// 
@Override
public byte[] createBridgeJump(long targetAddress, long targetEntry, long srcAddress, long structAddress) {

	byte[] instructions = new byte[]{
			0x1f, 0x20, 0x03, (byte) 0xd5,         // nop
			0x69, 0x02, 0x00, 0x58,                // ldr x9, source_method
			0x1f, 0x00, 0x09, (byte) 0xeb,         // cmp x0, x9
			(byte) 0xa1, 0x02, 0x00, 0x54,         // bne 5f
			(byte) 0x80, 0x01, 0x00, 0x58,         // ldr x0, target_method

			0x29, 0x02, 0x00, 0x58,                // ldr x9, struct
			(byte) 0xea, 0x03, 0x00, (byte) 0x91,  // mov x10, sp

			0x2a, 0x01, 0x00, (byte) 0xf9,         // str x10, [x9, #0]
			0x22, 0x05, 0x00, (byte) 0xf9,         // str x2, [x9, #8]

			0x23, 0x09, 0x00, (byte) 0xf9,         // str x3, [x9, #16]
			(byte) 0xe3, 0x03, 0x09, (byte) 0xaa,  // mov x3, x9
			0x22, 0x01, 0x00, 0x58,                // ldr x2, source_method
			0x22, 0x0d, 0x00, (byte) 0xf9,         // str x2, [x9, #24]
			(byte) 0xe2, 0x03, 0x13, (byte) 0xaa,  // mov x2, x19
			(byte) 0x89, 0x00, 0x00, 0x58,         // ldr x9, target_method_entry
			0x20, 0x01, 0x1f, (byte) 0xd6,         // br x9

			0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, // target_method_address
			0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, // target_method_entry
			0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, // source_method
			0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00  // struct

	};


	writeLong(targetAddress, ByteOrder.LITTLE_ENDIAN, instructions,
			instructions.length - 32);
	writeLong(targetEntry, ByteOrder.LITTLE_ENDIAN, instructions,
			instructions.length - 24);
	writeLong(srcAddress,
			ByteOrder.LITTLE_ENDIAN, instructions, instructions.length - 16);
	writeLong(structAddress, ByteOrder.LITTLE_ENDIAN, instructions,
			instructions.length - 8);

	return instructions;
}
```

汇编代码，通过分析发现判断
```asm

0000000000000000 <.data>:
   // 指令对齐
   0:   d503201f        nop
   // 1. 判断"原方法"和"当前执行的方法"是否是同一个（确保没有hook 不应该hook的方法）
   // 加载需要hook的artMethod内存地址
   4:   58000269        ldr     x9, 0x50
   // 对比当前的artMethod和需要hook的artMethod是否一致
   8:   eb09001f        cmp     x0, x9
   // 如果不一致跳转到0x60处，0x60只是当前case的一个跳转点
   //  (其实这个是相对pc跳转，0x60的位置正好是当前跳板的结束位置，是下一个hook函数的起始位置。)
   c:   540002a1        b.ne    0x60  // b.any
   // 2. 保存准备方法调用参数x0、x2、x3
   // 将target_method_address加载到x0中
  10:   58000180        ldr     x0, 0x40
  // 将结构体地址加载到x9中
  14:   58000229        ldr     x9, 0x58
  // 将sp设置到x10中
  18:   910003ea        mov     x10, sp
  // 将sp存放在结构体[0,8]位置
  1c:   f900012a        str     x10, [x9]
  // 将x2保存到结构体[8,16]的位置
  20:   f9000522        str     x2, [x9, #8]
  // 将x3保存到结构体[16,26]的位置
  24:   f9000923        str     x3, [x9, #16]
  // 将x3设置为结构体地址
  28:   aa0903e3        mov     x3, x9
  // 将source_method保存到x2中
  2c:   58000122        ldr     x2, 0x50
  // 将source_method保存到结构体[24,32]位置
  30:   f9000d22        str     x2, [x9, #24]
  // 将x19(Thread::Current)的值保存到x2中
  34:   aa1303e2        mov     x2, x19
  // 加载target_method_address到x9中
  38:   58000089        ldr     x9, 0x48
  // 跳转到target_method_address中去
  // x0 桥接方法的artMethod对象地址
  // x1 第一个参数
  // x2 Thread::Current
  // x3 结构体地址[sp,x2,x3,方法原始ArtMethod]
  3c:   d61f0120        br      x9
  40    00 00 00 00 00 00 00 00 - target_method_address 
  48    00 00 00 00 00 00 00 00 - target_method_entry
  50    00 00 00 00 00 00 00 00 - source_method
  58    00 00 00 00 00 00 00 00 - struct
  60    ......
```

##### 原始跳板

调用原始方法。
```java
public byte[] createCallOrigin(long originalAddress, byte[] originalPrologue) {
	// 原始ArtMethod entryPoint前4 * 4字节信息
	byte[] callOriginal = new byte[sizeOfCallOrigin()];
	System.arraycopy(originalPrologue, 0, callOriginal, 0, sizeOfDirectJump());
	// 跳转到特定的地址。
	byte[] directJump = createDirectJump(toPC(originalAddress + sizeOfDirectJump()));
	System.arraycopy(directJump, 0, callOriginal, sizeOfDirectJump(), directJump.length);
	return callOriginal;
}

// 跳板方法，用于跳转到特定的地址
@Override  
public byte[] createDirectJump(long targetAddress) {  
    byte[] instructions = new byte[]{  
            0x50, 0x00, 0x00, 0x58,         // ldr x9, _targetAddress  
            0x00, 0x02, 0x1F, (byte) 0xD6,  // br x9  
            0x00, 0x00, 0x00, 0x00,         // targetAddress  
            0x00, 0x00, 0x00, 0x00          // targetAddress  
    };  
    writeLong(targetAddress, ByteOrder.LITTLE_ENDIAN, instructions, instructions.length - 8);  
    return instructions;  
}
```
实际的汇编指令（总共8个指令，占据4 * 4 * 2指令）

```asm

// 原始ArtMethod的前4个指令(因为这四个指令会被覆盖)
...
...
...
...
ldr x9, _targetAddress  
br x9
00 00 00 00 00 00 00 00 -- _targetAddress（保存这ArtMethod + 4的地址）
```



### installTrampoline

Trampoline的安装逻辑即将生成的跳板方法“安装”到特定的地址，取决于entry_point的空间大小：
- 小于一层跳板(4 * 4字节)的大小：修改原始方法的entry_point为二层跳板的地址
- 大于等于一层表板的大小：将原始方法的entry_point前4个指令(4 * 4字节)覆盖为一层跳板的地址。
```java
// entry_point大小小于一层跳板大小
if (quickCompiledCodeSize < sizeOfDirectJump) {
	Logger.w(TAG, originMethod.toGenericString() + " quickCompiledCodeSize: " + quickCompiledCodeSize);
	originMethod.setEntryPointFromQuickCompiledCode(getTrampolinePc());
	return true;
}

// entry_point大小大于等于一层跳板大小
// 这里是绝对不能改EntryPoint的，碰到GC就挂(GC暂停线程的时候，遍历所有线程堆栈，如果被hook的方法在堆栈上，那就GG)
// source.setEntryPointFromQuickCompiledCode(script.getTrampolinePc());
return activate();
```

在讲安装逻辑之前需要说明上面提到的两个概念：
- 一层跳板
	通过shellCode.createDirectJump(pc)创建，其中pc为二层跳板的地址，该指令用于拦截程序执行流程到二级跳板
- 二层跳板
	通过Trampoline.create创建，


1.entry_point无法容纳一层跳板
```java
if (quickCompiledCodeSize < sizeOfDirectJump) {
	Logger.w(TAG, originMethod.toGenericString() + " quickCompiledCodeSize: " + quickCompiledCodeSize);
	// 直接设置原始方法的entry_point为二层跳板地址
	originMethod.setEntryPointFromQuickCompiledCode(getTrampolinePc());
	return true;
}
```
2.entry_point能容纳一层跳板
```java

private boolean activate() {
	long pc = getTrampolinePc();
	Logger.d(TAG, "Writing direct jump entry " + Debug.addrHex(pc) + " to origin entry: 0x" + Debug.addrHex(jumpToAddress));
	synchronized (Trampoline.class) {
	    // jumpToAddress —— 原始方法的entryPoint地址
	    // pc —— Trampoline.create方法创建的二级跳板地址
	    // shellCode.sizeOfDirectJump() —— jump跳转的大小(arm64下大小为4 * 4)
	    // shellCode.sizeOfBridgeJump() —— 单个ArtMethod桥接方法的大小
	    // shellCode.createDirectJump(pc) —— 创建一层跳板方法(byte[])
		return EpicNative.activateNative(jumpToAddress, pc, shellCode.sizeOfDirectJump(),
				shellCode.sizeOfBridgeJump(), shellCode.createDirectJump(pc));
	}
}
```

- directJump
```java

public byte[] createDirectJump(long targetAddress) {
	byte[] instructions = new byte[]{
			0x50, 0x00, 0x00, 0x58,         // ldr x9, _targetAddress
			0x00, 0x02, 0x1F, (byte) 0xD6,  // br x9
			0x00, 0x00, 0x00, 0x00,         // targetAddress
			0x00, 0x00, 0x00, 0x00          // targetAddress
	};
	writeLong(targetAddress, ByteOrder.LITTLE_ENDIAN, instructions, instructions.length - 8);
	return instructions;
}
```


- activateNative
  逻辑非常简单，就是用一层跳板覆盖原始方法entry_point的前4个字节。

  (需要注意的是>= SDK 24的系统版本会在hook过程中暂停所有线程！！！)
```c++

jboolean epic_activate(JNIEnv* env, jclass jclazz, jlong jumpToAddress, jlong pc, jlong sizeOfDirectJump,
                       jlong sizeOfBridgeJump, jbyteArray code) {

    // fetch the array, we can not call this when thread suspend(may lead deadlock)
    jbyte *srcPnt = env->GetByteArrayElements(code, 0);
    jsize length = env->GetArrayLength(code);

    jlong cookie = 0;
    bool isNougat = api_level >= 24;
    // #1 挂起所有的线程(>= SDK 24)
    if (isNougat) {
        // We do thus things:
        // 1. modify the code mprotect
        // 2. modify the code

        // Ideal, this two operation must be atomic. Below N, this is safe, because no one
        // modify the code except ourselves;
        // But in Android N, When the jit is working, between our step 1 and step 2,
        // if we modity the mprotect of the code, and planning to write the code,
        // the jit thread may modify the mprotect of the code meanwhile
        // we must suspend all thread to ensure the atomic operation.

        LOGV("suspend all thread.");
        cookie = epic_suspendAll(env, jclazz);
    }

	// #2 使用mprotect调整内存权限
    jboolean result = epic_munprotect(env, jclazz, jumpToAddress, sizeOfDirectJump);
    if (result) {
	    // 使用一层跳板覆盖原始artMethod的前4个指令
        unsigned char *destPnt = (unsigned char *) jumpToAddress;
        for (int i = 0; i < length; ++i) {
            destPnt[i] = (unsigned char) srcPnt[i];
        }
        // 缓存刷新
        jboolean ret = epic_cacheflush(env, jclazz, pc, sizeOfBridgeJump);
        if (!ret) {
            LOGV("cache flush failed!!");
        }
    } else {
	    // 异常case打印异常日志
        LOGV("Writing hook failed: Unable to unprotect memory at %d", jumpToAddress);
    }
	// 恢复执行
    if (cookie != 0) {
        LOGV("resume all thread.");
        epic_resumeAll(env, jclazz, cookie);
    }

    env->ReleaseByteArrayElements(code, srcPnt, 0);
    return result;
}
```

## 总结



# 方法桥接

桥接方法列表，这部分入口都是根据方法的返回值类型定的

```java
// Entry64.java
//region ---------------bridge---------------
private static void voidBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
	 referenceBridge(r1, self, struct, x4, x5, x6, x7);
}

private static boolean booleanBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
	return (Boolean) referenceBridge(r1, self, struct, x4, x5, x6, x7);
}

private static byte byteBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
	return (Byte) referenceBridge(r1, self, struct, x4, x5, x6, x7);
}

private static short shortBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
	return (Short) referenceBridge(r1, self, struct, x4, x5, x6, x7);
}

private static char charBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
	return (Character) referenceBridge(r1, self, struct, x4, x5, x6, x7);
}

private static int intBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
	return (Integer) referenceBridge(r1, self, struct, x4, x5, x6, x7);
}

private static long longBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
	return (Long) referenceBridge(r1, self, struct, x4, x5, x6, x7);
}

private static float floatBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
	return (Float) referenceBridge(r1, self, struct, x4, x5, x6, x7);
}

private static double doubleBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
	return (Double) referenceBridge(r1, self, struct, x4, x5, x6, x7);
}
//endregion
```
最终都会回到到referenceBridge这样一个方法
```java
private static Object referenceBridge(long x1, long self, long struct, long x4, long x5, long x6, long x7) {
        Logger.i(TAG, "enter bridge function.");

        // struct {
        //     void* sp;
        //     void* r2;
        //     void* r3;
        //     void* sourceMethod
        // }
        // sp + 16 = r4
        
        // 获取参数

		// 获取当前线程的nativePeer地址——即Native Thread对象
        Logger.d(TAG, "self:" + Long.toHexString(self));
        final long nativePeer = XposedHelpers.getLongField(Thread.currentThread(), "nativePeer");
        Logger.d(TAG, "java thread native peer:" + Long.toHexString(nativePeer));

        Logger.d(TAG, "struct:" + Long.toHexString(struct));
		// 获取sp寄存器地址
        final long sp = ByteBuffer.wrap(EpicNative.get(struct, 8)).order(ByteOrder.LITTLE_ENDIAN).getLong();

        Logger.d(TAG, "stack:" + sp);
		// 将x1写入新的内存中去
        final byte[] rr1 = ByteBuffer.allocate(8).order(ByteOrder.LITTLE_ENDIAN).putLong(x1).array();
        // 获取x2寄存器
        final byte[] r2 = EpicNative.get(struct + 8, 8);
		// 获取x3寄存器
        final byte[] r3 = EpicNative.get(struct + 16, 8);
		// 获取sourceMethod
        final long sourceMethod = ByteBuffer.wrap(EpicNative.get(struct + 24, 8)).order(ByteOrder.LITTLE_ENDIAN).getLong();
        Logger.d(TAG, "sourceMethod:" + Long.toHexString(sourceMethod));
		// 根据sourceMethod获取方法的详细信息
        Epic.MethodInfo originMethodInfo = Epic.getMethodInfo(sourceMethod);
        Logger.d(TAG, "originMethodInfo :" + originMethodInfo);
	
        boolean isStatic = originMethodInfo.isStatic;

        int numberOfArgs = originMethodInfo.paramNumber;
        Class<?>[] typeOfArgs = originMethodInfo.paramTypes;
        Object[] arguments = new Object[numberOfArgs];

        Object receiver;
		// 解析方法参数。
        self = nativePeer;

        if (isStatic) {
            receiver = null;
            do {
                if (numberOfArgs == 0) break;
                arguments[0] = wrapArgument(typeOfArgs[0], self, rr1);
                if (numberOfArgs == 1) break;
                arguments[1] = wrapArgument(typeOfArgs[1], self, r2);
                if (numberOfArgs == 2) break;
                arguments[2] = wrapArgument(typeOfArgs[2], self, r3);
                if (numberOfArgs == 3) break;
                arguments[3] = wrapArgument(typeOfArgs[3], self, x4);
                if (numberOfArgs == 4) break;
                arguments[4] = wrapArgument(typeOfArgs[4], self, x5);
                if (numberOfArgs == 5) break;
                arguments[5] = wrapArgument(typeOfArgs[5], self, x6);
                if (numberOfArgs == 6) break;
                arguments[6] = wrapArgument(typeOfArgs[6], self, x7);
                if (numberOfArgs == 7) break;

                for (int i = 7; i < numberOfArgs; i++) {
                    byte[] argsInStack = EpicNative.get(sp + i * 8 + 8, 8);
                    arguments[i] = wrapArgument(typeOfArgs[i], self, argsInStack);
                }
            } while (false);

        } else {

            receiver = EpicNative.getObject(self, x1);
            //Logger.i(TAG, "this :" + receiver);

            do {
                if (numberOfArgs == 0) break;
                arguments[0] = wrapArgument(typeOfArgs[0], self, r2);
                if (numberOfArgs == 1) break;
                arguments[1] = wrapArgument(typeOfArgs[1], self, r3);
                if (numberOfArgs == 2) break;
                arguments[2] = wrapArgument(typeOfArgs[2], self, x4);
                if (numberOfArgs == 3) break;
                arguments[3] = wrapArgument(typeOfArgs[3], self, x5);
                if (numberOfArgs == 4) break;
                arguments[4] = wrapArgument(typeOfArgs[4], self, x6);
                if (numberOfArgs == 5) break;
                arguments[5] = wrapArgument(typeOfArgs[5], self, x7);
                if (numberOfArgs == 6) break;

                for (int i = 6; i < numberOfArgs; i++) {
                    byte[] argsInStack = EpicNative.get(sp + i * 8 + 16, 8);
                    arguments[i] = wrapArgument(typeOfArgs[i], self, argsInStack);
                }
            } while (false);
        }

        Logger.i(TAG, "arguments:" + Arrays.toString(arguments));

        Class<?> returnType = originMethodInfo.returnType;
        Object artMethod = originMethodInfo.method;

        Logger.d(TAG, "leave bridge function");

        if (returnType == void.class) {
            onHookVoid(artMethod, receiver, arguments);
            return 0;
        } else if (returnType == char.class) {
            return onHookChar(artMethod, receiver, arguments);
        } else if (returnType == byte.class) {
            return onHookByte(artMethod, receiver, arguments);
        } else if (returnType == short.class) {
            return onHookShort(artMethod, receiver, arguments);
        } else if (returnType == int.class) {
            return onHookInt(artMethod, receiver, arguments);
        } else if (returnType == long.class) {
            return onHookLong(artMethod, receiver, arguments);
        } else if (returnType == float.class) {
            return onHookFloat(artMethod, receiver, arguments);
        } else if (returnType == double.class) {
            return onHookDouble(artMethod, receiver, arguments);
        } else if (returnType == boolean.class) {
            return onHookBoolean(artMethod, receiver, arguments);
        } else {
            return onHookObject(artMethod, receiver, arguments);
        }
    }
```

- handleHookedArtMethod

1.环境准备
	获取挂载在当前方法上的callback
2.调用before method回调
	逐一调用before method调用(中间可设置拦截标记)
3.调用原始方法
	如果#2中已经设置拦截编辑则跳过执行，否则会执行代码逻辑。
4.调用after method回调
	对#2的调用进行成对执行，调用了哪些before method就调用多少after method回调
	
```java

public static Object handleHookedArtMethod(Object artMethodObject, Object thisObject, Object[] args) {
		// 环境准备
		CopyOnWriteSortedSet<XC_MethodHook> callbacks;

		ArtMethod artmethod = (ArtMethod ) artMethodObject;
		synchronized (hookedMethodCallbacks) {
			callbacks = hookedMethodCallbacks.get(artmethod.getExecutable());
		}
		Object[] callbacksSnapshot = callbacks.getSnapshot();
		final int callbacksLength = callbacksSnapshot.length;
		//Logger.d(TAG, "callbacksLength:" + callbacksLength +  ", this:" + thisObject + ", args:" + Arrays.toString(args));
		// 特殊case,无callback注册，直接调用原始方法
		if (callbacksLength == 0) {
			try {
				ArtMethod method = Epic.getBackMethod(artmethod);
				return method.invoke(thisObject, args);
			} catch (Exception e) {
				log(e.getCause());
			}
		}

		XC_MethodHook.MethodHookParam param = new XC_MethodHook.MethodHookParam();
		param.method  = (Member) (artmethod).getExecutable();
		param.thisObject = thisObject;
		param.args = args;

		// call "before method" callbacks
		// 调用before method回调
		int beforeIdx = 0;
		do {
			try {
				((XC_MethodHook) callbacksSnapshot[beforeIdx]).beforeHookedMethod(param);
			} catch (Throwable t) {
				log(t);

				// reset result (ignoring what the unexpectedly exiting callback did)
				param.setResult(null);
				param.returnEarly = false;
				continue;
			}

			if (param.returnEarly) {
				// skip remaining "before" callbacks and corresponding "after" callbacks
				beforeIdx++;
				break;
			}
		} while (++beforeIdx < callbacksLength);

		// call original method if not requested otherwise
		// 如果没有拦截原始方法调用，则调用原始方法
		if (!param.returnEarly) {
			try {
				ArtMethod method = Epic.getBackMethod(artmethod);
				Object result = method.invoke(thisObject, args);
				param.setResult(result);
			} catch (Exception e) {
				// log(e); origin throw exception is normal.
				param.setThrowable(e);
			}
		}

		// call "after method" callbacks
		// 调用after method回掉
		int afterIdx = beforeIdx - 1;
		do {
			Object lastResult =  param.getResult();
			Throwable lastThrowable = param.getThrowable();

			try {
				((XC_MethodHook) callbacksSnapshot[afterIdx]).afterHookedMethod(param);
			} catch (Throwable t) {
				DexposedBridge.log(t);

				// reset to last result (ignoring what the unexpectedly exiting callback did)
				if (lastThrowable == null)
					param.setResult(lastResult);
				else
					param.setThrowable(lastThrowable);
			}
		} while (--afterIdx >= 0);

		// 异常处理 & 返回结果。
		if (param.hasThrowable()) {
			final Throwable throwable = param.getThrowable();
			if (throwable instanceof IllegalAccessException || throwable instanceof InvocationTargetException
					|| throwable instanceof InstantiationException) {
				// reflect exception, get the origin cause
				final Throwable cause = throwable.getCause();

				// We can not change the exception flow of origin call, rethrow
				// Logger.e(TAG, "origin call throw exception (not a real crash, just record for debug):", cause);
				DexposedBridge.<RuntimeException>throwNoCheck(param.getThrowable().getCause(), null);
				return null; //never reach.
			} else {
				// the exception cause by epic self, just log.
				Logger.e(TAG, "epic cause exception in call bridge!!", throwable);
			}
			return null; // never reached.
		} else {
			final Object result = param.getResult();
			//Logger.d(TAG, "return :" + result);
			return result;
		}
	}

```



# Q&A


## 怎么获取jit/aot代码片段大小？

在上述[createTrampoline](#createTrampoline)中提到过由于一层跳板需要占据4个指令大小，因此其存在一个策略：
- 如果方法的code大小小于一层跳板大小：直接修改quick_compile_code指向
- 如果方法的code大小大于等于一层跳板大小：通过修改修改原始quick_compile_code的前4个指令。


那么问题来了，我们怎么知晓某个jit/aot方法的大小多少呢？

有办法的兄弟！我们aot/jit方法在内存中的布局为OatQuickMethodHeader，其中二进制代码是存在code_中，因此根据下图可知，code\_ - 4位置就是代码的长度了～

![image-20251123142332044](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20251123142332044.png)

注上图来源于——深入理解Android： Java虚拟机 9.6.1 OAT文件格式 图9-37　Oat文件格式各区域详细内容



当然我们也可以从[源码层面](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r1:art/runtime/oat_quick_method_header.h?q=function:GetCodeSize)上确认这件事情(android11源码) （epic就是当前版本实现的codesize获取）

```c++
class PACKED(4) OatQuickMethodHeader { 

   // ......

   uint32_t GetCodeSize() const {
    // ART compiled method are prefixed with header, but we can also easily
    // accidentally use a function pointer to one of the stubs/trampolines.
    // We prefix those with 0xFF in the aseembly so that we can do DCHECKs.
    CHECK_NE(code_size_, 0xFFFFFFFF) << code_size_;
    return code_size_ & kCodeSizeMask;
   }
    
  // ......

  // The offset in bytes from the start of the vmap table to the end of the header.
  uint32_t vmap_table_offset_ = 0u;
  // The code size in bytes. The highest bit is used to signify if the compiled
  // code with the method header has should_deoptimize flag.
  uint32_t code_size_ = 0u;
  // The actual code.
  uint8_t code_[0];
};


```

不过呢在android12以后OatQuickMethodHeader的格式有所变化，不过呢可以参考GetCodeSize兼容下

```c++
class PACKED(4) OatQuickMethodHeader {
 
// 获取代码长度
ALWAYS_INLINE uint32_t GetCodeSize() const {
    return LIKELY(IsOptimized())
        ? CodeInfo::DecodeCodeSize(GetOptimizedCodeInfoPtr())
        : (data_ & kCodeSizeMask);
}
  
  
  uint32_t data_ = 0u;  // Combination of fields using the above masks.
  uint8_t code_[0];     // The actual method code.
};
```

Note: 另外再提一嘴，android14以后，OatQuickMethodHeader又又又被改了，不过依然可以参考OatQuickMethodHeader.GetCodeSize兼容适配，这里就不浪费篇幅解释了。



## 参数传递设计？



假如我们需要对方法进行hook,那么可能会调用外部的函数，那么这个调用传参是如何设计的呢？为什么这样设计呢？

- jit/aot传参规则说明

android对于jit/aot的二进制方法的传参如下：

x0~x7传递前8的参数，超过8个参数的部分会通过sp堆栈传递。

具体可见[源码](https://cs.android.com/android/platform/superproject/+/android-latest-release:art/runtime/arch/arm64/quick_entrypoints_arm64.S;l=696?q=art_quick_invoke_static_stub&ss=android%2Fplatform%2Fsuperproject)

```asm
ENTRY art_quick_invoke_static_stub
    // Spill registers as per AACPS64 calling convention.
    // 堆栈准备，此处会将>8的参数内容拷贝到sp中
    INVOKE_STUB_CREATE_FRAME

    // Load args into registers.
    // 加载填充x0~x7寄存器。
    INVOKE_STUB_LOAD_ALL_ARGS _static

    // Call the method and return.
    INVOKE_STUB_CALL_AND_RETURN
END art_quick_invoke_static_stub

```

Note：此处只挑了static方法作为示例，普通的方法调用可见art_quick_invoke_stub，这里就不贴了。

- epic传参逻辑

epic为了防止在hook函数调用的过程中**产生新的堆栈**，是通过malloc内存，再通过寄存器的方式传递malloc内存的方式传递额外的参数。

具体可见[Weishu大佬博客-bridge函数分发以及堆栈平衡章节](https://weishu.me/2017/11/23/dexposed-on-art/)

而关于hook函数具体的传参数目和类型可见[segments跳板分析](#segments跳板)，不难发现有如下改动：

Hook前：x0~x7 均为函数参数

rHook后：

x0 targetArtMethod对象地址

x1 首个参数(如果不是static方法可能是this对象的地址)

x2 c++ thread对象地址

x3 结构体地址(包含sp, x2,x3,原始的ArtMethod地址。)

x4~x7 函数参数

```asm

0000000000000000 <.data>:
   // 指令对齐
   0:   d503201f        nop
   // 1. 判断"原方法"和"当前执行的方法"是否是同一个（确保没有hook 不应该hook的方法）
   // 加载需要hook的artMethod内存地址
   4:   58000269        ldr     x9, 0x50
   // 对比当前的artMethod和需要hook的artMethod是否一致
   8:   eb09001f        cmp     x0, x9
   // 如果不一致跳转到0x60处，0x60只是当前case的一个跳转点
   //  (其实这个是相对pc跳转，0x60的位置正好是当前跳板的结束位置，是下一个hook函数的起始位置。)
   c:   540002a1        b.ne    0x60  // b.any
   // 2. 保存准备方法调用参数x0、x2、x3
   // 将target_method_address加载到x0中
  10:   58000180        ldr     x0, 0x40
  // 将结构体地址加载到x9中
  14:   58000229        ldr     x9, 0x58
  // 将sp设置到x10中
  18:   910003ea        mov     x10, sp
  // 将sp存放在结构体[0,8]位置
  1c:   f900012a        str     x10, [x9]
  // 将x2保存到结构体[8,16]的位置
  20:   f9000522        str     x2, [x9, #8]
  // 将x3保存到结构体[16,26]的位置
  24:   f9000923        str     x3, [x9, #16]
  // 将x3设置为结构体地址
  28:   aa0903e3        mov     x3, x9
  // 将source_method保存到x2中
  2c:   58000122        ldr     x2, 0x50
  // 将source_method保存到结构体[24,32]位置
  30:   f9000d22        str     x2, [x9, #24]
  // 将x19(Thread::Current)的值保存到x2中
  34:   aa1303e2        mov     x2, x19
  // 加载target_method_address到x9中
  38:   58000089        ldr     x9, 0x48
  // 跳转到target_method_address中去
  // x0 桥接方法的artMethod对象地址
  // x1 第一个参数
  // x2 Thread::Current
  // x3 结构体地址[sp,x2,x3,方法原始ArtMethod]
  // x4 ~ x7 原始传参类型
  3c:   d61f0120        br      x9
  40    00 00 00 00 00 00 00 00 - target_method_address 
  48    00 00 00 00 00 00 00 00 - target_method_entry
  50    00 00 00 00 00 00 00 00 - source_method
  58    00 00 00 00 00 00 00 00 - struct
  60    ......
```

Note: Art的方法寄存器传参说明如下：(系统源码可自行分析art_quick_invoke_static_stub/art_quick_invoke_stub逻辑)

x0 - 需要执行的artMethod的地址

x1 - 如果是static方法则是第一个参数的值，如果是实例方法就是this对象的地址。

x2 ～ x7 函数参数。

- 桥接函数 

上述br x9以后的会调用方法XXXBridge(根据返回值类型确定)，可以桥接方法的定义和原始方法对齐的。

```java
    private static void voidBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
         referenceBridge(r1, self, struct, x4, x5, x6, x7);
    }

    private static boolean booleanBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
        return (Boolean) referenceBridge(r1, self, struct, x4, x5, x6, x7);
    }

    private static byte byteBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
        return (Byte) referenceBridge(r1, self, struct, x4, x5, x6, x7);
    }

    private static short shortBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
        return (Short) referenceBridge(r1, self, struct, x4, x5, x6, x7);
    }

    private static char charBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
        return (Character) referenceBridge(r1, self, struct, x4, x5, x6, x7);
    }

    private static int intBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
        return (Integer) referenceBridge(r1, self, struct, x4, x5, x6, x7);
    }

    private static long longBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
        return (Long) referenceBridge(r1, self, struct, x4, x5, x6, x7);
    }

    private static float floatBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
        return (Float) referenceBridge(r1, self, struct, x4, x5, x6, x7);
    }

    private static double doubleBridge(long r1, long self, long struct, long x4, long x5, long x6, long x7) {
        return (Double) referenceBridge(r1, self, struct, x4, x5, x6, x7);
    }
```



到这我们做个总结,回答下前面提到的两个问题：

1.Hook方法传参原理？

epic的hook方法调用传参数是通过malloc内存，然后通过将额外的参数通过写入malloc内存的方式进行保存，并通过将这个内存的起始地址传递到x3寄存器中进行传递

2.为什么会这样实现？

其实[weishu大佬的博客中也提到了](https://weishu.me/2017/11/23/dexposed-on-art/#:~:text=%E4%BA%8B%E5%AE%9E%E8%AF%81%E6%98%8E%E8%BF%99%E6%A0%B7%E5%81%9A%E6%98%AF%E4%B8%8D%E8%A1%8C%E7%9A%84%EF%BC%8C%E5%A6%82%E6%9E%9C%E6%88%91%E4%BB%AC%E5%9C%A8%E4%BA%8C%E6%AE%B5%E8%B7%B3%E6%9D%BF%E4%BB%A3%E7%A0%81%E9%87%8C%E9%9D%A2%E5%BC%80%E8%BE%9F%E5%A0%86%E6%A0%88%EF%BC%8C%E8%BF%9B%E8%80%8C%E4%BF%AE%E6%94%B9%E4%BA%86sp%E5%AF%84%E5%AD%98%E5%99%A8%EF%BC%9B%E9%82%A3%E4%B9%88%E5%9C%A8%E6%88%91%E4%BB%AC%E4%BF%AE%E6%94%B9sp%E5%88%B0%E8%B0%83%E7%94%A8bridge%E5%87%BD%E6%95%B0%E7%9A%84%E8%BF%99%E6%AE%B5%E6%97%B6%E9%97%B4%E9%87%8C%EF%BC%8C%E5%A0%86%E6%A0%88%E7%BB%93%E6%9E%84%E4%B8%8E%E4%B8%8DHook%E7%9A%84%E6%97%B6%E5%80%99%E6%98%AF%E4%B8%8D%E4%B8%80%E6%A0%B7%E7%9A%84%EF%BC%88%E8%99%BD%E7%84%B6bridge%E5%87%BD%E6%95%B0%E6%89%A7%E8%A1%8C%E5%AE%8C%E6%AF%95%E4%B9%8B%E5%90%8E%E6%88%91%E4%BB%AC%E5%8F%AF%E4%BB%A5%E6%81%A2%E5%A4%8D%E6%AD%A3%E5%B8%B8%EF%BC%89%EF%BC%9B%E5%9C%A8%E8%BF%99%E6%AE%B5%E6%97%B6%E9%97%B4%E9%87%8C%E5%A6%82%E6%9E%9C%E8%99%9A%E6%8B%9F%E6%9C%BA%E9%9C%80%E8%A6%81%E8%BF%9B%E8%A1%8C%E6%A0%88%E5%9B%9E%E6%BA%AF%EF%BC%8Csp%E8%A2%AB%E4%BF%AE%E6%94%B9%E7%9A%84%E9%82%A3%E4%B8%80%E5%B8%A7%E4%BC%9A%E7%94%B1%E4%BA%8E%E5%9B%9E%E6%BA%AF%E4%B8%8D%E5%88%B0%E5%AF%B9%E5%BA%94%E7%9A%84%E5%87%BD%E6%95%B0%E5%BC%95%E5%8F%91%E8%87%B4%E5%91%BD%E9%94%99%E8%AF%AF%EF%BC%8C%E5%AF%BC%E8%87%B4Runtime%20%E7%9B%B4%E6%8E%A5Abort%E3%80%82%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E4%BC%9A%E5%9B%9E%E6%BA%AF%E5%A0%86%E6%A0%88%EF%BC%9F%E5%8F%91%E7%94%9F%E5%BC%82%E5%B8%B8%E6%88%96%E8%80%85GC%E7%9A%84%E6%97%B6%E5%80%99%E3%80%82%E6%9C%80%E7%9B%B4%E8%A7%82%E7%9A%84%E6%84%9F%E5%8F%97%E5%B0%B1%E6%98%AF%EF%BC%8C%E5%A6%82%E6%9E%9Cbridge%E5%87%BD%E6%95%B0%E9%87%8C%E9%9D%A2%E6%9C%89%E4%BB%BB%E4%BD%95%E5%BC%82%E5%B8%B8%E6%8A%9B%E5%87%BA%EF%BC%88%E5%8D%B3%E4%BD%BF%E8%A2%ABtry..catch%E4%BD%8F%EF%BC%89%E5%B0%B1%E4%BC%9A%E4%BD%BF%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9B%B4%E6%8E%A5%E5%B4%A9%E6%BA%83%E3%80%82dexposed%E7%9A%84%20dev_art%20%E5%88%86%E6%94%AF%E4%B8%AD%E7%9A%84AOP%E5%AE%9E%E7%8E%B0%E5%B0%B1%E6%9C%89%E8%BF%99%E4%B8%AA%E9%97%AE%E9%A2%98%E3%80%82)，我们额外的参数最容易想到的就是通过开辟新的堆栈实现传递，但是这存在问题，因为在一些特殊情况下art会扫描堆栈解析方法调用，因此可能会发生崩溃，通过malloc传递这样可以确保对方法堆栈无影响。






# refs

[Weishu's Notes](https://weishu.me/2017/11/23/dexposed-on-art/)

