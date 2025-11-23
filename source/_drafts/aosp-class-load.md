---
title: aosp-class-load
tags:
cover:
---



# Android 类加载流程


[续ClassLoader类加载流程](./clasloader-process)




# 关于本文


主要是类定义/创建过程进行分析


基于android-14.0.0_r73


# 起点

主要是对类加载/定义的触发时机做分析和梳理


总结一句话就是：类加载可能是由java层触发/native层触发，但是无论是哪一层触发，都会check当前类是否已经加载过，加载过就直接返回缓存，没加载过都会调用DefineClass对类进行定义。


## DexFile_defineClassNative


DexFile_defineClassNative光从函数名称看就知道是一个JNI方法，对应于Java中的DexFile.defineClassNative方法。通常情况下的调用路径如下：
BaseDexClassLoader#findClass -> 
	DexPathList#findClass -> 
		DexPathList.Element#findClass ->
			DexFile#loadClassBinaryName ->
				DexFile#defineClass ->
					DexFile#defineClassNative ->
						DexFile_defineClassNative



接着我们简单分析下DexFile_defineClassNative的执行流程。逻辑如下：
1.寻找包含需要定义的类的dexFile对象
2.调用DefineClass定义类型
3.将类型插入到classloader缓存中
```c++

static jclass DexFile_defineClassNative(JNIEnv* env,
                                        jclass,
                                        jstring javaName,
                                        jobject javaLoader,
                                        jobject cookie,
                                        jobject dexFile) {
  // #1 加载dexFiles到dex_files中                                      
  std::vector<const DexFile*> dex_files;
  const OatFile* oat_file;
  if (!ConvertJavaArrayToDexFiles(env, cookie, /*out*/ dex_files, /*out*/ oat_file)) {
    // 加载失败直接返回！！
    VLOG(class_linker) << "Failed to find dex_file";
    DCHECK(env->ExceptionCheck());
    return nullptr;
  }

  // 过滤异常case: 加载的类名称为null.
  ScopedUtfChars class_name(env, javaName);
  if (class_name.c_str() == nullptr) {
    VLOG(class_linker) << "Failed to find class_name";
    return nullptr;
  }
  // 获取descriptor & hash值(后面获取classDef有用)
  const std::string descriptor(DotToDescriptor(class_name.c_str()));
  const size_t hash(ComputeModifiedUtf8Hash(descriptor.c_str()));
  // ＃2 遍历所有dex，获取class对象
  for (auto& dex_file : dex_files) {
  // #2.1 通过descriptor & hash获取classDef
    const dex::ClassDef* dex_class_def =
        OatDexFile::FindClassDef(*dex_file, descriptor.c_str(), hash);
    if (dex_class_def != nullptr) {
      ScopedObjectAccess soa(env);
      ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
      StackHandleScope<1> hs(soa.Self());
      // 获取对于的classloade对象
      Handle<mirror::ClassLoader> class_loader(
          hs.NewHandle(soa.Decode<mirror::ClassLoader>(javaLoader)));
      // 获取dexFile对应的dexCache对象(如果没有会自动关联)
      ObjPtr<mirror::DexCache> dex_cache =
          class_linker->RegisterDexFile(*dex_file, class_loader.Get());
      // fail-fast
      if (dex_cache == nullptr) {
        // OOME or InternalError (dexFile already registered with a different class loader).
        soa.Self()->AssertPendingException();
        return nullptr;
      }
      // #2.2 定义Class
      ObjPtr<mirror::Class> result = class_linker->DefineClass(soa.Self(),
                                                               descriptor.c_str(),
                                                               hash,
                                                               class_loader,
                                                               *dex_file,
                                                               *dex_class_def);
      // Add the used dex file. This only required for the DexFile.loadClass API since normal
      // class loaders already keep their dex files live.
      // #2.3 将Class插入到ClassLoader中去
      class_linker->InsertDexFileInToClassLoader(soa.Decode<mirror::Object>(dexFile),
                                                 class_loader.Get());
      // ＃2.4 返回结果
      if (result != nullptr) {
        VLOG(class_linker) << "DexFile_defineClassNative returning " << result
                           << " for " << class_name.c_str();
        return soa.AddLocalReference<jclass>(result);
      }
    }
  }
  VLOG(class_linker) << "Failed to find dex_class_def " << class_name.c_str();
  return nullptr;
}
```




## ClassLinker::FindClass


调用路径这里就不分析了，上篇文章已经分析过了，简单来说就是loadClass会一层层调用到该方法.
我们直接看FindClass里面做了啥
```c++
ObjPtr<mirror::Class> ClassLinker::FindClass(Thread* self,
                                             const char* descriptor,
                                             Handle<mirror::ClassLoader> class_loader) {
    
  // debug下的check逻辑，ignore  
  DCHECK_NE(*descriptor, '\0') << "descriptor is empty string";
  DCHECK(self != nullptr);
  // 确保无pending Exception标记 
  self->AssertNoPendingException();
  self->PoisonObjectPointers();  // For DefineClass, CreateArrayClass, etc...
  // 寻找基本类型
    if (descriptor[1] == '\0') {
    // only the descriptors of primitive types should be 1 character long, also avoid class lookup
    // for primitive classes that aren't backed by dex files.
    return FindPrimitiveClass(descriptor[0]);
  }
  const size_t hash = ComputeModifiedUtf8Hash(descriptor);
  // Find the class in the loaded classes table.
  // 从已加载的类缓存中寻找 
  ObjPtr<mirror::Class> klass = LookupClass(self, descriptor, hash, class_loader.Get());
  if (klass != nullptr) {
    return EnsureResolved(self, descriptor, klass);
  }
  // Class is not yet loaded.
  // 如果类型还没有加载并且class_loader为null直接使用boot_class_loader进行加载
  if (descriptor[0] != '[' && class_loader == nullptr) {
    // Non-array class and the boot class loader, search the boot class path.
    // boot class path中如果
    ClassPathEntry pair = FindInClassPath(descriptor, hash, boot_class_path_);
    if (pair.second != nullptr) { 
      return DefineClass(self,
                         descriptor,
                         hash,
                         ScopedNullHandle<mirror::ClassLoader>(),
                         *pair.first,
                         *pair.second);
    } else {
      // The boot class loader is searched ahead of the application class loader, failures are
      // expected and will be wrapped in a ClassNotFoundException. Use the pre-allocated error to
      // trigger the chaining with a proper stack trace.
      ObjPtr<mirror::Throwable> pre_allocated =
          Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
      self->SetException(pre_allocated);
      return nullptr;
    }
  }
    
}

```



## ClassLinker::FindClassInBaseDexClassLoaderClassPath

 

## ClassLinker::FindClassInBootClassLoaderClassPath









# ClassLinker::DefineClass


如下是DefineClass的完整代码。看着挺长的，接下来我们会对这个方法进行拆解逐一进行分析。

```c++

ObjPtr<mirror::Class> ClassLinker::DefineClass(Thread* self,
                                               const char* descriptor,
                                               size_t hash,
                                               Handle<mirror::ClassLoader> class_loader,
                                               const DexFile& dex_file,
                                               const dex::ClassDef& dex_class_def) {
  ScopedDefiningClass sdc(self);
  StackHandleScope<3> hs(self);
  metrics::AutoTimer timer{GetMetrics()->ClassLoadingTotalTime()};
  metrics::AutoTimer timeDelta{GetMetrics()->ClassLoadingTotalTimeDelta()};
  auto klass = hs.NewHandle<mirror::Class>(nullptr);

  // Load the class from the dex file.
  // 处理虚拟机未init完成的case
  if (UNLIKELY(!init_done_)) {
    // finish up init of hand crafted class_roots_
    if (strcmp(descriptor, "Ljava/lang/Object;") == 0) {
      klass.Assign(GetClassRoot<mirror::Object>(this));
    } else if (strcmp(descriptor, "Ljava/lang/Class;") == 0) {
      klass.Assign(GetClassRoot<mirror::Class>(this));
    } else if (strcmp(descriptor, "Ljava/lang/String;") == 0) {
      klass.Assign(GetClassRoot<mirror::String>(this));
    } else if (strcmp(descriptor, "Ljava/lang/ref/Reference;") == 0) {
      klass.Assign(GetClassRoot<mirror::Reference>(this));
    } else if (strcmp(descriptor, "Ljava/lang/DexCache;") == 0) {
      klass.Assign(GetClassRoot<mirror::DexCache>(this));
    } else if (strcmp(descriptor, "Ldalvik/system/ClassExt;") == 0) {
      klass.Assign(GetClassRoot<mirror::ClassExt>(this));
    }
  }

  // For AOT-compilation of an app, we may use only a public SDK to resolve symbols. If the SDK
  // checks are configured (a non null SdkChecker) and the descriptor is not in the provided
  // public class path then we prevent the definition of the class.
  //
  // NOTE that we only do the checks for the boot classpath APIs. Anything else, like the app
  // classpath is not checked.
  if (class_loader == nullptr &&
      Runtime::Current()->IsAotCompiler() &&
      DenyAccessBasedOnPublicSdk(descriptor)) {
    ObjPtr<mirror::Throwable> pre_allocated =
        Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
    self->SetException(pre_allocated);
    return sdc.Finish(nullptr);
  }

  // This is to prevent the calls to ClassLoad and ClassPrepare which can cause java/user-supplied
  // code to be executed. We put it up here so we can avoid all the allocations associated with
  // creating the class. This can happen with (eg) jit threads.
  if (!self->CanLoadClasses()) {
    // Make sure we don't try to load anything, potentially causing an infinite loop.
    ObjPtr<mirror::Throwable> pre_allocated =
        Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
    self->SetException(pre_allocated);
    return sdc.Finish(nullptr);
  }

  ScopedTrace trace(descriptor);
  if (klass == nullptr) {
    // Allocate a class with the status of not ready.
    // Interface object should get the right size here. Regular class will
    // figure out the right size later and be replaced with one of the right
    // size when the class becomes resolved.
    if (CanAllocClass()) {
      klass.Assign(AllocClass(self, SizeOfClassWithoutEmbeddedTables(dex_file, dex_class_def)));
    } else {
      return sdc.Finish(nullptr);
    }
  }
  if (UNLIKELY(klass == nullptr)) {
    self->AssertPendingOOMException();
    return sdc.Finish(nullptr);
  }
  // Get the real dex file. This will return the input if there aren't any callbacks or they do
  // nothing.
  DexFile const* new_dex_file = nullptr;
  dex::ClassDef const* new_class_def = nullptr;
  // TODO We should ideally figure out some way to move this after we get a lock on the klass so it
  // will only be called once.
  Runtime::Current()->GetRuntimeCallbacks()->ClassPreDefine(descriptor,
                                                            klass,
                                                            class_loader,
                                                            dex_file,
                                                            dex_class_def,
                                                            &new_dex_file,
                                                            &new_class_def);
  // Check to see if an exception happened during runtime callbacks. Return if so.
  if (self->IsExceptionPending()) {
    return sdc.Finish(nullptr);
  }
  ObjPtr<mirror::DexCache> dex_cache = RegisterDexFile(*new_dex_file, class_loader.Get());
  if (dex_cache == nullptr) {
    self->AssertPendingException();
    return sdc.Finish(nullptr);
  }
  klass->SetDexCache(dex_cache);
  SetupClass(*new_dex_file, *new_class_def, klass, class_loader.Get());

  // Mark the string class by setting its access flag.
  if (UNLIKELY(!init_done_)) {
    if (strcmp(descriptor, "Ljava/lang/String;") == 0) {
      klass->SetStringClass();
    }
  }

  ObjectLock<mirror::Class> lock(self, klass);
  klass->SetClinitThreadId(self->GetTid());
  // Make sure we have a valid empty iftable even if there are errors.
  klass->SetIfTable(GetClassRoot<mirror::Object>(this)->GetIfTable());

  // Add the newly loaded class to the loaded classes table.
  ObjPtr<mirror::Class> existing = InsertClass(descriptor, klass.Get(), hash);
  if (existing != nullptr) {
    // We failed to insert because we raced with another thread. Calling EnsureResolved may cause
    // this thread to block.
    return sdc.Finish(EnsureResolved(self, descriptor, existing));
  }

  // Load the fields and other things after we are inserted in the table. This is so that we don't
  // end up allocating unfree-able linear alloc resources and then lose the race condition. The
  // other reason is that the field roots are only visited from the class table. So we need to be
  // inserted before we allocate / fill in these fields.
  LoadClass(self, *new_dex_file, *new_class_def, klass);
  if (self->IsExceptionPending()) {
    VLOG(class_linker) << self->GetException()->Dump();
    // An exception occured during load, set status to erroneous while holding klass' lock in case
    // notification is necessary.
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
    }
    return sdc.Finish(nullptr);
  }

  // Finish loading (if necessary) by finding parents
  CHECK(!klass->IsLoaded());
  if (!LoadSuperAndInterfaces(klass, *new_dex_file)) {
    // Loading failed.
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
    }
    return sdc.Finish(nullptr);
  }
  CHECK(klass->IsLoaded());

  // At this point the class is loaded. Publish a ClassLoad event.
  // Note: this may be a temporary class. It is a listener's responsibility to handle this.
  Runtime::Current()->GetRuntimeCallbacks()->ClassLoad(klass);

  // Link the class (if necessary)
  CHECK(!klass->IsResolved());
  // TODO: Use fast jobjects?
  auto interfaces = hs.NewHandle<mirror::ObjectArray<mirror::Class>>(nullptr);

  MutableHandle<mirror::Class> h_new_class = hs.NewHandle<mirror::Class>(nullptr);
  if (!LinkClass(self, descriptor, klass, interfaces, &h_new_class)) {
    // Linking failed.
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
    }
    return sdc.Finish(nullptr);
  }
  self->AssertNoPendingException();
  CHECK(h_new_class != nullptr) << descriptor;
  CHECK(h_new_class->IsResolved()) << descriptor << " " << h_new_class->GetStatus();

  // Instrumentation may have updated entrypoints for all methods of all
  // classes. However it could not update methods of this class while we
  // were loading it. Now the class is resolved, we can update entrypoints
  // as required by instrumentation.
  if (Runtime::Current()->GetInstrumentation()->EntryExitStubsInstalled()) {
    // We must be in the kRunnable state to prevent instrumentation from
    // suspending all threads to update entrypoints while we are doing it
    // for this class.
    DCHECK_EQ(self->GetState(), ThreadState::kRunnable);
    Runtime::Current()->GetInstrumentation()->InstallStubsForClass(h_new_class.Get());
  }

  /*
   * We send CLASS_PREPARE events to the debugger from here.  The
   * definition of "preparation" is creating the static fields for a
   * class and initializing them to the standard default values, but not
   * executing any code (that comes later, during "initialization").
   *
   * We did the static preparation in LinkClass.
   *
   * The class has been prepared and resolved but possibly not yet verified
   * at this point.
   */
  Runtime::Current()->GetRuntimeCallbacks()->ClassPrepare(klass, h_new_class);

  // Notify native debugger of the new class and its layout.
  jit::Jit::NewTypeLoadedIfUsingJit(h_new_class.Get());

  return sdc.Finish(h_new_class);
}

```




## 类初始化与预检查


执行逻辑如下：

- **特殊类处理**
	若 `init_done_`未完成（ART 初始化阶段），直接返回预定义的 `ClassRoot`对象（如 `java.lang.Object`、`java.lang.String`等）
- **AOT 编译安全检查**
	若类加载器为空且处于 AOT 编译模式，检查类是否在公共 SDK 路径中，否则抛出 `NoClassDefFoundError`
- **线程权限检查**
	若当前线程无法加载类（如 JIT 线程限制），抛出异常
通过代码的分析可以发现主要是一些系统线程/进程的过滤，除开”特殊类处理“以外其余两个逻辑不影响普通应用线程。

```c++
ScopedDefiningClass sdc(self);
StackHandleScope<3> hs(self);
metrics::AutoTimer timer{GetMetrics()->ClassLoadingTotalTime()};
metrics::AutoTimer timeDelta{GetMetrics()->ClassLoadingTotalTimeDelta()};
auto klass = hs.NewHandle<mirror::Class>(nullptr);

// Load the class from the dex file.
// #1 若虚拟机未初始化完成，并且描述符是如下，则直接获取class返回(可能已经提前加载好了吧？)
if (UNLIKELY(!init_done_)) {
	// finish up init of hand crafted class_roots_
	if (strcmp(descriptor, "Ljava/lang/Object;") == 0) {
	  klass.Assign(GetClassRoot<mirror::Object>(this));
	} else if (strcmp(descriptor, "Ljava/lang/Class;") == 0) {
	  klass.Assign(GetClassRoot<mirror::Class>(this));
	} else if (strcmp(descriptor, "Ljava/lang/String;") == 0) {
	  klass.Assign(GetClassRoot<mirror::String>(this));
	} else if (strcmp(descriptor, "Ljava/lang/ref/Reference;") == 0) {
	  klass.Assign(GetClassRoot<mirror::Reference>(this));
	} else if (strcmp(descriptor, "Ljava/lang/DexCache;") == 0) {
	  klass.Assign(GetClassRoot<mirror::DexCache>(this));
	} else if (strcmp(descriptor, "Ldalvik/system/ClassExt;") == 0) {
	  klass.Assign(GetClassRoot<mirror::ClassExt>(this));
	}
}

// For AOT-compilation of an app, we may use only a public SDK to resolve symbols. If the SDK
// checks are configured (a non null SdkChecker) and the descriptor is not in the provided
// public class path then we prevent the definition of the class.
//
// NOTE that we only do the checks for the boot classpath APIs. Anything else, like the app
// classpath is not checked.
// #2 针对于aot编译(dex2oat进程)进行安全检查，如果加载类非公开的类型直接抛NoClassDefFound异常。
// Note：DenyAccessBasedOnPublicSdk是一个virtual方法，具体实现在AotClassLinker::DenyAccessBasedOnPublicSdk中
if (class_loader == nullptr &&
  Runtime::Current()->IsAotCompiler() &&
  DenyAccessBasedOnPublicSdk(descriptor)) {
	ObjPtr<mirror::Throwable> pre_allocated =
    Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
	self->SetException(pre_allocated);
	return sdc.Finish(nullptr);
}

// This is to prevent the calls to ClassLoad and ClassPrepare which can cause java/user-supplied
// code to be executed. We put it up here so we can avoid all the allocations associated with
// creating the class. This can happen with (eg) jit threads.
// #3 阻止art线程加载类，只针对于debug包下的jit线程等系统线程(应用线程不受影响)
if (!self->CanLoadClasses()) {
	// Make sure we don't try to load anything, potentially causing an infinite loop.
	// 这样做是为了确保任何类都不要加载，从而规避无限循环的case（类依赖循环）
	ObjPtr<mirror::Throwable> pre_allocated =
    Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
	self->SetException(pre_allocated);
	return sdc.Finish(nullptr);
}
```


## 分配与注册 Class 对象


- **分配内存**：
    通过 `AllocClass`分配 `mirror::Class`对象内存，大小由 `SizeOfClassWithoutEmbeddedTables`计算
- **关联 DexCache**：
    调用 `RegisterDexFile`将 Dex 文件注册到类加载器的 `DexCache`，确保后续访问类信息可直接从缓存获取
- **设置类属性**：
    包括类描述符, 访问标志, classLoader, class索引, 类型索引等元数据

```c++
  // #1 预分配类对象内存大小
  // 对于接口类型的数据这里分配的大小就是正确的大小
  // 对于普通的class,这里分配的大小之包含static变量的大小，后续会在解析父类的时候进行一轮扩展。
  ScopedTrace trace(descriptor);
  if (klass == nullptr) {
    // Allocate a class with the status of not ready.
    // Interface object should get the right size here. Regular class will
    // figure out the right size later and be replaced with one of the right
    // size when the class becomes resolved.
    
    // 这里普通应用进程的默认为true, aot进程的CanAllocClass方法被重写咧～懒得分析咧～
    if (CanAllocClass()) {
	  // SizeOfClassWithoutEmbeddedTables会从dex中遍历类的staticFields计算空间占用
	  // AllocClass会从堆中分配特定的内存大小
      klass.Assign(AllocClass(self, SizeOfClassWithoutEmbeddedTables(dex_file, dex_class_def)));
    } else {
      return sdc.Finish(nullptr);
    }
  }
  if (UNLIKELY(klass == nullptr)) {
    self->AssertPendingOOMException();
    return sdc.Finish(nullptr);
  }

  // Get the real dex file. This will return the input if there aren't any callbacks or they do
  // nothing.
  // 获取real dex file, 此处将调用PreDefine回调对dex_file进行一层转换。
  // 如果此处没有添加任何callback,那么new_dex_file将会指向以往的dex文件。
  DexFile const* new_dex_file = nullptr;
  dex::ClassDef const* new_class_def = nullptr;
  // TODO We should ideally figure out some way to move this after we get a lock on the klass so it
  // will only be called once.
  Runtime::Current()->GetRuntimeCallbacks()->ClassPreDefine(descriptor,
                                                            klass,
                                                            class_loader,
                                                            dex_file,
                                                            dex_class_def,
                                                            &new_dex_file,
                                                            &new_class_def);
  // Check to see if an exception happened during runtime callbacks. Return if so.
  // fail-fast 异常处理
  if (self->IsExceptionPending()) {
    return sdc.Finish(nullptr);
  }
  // ＃2 关联DexCache对象
  // Note: 准确来说是将DexFile,ClassTable(ClassLoader缓存的所有类集合), DexCache这三者做绑定
  // 其中绑定关系存储在一个map里面，key为DexFile, value为ClassTable + DexCache. 具体逻辑就不展开了.....
  ObjPtr<mirror::DexCache> dex_cache = RegisterDexFile(*new_dex_file, class_loader.Get());
  // 如果关联失败，直接返回null.
  if (dex_cache == nullptr) {
    self->AssertPendingException();
    return sdc.Finish(nullptr);
  }
  
  // #3 设置类的元数据信息
  // 设置dexCache
  klass->SetDexCache(dex_cache);
  // 设置klass属性（Class是Object）
  // 设置accessFlags
  // 设置classLoader
  // 设置dexClassDefIndex
  // 设置dexTypeIndex 
  SetupClass(*new_dex_file, *new_class_def, klass, class_loader.Get());

  // Mark the string class by setting its access flag.
  // 设置虚拟机未启动完成时的String类型的accessFlags
  if (UNLIKELY(!init_done_)) {
    if (strcmp(descriptor, "Ljava/lang/String;") == 0) {
      klass->SetStringClass();
    }
  }

  // 获取锁，还记得我前面说的“Class也是Object”，这里获取的是object的synchronized锁
  ObjectLock<mirror::Class> lock(self, klass);
  // 设置clinitThreadId
  klass->SetClinitThreadId(self->GetTid());
  // 设置ifTable
  // Make sure we have a valid empty iftable even if there are errors.
  klass->SetIfTable(GetClassRoot<mirror::Object>(this)->GetIfTable());
```

## 加载类元数据


- **插入类表**：
    通过 `InsertClass`将类插入到类加载器的哈希表中，若冲突则触发 `EnsureResolved`处理重复加载
- **加载字段与方法**：
    调用 `LoadClass`解析 Dex 文件中的静态字段、实例字段及方法，填充 `ArtField`和 `ArtMethod`数组
    - **静态字段初始化**：分配内存并设置默认值（如 `0`、`null`）。
    - **方法入口点设置**：通过 `LinkCode`关联 JIT/AOT 编译后的机器码入口（如 `SetEntryPointFromQuickCompiledCode`）

```c++

  // #1 将新创建的class对象插入到classTable中去
  // Add the newly loaded class to the loaded classes table.
  ObjPtr<mirror::Class> existing = InsertClass(descriptor, klass.Get(), hash);
  if (existing != nullptr) { // 如果在插入前已经有class对象了，确保Class已经Resolve以后哦直接返回。以免重复加载类。
    // We failed to insert because we raced with another thread. Calling EnsureResolved may cause
    // this thread to block.
    return sdc.Finish(EnsureResolved(self, descriptor, existing));
  }

  // Load the fields and other things after we are inserted in the table. This is so that we don't
  // end up allocating unfree-able linear alloc resources and then lose the race condition. The
  // other reason is that the field roots are only visited from the class table. So we need to be
  // inserted before we allocate / fill in these fields.
  // #2 加载剩余的 static属性/成员属性/direct method/virtual method
  LoadClass(self, *new_dex_file, *new_class_def, klass);
  // fail-fast check 是否有异常设置，如有则直接将类状态字段设置为error并返回
  if (self->IsExceptionPending()) {
    VLOG(class_linker) << self->GetException()->Dump();
    // An exception occured during load, set status to erroneous while holding klass' lock in case
    // notification is necessary.
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
    }
    return sdc.Finish(nullptr);
  }
```

### LoadClass

由于LoadClass的过程比较长，因此对这个方法的执行逻辑拆开了分析下。从执行逻辑来看可以将其分为以下部分：
- **ArtField/ArtMethod 数组分配(对于于类属性/方法)***
	首先通过accessor获取dex中static field/instance field的数目，在通过调用AllocArtFieldArray/AllocArtFieldArray分配内存空间
- **填充ArtField/ArtMethod数组内容**
	通过调用accessor的VisitFieldsAndMethods方法使用访问者的模式对[class data item遍历](https://source.android.google.cn/docs/core/runtime/dex-format?hl=ur&authuser=1#class-data-item)，并填充ArtField/ArtMethod
	- static field —— 静态属性
	- instance field —— 成员属性
	- direct method —— 直接方法(`static`、`private` 或构造函数的任何一个)
	- virtual method —— 虚拟方法(非 `static`、`private` 或构造函数)
- **设置Class字段**
	主要是用来设置将，sfields(静态属性集合), ifields(实例属性集合)设置到class对象中。

```c++

void ClassLinker::LoadClass(Thread* self,
                            const DexFile& dex_file,
                            const dex::ClassDef& dex_class_def,
                            Handle<mirror::Class> klass) {
  // 类似于java的asm框架，如果我们要读取dex中的信息，也需要通过访问者的模式获取，而ClassAccessor就好比ASM中的ClassVisitor                          
  ClassAccessor accessor(dex_file,
                         dex_class_def,
                         /* parse_hiddenapi_class_data= */ klass->IsBootStrapClassLoaded());
  // fail-fast机制                     
  if (!accessor.HasClassData()) {
    return;
  }
  // 核心的处理逻辑
  Runtime* const runtime = Runtime::Current();
  {
    // Note: We cannot have thread suspension until the field and method arrays are setup or else
    // Class::VisitFieldRoots may miss some fields or methods.
    ScopedAssertNoThreadSuspension nts(__FUNCTION__);
    // Load static fields.
    // We allow duplicate definitions of the same field in a class_data_item
    // but ignore the repeated indexes here, b/21868015.
    // #1 为属性(static/成员属性) & 方法(direct/virtual方法)分配内存空间
    
    // 获取分配器，用于分配内存
    LinearAlloc* const allocator = GetAllocatorForClassLoader(klass->GetClassLoader());
    // 为静态属性分配内存
    LengthPrefixedArray<ArtField>* sfields = AllocArtFieldArray(self,
                                                                allocator,
                                                                accessor.NumStaticFields());
    // 为成员属性分配内存                                         
    LengthPrefixedArray<ArtField>* ifields = AllocArtFieldArray(self,
                                                                allocator,
                                                                accessor.NumInstanceFields());
    size_t num_sfields = 0u;
    size_t num_ifields = 0u;
    uint32_t last_static_field_idx = 0u;
    uint32_t last_instance_field_idx = 0u;

    // Methods
    bool has_oat_class = false;
    const OatFile::OatClass oat_class = (runtime->IsStarted() && !runtime->IsAotCompiler())
        ? OatFile::FindOatClass(dex_file, klass->GetDexClassDefIndex(), &has_oat_class)
        : OatFile::OatClass::Invalid();
    const OatFile::OatClass* oat_class_ptr = has_oat_class ? &oat_class : nullptr;
    // 为direct & virtual方法非配内存
    klass->SetMethodsPtr(
        AllocArtMethodArray(self, allocator, accessor.NumMethods()),
        accessor.NumDirectMethods(),
        accessor.NumVirtualMethods());
    size_t class_def_method_index = 0;
    uint32_t last_dex_method_index = dex::kDexNoIndex;
    size_t last_class_def_method_index = 0;

    uint16_t hotness_threshold = runtime->GetJITOptions()->GetWarmupThreshold();
    // Use the visitor since the ranged based loops are bit slower from seeking. Seeking to the
    // methods needs to decode all of the fields.
    // #2 填充array内存空间
    // 通过使用visitor的方式为ArtField array & ArtMethod array填充实际的信息。关于填充的实现逻辑如下：
    // 针对于属性：使用LoadField对ArtField进行填充
    // 针对于方法：使用LoadMethod + LinkCode进行填充
    accessor.VisitFieldsAndMethods([&](
        const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
          uint32_t field_idx = field.GetIndex();
          DCHECK_GE(field_idx, last_static_field_idx);  // Ordering enforced by DexFileVerifier.
          if (num_sfields == 0 || LIKELY(field_idx > last_static_field_idx)) {
            LoadField(field, klass, &sfields->At(num_sfields));
            ++num_sfields;
            last_static_field_idx = field_idx;
          }
        }, [&](const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
          uint32_t field_idx = field.GetIndex();
          DCHECK_GE(field_idx, last_instance_field_idx);  // Ordering enforced by DexFileVerifier.
          if (num_ifields == 0 || LIKELY(field_idx > last_instance_field_idx)) {
            LoadField(field, klass, &ifields->At(num_ifields));
            ++num_ifields;
            last_instance_field_idx = field_idx;
          }
        }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
          ArtMethod* art_method = klass->GetDirectMethodUnchecked(class_def_method_index,
              image_pointer_size_);
          LoadMethod(dex_file, method, klass.Get(), art_method);
          LinkCode(this, art_method, oat_class_ptr, class_def_method_index);
          uint32_t it_method_index = method.GetIndex();
          if (last_dex_method_index == it_method_index) {
            // duplicate case
            art_method->SetMethodIndex(last_class_def_method_index);
          } else {
            art_method->SetMethodIndex(class_def_method_index);
            last_dex_method_index = it_method_index;
            last_class_def_method_index = class_def_method_index;
          }
          art_method->ResetCounter(hotness_threshold);
          ++class_def_method_index;
        }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
          ArtMethod* art_method = klass->GetVirtualMethodUnchecked(
              class_def_method_index - accessor.NumDirectMethods(),
              image_pointer_size_);
          art_method->ResetCounter(hotness_threshold);
          LoadMethod(dex_file, method, klass.Get(), art_method);
          LinkCode(this, art_method, oat_class_ptr, class_def_method_index);
          ++class_def_method_index;
        });
	// 如果填充的instance field + static field != dex中实际的field数目，则对ifields, sfields大小进行修改。
	// Note: 只有当有重复的ifields/sfields才会出现这样而case.
    if (UNLIKELY(num_ifields + num_sfields != accessor.NumFields())) {
      LOG(WARNING) << "Duplicate fields in class " << klass->PrettyDescriptor()
          << " (unique static fields: " << num_sfields << "/" << accessor.NumStaticFields()
          << ", unique instance fields: " << num_ifields << "/" << accessor.NumInstanceFields()
          << ")";
      // NOTE: Not shrinking the over-allocated sfields/ifields, just setting size.
      if (sfields != nullptr) {
        sfields->SetSize(num_sfields);
      }
      if (ifields != nullptr) {
        ifields->SetSize(num_ifields);
      }
    }
    // Set the field arrays.
    // #3 更新class字段信息.
    
    // 设置ifields
    klass->SetSFieldsPtr(sfields);
    // 检查大小是否匹配，但debug环境会抛出异常
    DCHECK_EQ(klass->NumStaticFields(), num_sfields);
    // 设置sfields
    klass->SetIFieldsPtr(ifields);
    // 检查大小是否匹配，但debug环境会抛出异常
    DCHECK_EQ(klass->NumInstanceFields(), num_ifields);
  }
  // Ensure that the card is marked so that remembered sets pick up native roots.
  WriteBarrier::ForEveryFieldWrite(klass.Get());
  self->AllowThreadSuspension();
}
```


#### LoadField

LoadField主要作用是用于填充ArtField.
调用处集中在VisitFieldsAndMethods传入的第一、第二个lambda中
```c++

sor.VisitFieldsAndMethods([&](const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
          uint32_t field_idx = field.GetIndex();
          DCHECK_GE(field_idx, last_static_field_idx);  // Ordering enforced by DexFileVerifier.
          // dex中field_idx递增，使用这样的判断条件是用来去重。
          if (num_sfields == 0 || LIKELY(field_idx > last_static_field_idx)) {
            // 1. 填充处理static field
            // Note：这里的sfields是AllocArtFieldArray创建的ArtField集合，VisitFieldsAndMethods遍历完成后会设置到Class对象中去。
            LoadField(field, klass, &sfields->At(num_sfields));
            ++num_sfields;
            last_static_field_idx = field_idx;
          }
        }, [&](const ClassAccessor::Field& field) REQUIRES_SHARED(Locks::mutator_lock_) {
          uint32_t field_idx = field.GetIndex();
          DCHECK_GE(field_idx, last_instance_field_idx);  // Ordering enforced by DexFileVerifier.
          if (num_ifields == 0 || LIKELY(field_idx > last_instance_field_idx)) {
	        // 2.填充处理instance field
            LoadField(field, klass, &ifields->At(num_ifields));
            ++num_ifields;
            last_instance_field_idx = field_idx;
          }
        },
        [&]() {
	        ...
        }, [&]() {
	        ...
        })
```


LoadField具体实现逻辑, 非常简单就是设置了几个index标记。
```c++
void ClassLinker::LoadField(const ClassAccessor::Field& field,
                            Handle<mirror::Class> klass,
                            ArtField* dst) {
  const uint32_t field_idx = field.GetIndex();
  dst->SetDexFieldIndex(field_idx);
  dst->SetDeclaringClass(klass.Get());

  // Get access flags from the DexFile and set hiddenapi runtime access flags.
  dst->SetAccessFlags(field.GetAccessFlags() | hiddenapi::CreateRuntimeFlags(field));
}

```


#### LoadMethod


LoadField主要作用是用于填充ArtMethod.
调用处集中在VisitFieldsAndMethods传入的第三、第四个lambda中
```c++
accessor.VisitFieldsAndMethods([&]() {
	        ...
        }, [&]() {
	        ...
        }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
          // 获取artMethod 
          ArtMethod* art_method = klass->GetDirectMethodUnchecked(class_def_method_index,
              image_pointer_size_);
          // 填充artMethod    
          LoadMethod(dex_file, method, klass.Get(), art_method);
          LinkCode(this, art_method, oat_class_ptr, class_def_method_index);
          uint32_t it_method_index = method.GetIndex();
          if (last_dex_method_index == it_method_index) {
            // duplicate case
            art_method->SetMethodIndex(last_class_def_method_index);
          } else {
            art_method->SetMethodIndex(class_def_method_index);
            last_dex_method_index = it_method_index;
            last_class_def_method_index = class_def_method_index;
          }
          art_method->ResetCounter(hotness_threshold);
          ++class_def_method_index;
        }, [&](const ClassAccessor::Method& method) REQUIRES_SHARED(Locks::mutator_lock_) {
          // 获取artMethod 
          ArtMethod* art_method = klass->GetVirtualMethodUnchecked(
              class_def_method_index - accessor.NumDirectMethods(),
              image_pointer_size_);
          // 填充artMethod    
          art_method->ResetCounter(hotness_threshold);
          LoadMethod(dex_file, method, klass.Get(), art_method);
          LinkCode(this, art_method, oat_class_ptr, class_def_method_index);
          ++class_def_method_index;
        });
```

接下来我们具体分析下ArtMethod的填充逻辑
- GetDirectMethodUnchecked
	通过调用GetMethodsPtr()获取class.methods_，在通过下标获取特定的ArtMethod对象。
```c++
inline ArtMethod* Class::GetDirectMethodUnchecked(size_t i, PointerSize pointer_size) {
  CheckPointerSize(pointer_size);
  return &GetDirectMethodsSliceUnchecked(pointer_size)[i];
}

inline ArraySlice<ArtMethod> Class::GetDirectMethodsSliceUnchecked(PointerSize pointer_size) {
  return GetMethodsSliceRangeUnchecked(GetMethodsPtr(),
                                       pointer_size,
                                       GetDirectMethodsStartOffset(),
                                       GetVirtualMethodsStartOffset());
}
```
- LoadMethod
	LoadMethod方法主要为ArtMethod填充access_flags、data_字段

方法的执行逻辑比较长，主要原因是针对不同的而方法类型做了一些特殊的处理，具体可见下方表格。

| 方法          | 解释                                                                                                                       |
| ----------- | ------------------------------------------------------------------------------------------------------------------------ |
| finalize方法  | 1.将accessFlags中添加kAccClassIsFinalizable<br>2.设置accessFlags                                                               |
| init/clinit | 取保init方法accessFlag中含有kAccConstructor                                                                                     |
| Native方法    | 1.为FastNative方法设置kAccFastNative标记<br>2.为CriticalNative方法设置kAccCriticalNative标记<br>3.设置data_字段为null<br>4.设置accessFlags    |
| 抽象方法        | 1.如果抽象方法是接口类型计算并设置imt\_index\_<br>2.设置data_字段为null<br>3.设置accessFlags                                                    |
| 其他普通方法      | 1.获取annotation标记，如果被标记为NeverCompile则将accessFlags添加kAccCompileDontBother字段<br>2.将data_字段设置为codeItem偏移量<br>3.设置accessFlags |

除开上述所有不同方法类型的分叉逻辑，还有如下共享的逻辑
1.设置method idx、declaring class
2.根据shorty为nterp fast path 方法设置特定的accessFlags
```c++

void ClassLinker::LoadMethod(const DexFile& dex_file,
                             const ClassAccessor::Method& method,
                             ObjPtr<mirror::Class> klass,
                             ArtMethod* dst) {
  ScopedAssertNoThreadSuspension sants(__FUNCTION__);
  // 获取method信息，包括method_id、method_name、shorty
  const uint32_t dex_method_idx = method.GetIndex();
  const dex::MethodId& method_id = dex_file.GetMethodId(dex_method_idx);
  uint32_t name_utf16_length;
  const char* method_name = dex_file.StringDataAndUtf16LengthByIdx(method_id.name_idx_,
                                                                   &name_utf16_length);
  std::string_view shorty = dex_file.GetShortyView(dex_file.GetProtoId(method_id.proto_idx_));
  // 1.设置method idx、declaring class
  dst->SetDexMethodIndex(dex_method_idx);
  dst->SetDeclaringClass(klass);

  // Get access flags from the DexFile and set hiddenapi runtime access flags.
  uint32_t access_flags = method.GetAccessFlags() | hiddenapi::CreateRuntimeFlags(method);

  auto has_ascii_name = [method_name, name_utf16_length](const char* ascii_name,
                                                         size_t length) ALWAYS_INLINE {
    DCHECK_EQ(strlen(ascii_name), length);
    return length == name_utf16_length &&
           method_name[length] == 0 &&  // Is `method_name` an ASCII string?
           memcmp(ascii_name, method_name, length) == 0;
  };
  
  // 不同方法类类型的分叉逻辑
  if (UNLIKELY(has_ascii_name("finalize", sizeof("finalize") - 1u))) { // 处理finalize方法
    // Set finalizable flag on declaring class.
    if (shorty == "V") {
      // Void return type.
      if (klass->GetClassLoader() != nullptr) {  // All non-boot finalizer methods are flagged.
        klass->SetFinalizable();
      } else {
        std::string_view klass_descriptor =
            dex_file.GetTypeDescriptorView(dex_file.GetTypeId(klass->GetDexTypeIndex()));
        // The Enum class declares a "final" finalize() method to prevent subclasses from
        // introducing a finalizer. We don't want to set the finalizable flag for Enum or its
        // subclasses, so we exclude it here.
        // We also want to avoid setting the flag on Object, where we know that finalize() is
        // empty.
        if (klass_descriptor != "Ljava/lang/Object;" &&
            klass_descriptor != "Ljava/lang/Enum;") {
          klass->SetFinalizable();
        }
      }
    }
  } else if (method_name[0] == '<') { // 构造方法/静态代码块
    // Fix broken access flags for initializers. Bug 11157540.
    bool is_init = has_ascii_name("<init>", sizeof("<init>") - 1u);
    bool is_clinit = has_ascii_name("<clinit>", sizeof("<clinit>") - 1u);
    if (UNLIKELY(!is_init && !is_clinit)) {
      LOG(WARNING) << "Unexpected '<' at start of method name " << method_name;
    } else {
      if (UNLIKELY((access_flags & kAccConstructor) == 0)) {
        LOG(WARNING) << method_name << " didn't have expected constructor access flag in class "
            << klass->PrettyDescriptor() << " in dex file " << dex_file.GetLocation();
        access_flags |= kAccConstructor;
      }
    }
  }

  // 2.根据shorty为方法设置kAccNterpEntryPointFastPathFlag/kAccNterpInvokeFastPathFlag accessFlags
  access_flags |= GetNterpFastPathFlags(shorty, access_flags, kRuntimeISA);

  if (UNLIKELY((access_flags & kAccNative) != 0u)) { // native方法
    // Check if the native method is annotated with @FastNative or @CriticalNative.
    const dex::ClassDef& class_def = dex_file.GetClassDef(klass->GetDexClassDefIndex());
    access_flags |=
        annotations::GetNativeMethodAnnotationAccessFlags(dex_file, class_def, dex_method_idx);
    dst->SetAccessFlags(access_flags);
    DCHECK(!dst->IsAbstract());
    DCHECK(!dst->HasCodeItem());
    DCHECK_EQ(method.GetCodeItemOffset(), 0u);
    dst->SetDataPtrSize(nullptr, image_pointer_size_);  // JNI stub/trampoline not linked yet.
  } else if ((access_flags & kAccAbstract) != 0u) { // 抽象方法
    dst->SetAccessFlags(access_flags);
    // Must be done after SetAccessFlags since IsAbstract depends on it.
    DCHECK(dst->IsAbstract());
    if (klass->IsInterface()) {
      dst->CalculateAndSetImtIndex();
    }
    DCHECK(!dst->HasCodeItem());
    DCHECK_EQ(method.GetCodeItemOffset(), 0u);
    dst->SetDataPtrSize(nullptr, image_pointer_size_);  // Single implementation not set yet.
  } else { // 其他普通方法
    const dex::ClassDef& class_def = dex_file.GetClassDef(klass->GetDexClassDefIndex());
    if (annotations::MethodIsNeverCompile(dex_file, class_def, dex_method_idx)) {
      access_flags |= kAccCompileDontBother;
    }
    dst->SetAccessFlags(access_flags);
    DCHECK(!dst->IsAbstract());
    DCHECK(dst->HasCodeItem());
    uint32_t code_item_offset = method.GetCodeItemOffset();
    DCHECK_NE(code_item_offset, 0u);
    if (Runtime::Current()->IsAotCompiler()) {
      dst->SetDataPtrSize(reinterpret_cast32<void*>(code_item_offset), image_pointer_size_);
    } else {
      dst->SetCodeItem(dex_file.GetCodeItem(code_item_offset), dex_file.IsCompactDexFile());
    }
  }

  if (Runtime::Current()->IsZygote() &&
      !Runtime::Current()->GetJITOptions()->GetProfileSaverOptions().GetProfileBootClassPath()) {
    dst->SetMemorySharedMethod();
  }
}

```


#### LinkCode


LinkCode方法核心目的是
1.ArtMethod entry_point_from_quick_compiled_code的设置(代码主要在InitializeMethodsCode)
2.ArtMethod data_字段的设置

按照代码执行逻辑可分为如下几个步骤
1.过滤aot编译器，不允许aot编译器链接代码
2.link abstract方法
3.尝试获取oat方法的地址
4.集中设置ArtMethod entroy point的值(此处需要传入oat地址，会处理方法oat的情况)
5.为Native方法设置Data字段

```c++
static void LinkCode(ClassLinker* class_linker,
                     ArtMethod* method,
                     const OatFile::OatClass* oat_class,
                     uint32_t class_def_method_index) REQUIRES_SHARED(Locks::mutator_lock_) {
  ScopedAssertNoThreadSuspension sants(__FUNCTION__);
  Runtime* const runtime = Runtime::Current();
  // #1 过滤aot编译器，不允许aot编译器链接代码
  if (runtime->IsAotCompiler()) {
    // The following code only applies to a non-compiler runtime.
    return;
  }

  // 链接前确保连接之前没有链接过
  // Method shouldn't have already been linked.
  DCHECK_EQ(method->GetEntryPointFromQuickCompiledCode(), nullptr);
  DCHECK(!method->GetDeclaringClass()->IsVisiblyInitialized());  // Actually ClassStatus::Idx.

  // #2 link abstract方法
  if (!method->IsInvokable()) {
    EnsureThrowsInvocationError(class_linker, method);
    return;
  }

  // #3 尝试获取oat方法的地址(不一定能拿到)
  const void* quick_code = nullptr;
  if (oat_class != nullptr) {
    // Every kind of method should at least get an invoke stub from the oat_method.
    // non-abstract methods also get their code pointers.
    const OatFile::OatMethod oat_method = oat_class->GetOatMethod(class_def_method_index);
    quick_code = oat_method.GetQuickCode();
  }
  // #4 集中设置ArtMethod entroy point的值
  runtime->GetInstrumentation()->InitializeMethodsCode(method, quick_code);
  // #5 为Native方法设置Data字段
  if (method->IsNative()) {
    // Set up the dlsym lookup stub. Do not go through `UnregisterNative()`
    // as the extra processing for @CriticalNative is not needed yet.
    method->SetEntryPointFromJni(
        method->IsCriticalNative() ? GetJniDlsymLookupCriticalStub() : GetJniDlsymLookupStub());
  }
}
```


InitializeMethodsCode主要就做了一件事设置ArtMethod的entry_point(具体来讲是ArtMethod的属性entry_point_from_quick_compiled_code),其中有比较多的分叉。如下表格，将entry_point的设置按代码执行顺序拆分成了6个阶段。

| 类型           | 备注                                                                                                                                                                                        |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.abstract方法 | 入口设置为art_quick_to_interpreter_bridge                                                                                                                                                      |
| 2.插桩代码       | 1.如果是native方法入口设置为art_quick_generic_jni_trampoline<br>2.如果其他方法设置为art_quick_to_interpreter_bridge                                                                                          |
| 3.需要初始化检查的方法 | 1.需要初始化检查的方法指的是**非静态初始化方法(clinit)以外**的所有方法<br>2.在#1的范围内，入口设置有两个分支<br>    a. aot方法/native方法/能够使用Nterp的方法入口设置为art_quick_resolution_trampoline<br>	b. 其他方法设置为art_quick_to_interpreter_bridge |
| 4.aot方法处理    | 1.aot方法即方法经过aot编译的方法（保存在\*.odex,\*.oat产物中）<br>2.针对于aot方法，入口设置为aot后的方法地址                                                                                                                   |
| 5.nterp方法    | 1.环境满足Nterp & method满足NTerp限制 & 类已经校验通过<br>2.入口设置为interpreter::ExecuteNterpImpl                                                                                                           |
| 6.其他         | 1.Native方法设置为art_quick_generic_jni_trampoline<br>2.其余设置为art_quick_to_interpreter_bridge                                                                                                   |
具体代码如下

```c++
void __attribute__((optnone, noinline, used)) Instrumentation::InitializeMethodsCode(ArtMethod* method, const void* aot_code)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  // #1 处理abstract方法 
  // 将abstract方法的entry_point设置为art_quick_to_interpreter_bridge
  if (!method->IsInvokable()) {
    DCHECK(method->GetEntryPointFromQuickCompiledCode() == nullptr ||
           Runtime::Current()->GetClassLinker()->IsQuickToInterpreterBridge(
               method->GetEntryPointFromQuickCompiledCode()));
    UpdateEntryPoints(method, GetQuickToInterpreterBridge());
    return;
  }

  // Use instrumentation entrypoints if instrumentation is installed.
  // #2 处理插桩代码
  // 当有设置instrument listener,方法将以解释的方式执行
  if (UNLIKELY(EntryExitStubsInstalled() || IsForcedInterpretOnly() || IsDeoptimized(method))) {
    // native方法entry_point设置为 art_quick_generic_jni_trampoline
    // 普通方法entry_point设置为art_quick_to_interpreter_bridge 
    UpdateEntryPoints(
        method, method->IsNative() ? GetQuickGenericJniStub() : GetQuickToInterpreterBridge());
    return;
  }

  // Special case if we need an initialization check.
  // The method and its declaring class may be dead when starting JIT GC during managed heap GC.
  // #3 处理需要做初始化检查的方法
  // 由于method以及它的declaring class可能在jit gc的过程中被回收。
  // 因此在一些case下需要进行initialization检查。
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

  // Use the provided AOT code if possible.
  // #4 设置aot代码
  if (CanUseAotCode(aot_code)) {
    UpdateEntryPoints(method, aot_code);
    return;
  }

  // We check if the class is verified as we need the slow interpreter for lock verification.
  // If the class is not verified, This will be updated in
  // ClassLinker::UpdateClassAfterVerification.
  // #5 fast-path
  if (CanUseNterp(method)) {
    UpdateEntryPoints(method, interpreter::GetNterpEntryPoint());
    return;
  }

  // Use default entrypoints.
  // #6 默认入口 slow-path
  UpdateEntryPoints(
      method, method->IsNative() ? GetQuickGenericJniStub() : GetQuickToInterpreterBridge());
}
```



## 解析父类与接口


- **加载父类和接口**：
    调用 `LoadSuperAndInterfaces`递归加载父类及接口，确保继承链完整。若失败，标记类状态为 `kErrorUnresolved`
- **验证类结构**：
    隐含在加载过程中，确保父类、接口的访问权限和符号引用合法性。
```c++
  // Finish loading (if necessary) by finding parents
  CHECK(!klass->IsLoaded());
  // #1 加载父类以及接口类
  if (!LoadSuperAndInterfaces(klass, *new_dex_file)) { // 如果父类 & 接口类有任意一个有问题
    // Loading failed.
    // kclass过程中如果有异常直接抛出
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
    }
    // 返回
    return sdc.Finish(nullptr);
  }
  CHECK(klass->IsLoaded());

  // At this point the class is loaded. Publish a ClassLoad event.
  // Note: this may be a temporary class. It is a listener's responsibility to handle this.
  // 发送一个ClassLoad Event.
  Runtime::Current()->GetRuntimeCallbacks()->ClassLoad(klass);
```

- LoadSuperAndInterfaces
```c++


bool ClassLinker::LoadSuperAndInterfaces(Handle<mirror::Class> klass, const DexFile& dex_file) {
  CHECK_EQ(ClassStatus::kIdx, klass->GetStatus());
  const dex::ClassDef& class_def = dex_file.GetClassDef(klass->GetDexClassDefIndex());
  dex::TypeIndex super_class_idx = class_def.superclass_idx_;
  if (super_class_idx.IsValid()) {
    // Check that a class does not inherit from itself directly.
    //
    // TODO: This is a cheap check to detect the straightforward case
    // of a class extending itself (b/28685551), but we should do a
    // proper cycle detection on loaded classes, to detect all cases
    // of class circularity errors (b/28830038).
    if (super_class_idx == class_def.class_idx_) {
      ThrowClassCircularityError(klass.Get(),
                                 "Class %s extends itself",
                                 klass->PrettyDescriptor().c_str());
      return false;
    }

    ObjPtr<mirror::Class> super_class = ResolveType(super_class_idx, klass.Get());
    if (super_class == nullptr) {
      DCHECK(Thread::Current()->IsExceptionPending());
      return false;
    }
    // Verify
    if (!klass->CanAccess(super_class)) {
      ThrowIllegalAccessError(klass.Get(), "Class %s extended by class %s is inaccessible",
                              super_class->PrettyDescriptor().c_str(),
                              klass->PrettyDescriptor().c_str());
      return false;
    }
    CHECK(super_class->IsResolved());
    klass->SetSuperClass(super_class);
  }
  const dex::TypeList* interfaces = dex_file.GetInterfacesList(class_def);
  if (interfaces != nullptr) {
    for (size_t i = 0; i < interfaces->Size(); i++) {
      dex::TypeIndex idx = interfaces->GetTypeItem(i).type_idx_;
      if (idx.IsValid()) {
        // Check that a class does not implement itself directly.
        //
        // TODO: This is a cheap check to detect the straightforward case of a class implementing
        // itself, but we should do a proper cycle detection on loaded classes, to detect all cases
        // of class circularity errors. See b/28685551, b/28830038, and b/301108855
        if (idx == class_def.class_idx_) {
          ThrowClassCircularityError(
              klass.Get(), "Class %s implements itself", klass->PrettyDescriptor().c_str());
          return false;
        }
      }

      ObjPtr<mirror::Class> interface = ResolveType(idx, klass.Get());
      if (interface == nullptr) {
        DCHECK(Thread::Current()->IsExceptionPending());
        return false;
      }
      // Verify
      if (!klass->CanAccess(interface)) {
        // TODO: the RI seemed to ignore this in my testing.
        ThrowIllegalAccessError(klass.Get(),
                                "Interface %s implemented by class %s is inaccessible",
                                interface->PrettyDescriptor().c_str(),
                                klass->PrettyDescriptor().c_str());
        return false;
      }
    }
  }
  // Mark the class as loaded.
  mirror::Class::SetStatus(klass, ClassStatus::kLoaded, nullptr);
  return true;
}
```


## 连接与优化

- **链接虚方法表（vtable）**：
    解析虚方法调用，生成虚方法表指针 `vtable_`，支持动态分派    
- **接口方法表（iftable_）**：
    处理接口方法实现，填充 `iftable_`以支持多接口继承
- **JIT 编译优化**：
    若 JIT 处于活动状态，调用 `Jit::NewTypeLoadedIfUsingJit`通知 JIT 编译器新类型加载

```c++
  // Link the class (if necessary)
  
  // 确保当前类是没有链接的状态
  CHECK(!klass->IsResolved());
  // TODO: Use fast jobjects?
  auto interfaces = hs.NewHandle<mirror::ObjectArray<mirror::Class>>(nullptr);

  MutableHandle<mirror::Class> h_new_class = hs.NewHandle<mirror::Class>(nullptr);
  // #1 对方法做链接操作
  if (!LinkClass(self, descriptor, klass, interfaces, &h_new_class)) {
    // Linking failed.
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
    }
    return sdc.Finish(nullptr);
  }
  self->AssertNoPendingException();
  CHECK(h_new_class != nullptr) << descriptor;
  CHECK(h_new_class->IsResolved()) << descriptor << " " << h_new_class->GetStatus();

  // Instrumentation may have updated entrypoints for all methods of all
  // classes. However it could not update methods of this class while we
  // were loading it. Now the class is resolved, we can update entrypoints
  // as required by instrumentation.
  // #2 处理Instrumentation钩子的case
  if (Runtime::Current()->GetInstrumentation()->EntryExitStubsInstalled()) {
    // We must be in the kRunnable state to prevent instrumentation from
    // suspending all threads to update entrypoints while we are doing it
    // for this class.
    DCHECK_EQ(self->GetState(), ThreadState::kRunnable);
    Runtime::Current()->GetInstrumentation()->InstallStubsForClass(h_new_class.Get());
  }

  /*
   * We send CLASS_PREPARE events to the debugger from here.  The
   * definition of "preparation" is creating the static fields for a
   * class and initializing them to the standard default values, but not
   * executing any code (that comes later, during "initialization").
   *
   * We did the static preparation in LinkClass.
   *
   * The class has been prepared and resolved but possibly not yet verified
   * at this point.
   */
  // 发送准备完成的回调 
  Runtime::Current()->GetRuntimeCallbacks()->ClassPrepare(klass, h_new_class);

  // Notify native debugger of the new class and its layout.
  // #3 通知解释器有新类加载完成
  jit::Jit::NewTypeLoadedIfUsingJit(h_new_class.Get());

  return sdc.Finish(h_new_class);
```


### LinkClass

1. 验证父类、方法、字段的合法性；
2. 解析符号引用（方法、字段）为实际内存地址；
3. 构建虚方法表（VTable）和接口方法表（IMT），优化方法调用；
4. 处理临时类的替换，确保类内存布局正确；
5. 最终将类标记为 “已解析”（`kResolved`），为初始化阶段做准备。

```c++

bool ClassLinker::LinkClass(Thread* self,
                            const char* descriptor,
                            Handle<mirror::Class> klass,
                            Handle<mirror::ObjectArray<mirror::Class>> interfaces,
                            MutableHandle<mirror::Class>* h_new_class_out) {
  // 确保类处于 “已加载” 状态
  CHECK_EQ(ClassStatus::kLoaded, klass->GetStatus());
  // #1 链接父类（检查父类信息）
  if (!LinkSuperClass(klass)) {
    return false;
  }
  ArtMethod* imt_data[ImTable::kSize];
  // If there are any new conflicts compared to super class.
  bool new_conflict = false;
  std::fill_n(imt_data, arraysize(imt_data), Runtime::Current()->GetImtUnimplementedMethod());
  // #2 链接方法
  if (!LinkMethods(self, klass, interfaces, &new_conflict, imt_data)) {
    return false;
  }
  // #3 链接实例字段
  if (!LinkInstanceFields(self, klass)) {
    return false;
  }
  size_t class_size;
  // #4 链接静态字段
  if (!LinkStaticFields(self, klass, &class_size)) {
    return false;
  }
  CreateReferenceInstanceOffsets(klass);
  CHECK_EQ(ClassStatus::kLoaded, klass->GetStatus());

  // #5 构建IME Table
  ImTable* imt = nullptr;
  if (klass->ShouldHaveImt()) {
    // If there are any new conflicts compared to the super class we can not make a copy. There
    // can be cases where both will have a conflict method at the same slot without having the same
    // set of conflicts. In this case, we can not share the IMT since the conflict table slow path
    // will possibly create a table that is incorrect for either of the classes.
    // Same IMT with new_conflict does not happen very often.
    if (!new_conflict) {
      ImTable* super_imt = klass->FindSuperImt(image_pointer_size_);
      if (super_imt != nullptr) {
        bool imt_equals = true;
        for (size_t i = 0; i < ImTable::kSize && imt_equals; ++i) {
          imt_equals = imt_equals && (super_imt->Get(i, image_pointer_size_) == imt_data[i]);
        }
        if (imt_equals) {
          imt = super_imt;
        }
      }
    }
    if (imt == nullptr) {
      LinearAlloc* allocator = GetAllocatorForClassLoader(klass->GetClassLoader());
      imt = reinterpret_cast<ImTable*>(
          allocator->Alloc(self,
                           ImTable::SizeInBytes(image_pointer_size_),
                           LinearAllocKind::kNoGCRoots));
      if (imt == nullptr) {
        return false;
      }
      imt->Populate(imt_data, image_pointer_size_);
    }
  }

  // #6 处理非临时类，标记为已解析 
  if (!klass->IsTemp() || (!init_done_ && klass->GetClassSize() == class_size)) {
    // We don't need to retire this class as it has no embedded tables or it was created the
    // correct size during class linker initialization.
    CHECK_EQ(klass->GetClassSize(), class_size) << klass->PrettyDescriptor();

    if (klass->ShouldHaveEmbeddedVTable()) {
      klass->PopulateEmbeddedVTable(image_pointer_size_);
    }
    if (klass->ShouldHaveImt()) {
      klass->SetImt(imt, image_pointer_size_);
    }

    // Update CHA info based on whether we override methods.
    // Have to do this before setting the class as resolved which allows
    // instantiation of klass.
    if (LIKELY(descriptor != nullptr) && cha_ != nullptr) {
      cha_->UpdateAfterLoadingOf(klass);
    }

    // This will notify waiters on klass that saw the not yet resolved
    // class in the class_table_ during EnsureResolved.
    mirror::Class::SetStatus(klass, ClassStatus::kResolved, self);
    h_new_class_out->Assign(klass.Get());
  } else {
    // #7 处理临时类，替换为正式类
  
    CHECK(!klass->IsResolved());
    // Retire the temporary class and create the correctly sized resolved class.
    StackHandleScope<1> hs(self);
    Handle<mirror::Class> h_new_class =
        hs.NewHandle(mirror::Class::CopyOf(klass, self, class_size, imt, image_pointer_size_));
    // Set arrays to null since we don't want to have multiple classes with the same ArtField or
    // ArtMethod array pointers. If this occurs, it causes bugs in remembered sets since the GC
    // may not see any references to the target space and clean the card for a class if another
    // class had the same array pointer.
    klass->SetMethodsPtrUnchecked(nullptr, 0, 0);
    klass->SetSFieldsPtrUnchecked(nullptr);
    klass->SetIFieldsPtrUnchecked(nullptr);
    if (UNLIKELY(h_new_class == nullptr)) {
      self->AssertPendingOOMException();
      mirror::Class::SetStatus(klass, ClassStatus::kErrorUnresolved, self);
      return false;
    }

    CHECK_EQ(h_new_class->GetClassSize(), class_size);
    ObjectLock<mirror::Class> lock(self, h_new_class);
    FixupTemporaryDeclaringClass(klass.Get(), h_new_class.Get());

    if (LIKELY(descriptor != nullptr)) {
      WriterMutexLock mu(self, *Locks::classlinker_classes_lock_);
      const ObjPtr<mirror::ClassLoader> class_loader = h_new_class.Get()->GetClassLoader();
      ClassTable* const table = InsertClassTableForClassLoader(class_loader);
      const ObjPtr<mirror::Class> existing =
          table->UpdateClass(descriptor, h_new_class.Get(), ComputeModifiedUtf8Hash(descriptor));
      CHECK_EQ(existing, klass.Get());
      WriteBarrierOnClassLoaderLocked(class_loader, h_new_class.Get());
    }

    // Update CHA info based on whether we override methods.
    // Have to do this before setting the class as resolved which allows
    // instantiation of klass.
    if (LIKELY(descriptor != nullptr) && cha_ != nullptr) {
      cha_->UpdateAfterLoadingOf(h_new_class);
    }

    // This will notify waiters on temp class that saw the not yet resolved class in the
    // class_table_ during EnsureResolved.
    mirror::Class::SetStatus(klass, ClassStatus::kRetired, self);

    CHECK_EQ(h_new_class->GetStatus(), ClassStatus::kResolving);
    // This will notify waiters on new_class that saw the not yet resolved
    // class in the class_table_ during EnsureResolved.
    mirror::Class::SetStatus(h_new_class, ClassStatus::kResolved, self);
    // Return the new class.
    h_new_class_out->Assign(h_new_class.Get());
  }
  return true;
}
```

#### LinkSuperClass


```c++
bool ClassLinker::LinkSuperClass(Handle<mirror::Class> klass) {
  // 确保类型是非基本类型
  CHECK(!klass->IsPrimitive());
  // 获取当前类实例和父类实例
  ObjPtr<mirror::Class> super = klass->GetSuperClass();
  ObjPtr<mirror::Class> object_class = GetClassRoot<mirror::Object>(this);
  if (klass.Get() == object_class) {
    if (super != nullptr) {
      ThrowClassFormatError(klass.Get(), "java.lang.Object must not have a superclass");
      return false;
    }
    return true;
  }
  if (super == nullptr) {
    ThrowLinkageError(klass.Get(), "No superclass defined for class %s",
                      klass->PrettyDescriptor().c_str());
    return false;
  }
  // Verify
  if (klass->IsInterface() && super != object_class) {
    ThrowClassFormatError(klass.Get(), "Interfaces must have java.lang.Object as superclass");
    return false;
  }
  if (super->IsFinal()) {
    ThrowVerifyError(klass.Get(),
                     "Superclass %s of %s is declared final",
                     super->PrettyDescriptor().c_str(),
                     klass->PrettyDescriptor().c_str());
    return false;
  }
  if (super->IsInterface()) {
    ThrowIncompatibleClassChangeError(klass.Get(),
                                      "Superclass %s of %s is an interface",
                                      super->PrettyDescriptor().c_str(),
                                      klass->PrettyDescriptor().c_str());
    return false;
  }
  if (!klass->CanAccess(super)) {
    ThrowIllegalAccessError(klass.Get(), "Superclass %s is inaccessible to class %s",
                            super->PrettyDescriptor().c_str(),
                            klass->PrettyDescriptor().c_str());
    return false;
  }
  if (!VerifyRecordClass(klass, super)) {
    DCHECK(Thread::Current()->IsExceptionPending());
    return false;
  }

  // Inherit kAccClassIsFinalizable from the superclass in case this
  // class doesn't override finalize.
  if (super->IsFinalizable()) {
    klass->SetFinalizable();
  }

  // Inherit class loader flag form super class.
  if (super->IsClassLoaderClass()) {
    klass->SetClassLoaderClass();
  }

  // Inherit reference flags (if any) from the superclass.
  uint32_t reference_flags = (super->GetClassFlags() & mirror::kClassFlagReference);
  if (reference_flags != 0) {
    CHECK_EQ(klass->GetClassFlags(), 0u);
    klass->SetClassFlags(klass->GetClassFlags() | reference_flags);
  }
  // Disallow custom direct subclasses of java.lang.ref.Reference.
  if (init_done_ && super == GetClassRoot<mirror::Reference>(this)) {
    ThrowLinkageError(klass.Get(),
                      "Class %s attempts to subclass java.lang.ref.Reference, which is not allowed",
                      klass->PrettyDescriptor().c_str());
    return false;
  }

  if (kIsDebugBuild) {
    // Ensure super classes are fully resolved prior to resolving fields..
    while (super != nullptr) {
      CHECK(super->IsResolved());
      super = super->GetSuperClass();
    }
  }
  return true;
}
```


#### LinkMethods


#### LinkInstanceFields


#### LinkStaticFields


#### 处理非临时类


#### 处理临时类


### Stubs


### Jit::NewTypeLoadedIfUsingJit





# TODO



- [ ] 分析Class的生命周期
- [ ] 补充下Class回调逻辑的分析GetRuntimeCallbacks.XXX
- [ ] 

