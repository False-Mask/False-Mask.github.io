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





## 类初始化与预检查



## 分配与注册 Class 对象




## 加载类元数据



## 解析父类与接口


## 连接与优化



## 状态发布与回调

