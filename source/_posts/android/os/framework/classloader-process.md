---
title: Android ClassLoader加载流程解析
tags:
  - android
  - aosp
cover: >-
  https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/loadclass-and-classforname.png
date: 2025-03-31 00:20:47
---




# Android ClassLoader加载流程



> Note：
>
> 本文主要是针对于ClassLoader加载逻辑进行分析，并未对Class define逻辑进行分析。

![loadclass-and-classforname](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/loadclass-and-classforname.png)

# 起点



1. ClassLoader.load

2. Class.forName



# 过程





## Java执行过程





### 1. ClassLoader.loadClass(String name)

```java
public Class<?> loadClass(String name) throws ClassNotFoundException {
    return loadClass(name, false);
}
```





### 2. ClassLoader.loadClass(name, false)

> 加载流程其实比较固定
>
> 1. 通过findLoadedClass查看该class是否已经加载过
> 2. 调用parent.loadClass加载class
> 3. 如果#2加载失败通过findClass通过自生加载Class

``` java
/**
* name   是需要加载的class名称
* resolve表示是否需要加载class
*/
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
```



#### 2.1 findLoadedClass(name)



> 这一步对应上述加载流程的的第一步`通过findLoadedClass查看该class是否已经加载过`

``` java
protected final Class<?> findLoadedClass(String name) {
        ClassLoader loader;
        // 如果当前ClassLoader是bootClassLoader loader传入null
        if (this == BootClassLoader.getInstance())
            loader = null;
        else
            loader = this;
        // jni调用
        // 在后续的native执行过程中分析
        return VMClassLoader.findLoadedClass(loader, name);
    }
```







#### 2.2 parent.loadClass



> 对应第二步骤。很明显这是一个递归调用。会让loadClass操作流转到parent
>
> 然后递归执行loadClass操作
>
> Note: 需要有一点需要注意，如果parent为null会直接return null

``` java
if (parent != null) { // parent不为null 调用loadClass
    c = parent.loadClass(name, false);
} else {
    c = findBootstrapClassOrNull(name); // parent 为 null直接返回null
}

 private Class<?> findBootstrapClassOrNull(String name)
{
    return null;
}

```



#### 2.3 findClass



> 对应上述的第三步。



> 这方法的原型在ClassLoader.java里面，默认实现是直接跑一场

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
```



> 他有两个比较常见的实现类



> BootClassLoader.java
>
> 约等于Class.forName

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
    // 直接调用到jni
    return Class.classForName(name, false, null);
}
```



> BaseDexClassLoader.java

> 从下方代码可以明显看出#1,＃2的执行是递归操作

```java
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    // First, check whether the class is present in our shared libraries.
    // 1. 首先逐个从sharedLibraries中查看是否包含class
    if (sharedLibraryLoaders != null) {
        for (ClassLoader loader : sharedLibraryLoaders) {
            try {
                return loader.loadClass(name);
            } catch (ClassNotFoundException ignored) {
            }
        }
    }
    // Check whether the class in question is present in the dexPath that
    // this classloader operates on.
    // 2. 从dexElements中逐个查看能否加载成功
    List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
    Class c = pathList.findClass(name, suppressedExceptions);
    if (c != null) {
        return c;
    }
    // Now, check whether the class is present in the "after" shared libraries.
    // 3. 从 sharedLibraryLoadersAfter中逐一查找
    if (sharedLibraryLoadersAfter != null) {
        for (ClassLoader loader : sharedLibraryLoadersAfter) {
            try {
                return loader.loadClass(name);
            } catch (ClassNotFoundException ignored) {
            }
        }
    }
    // 如果1～3都查找过没找到对应的class,直接抛出异常
    if (c == null) {
        ClassNotFoundException cnfe = new ClassNotFoundException(
                "Didn't find class \"" + name + "\" on path: " + pathList);
        for (Throwable t : suppressedExceptions) {
            cnfe.addSuppressed(t);
        }
        throw cnfe;
    }
    return c;
}
```

> DexPathList.findClass

```java
public Class<?> findClass(String name, List<Throwable> suppressed) {
    // 逐一从dexElements中查找class
    // 一层层最终调用到defineClassNative（jni）
    for (Element element : dexElements) {
        Class<?> clazz = element.findClass(name, definingContext, suppressed);
        if (clazz != null) {
            return clazz;
        }
    }

    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
}

public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
    return defineClass(name, loader, mCookie, this, suppressed);
}

 private static Class defineClass(String name, ClassLoader loader, Object cookie,
                                     DexFile dexFile, List<Throwable> suppressed) {
    Class result = null;
    try {
        result = defineClassNative(name, loader, cookie, dexFile);
    } catch (NoClassDefFoundError e) {
        if (suppressed != null) {
            suppressed.add(e);
        }
    } catch (ClassNotFoundException e) {
        if (suppressed != null) {
            suppressed.add(e);
        }
    }
    return result;
}
```





### 小结



> 到此java层的加载流程就结束了。
>
> 简单总结下流程

1. ClassLoader.loadClass(String name)   (这个方法没做什么实际操作)

2. ClassLoader.loadClass(name, false) 

   a. findLoadedClass => VMClassLoader.findLoadedClass (JNI)

   b. parent.loadClass(name, false) (递归)

   c. findClass (不同的ClassLoader有不同的是实现，以BaseDexClassLoader为例)

   ​	i.   sharedLibraryLoaders.loadClass(name) (递归)

   ​	ii.  pathList.findClass => dexElements.findClass => .. => DexFile.defineClassNative (JNI)

   ​	iii. sharedLibraryLoadersAfter.loadClass(name) (递归)



> 如下是一个简单的流程图（中间省略了一些native实现逻辑，留到后续讲解说明。）

![classloader-java](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/classloader-java.png)







## Native执行流程



> 首先需要明确我们需要分析那些逻辑
>
> 1. VMClassLoader.findLoadedClass (JNI)
> 2. Class.classForName
> 3. DexFile.defileClassNative



### VMClassLoadr.findLoadedClass



> 方法整体的执行逻辑如下

![classloader-native-findloadedclass](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/classloader-native-findloadedclass.png)

> 1. 计算java name对应的 hash值
> 2. 通过hash值从classloader table中寻找class是否加载过（加载过直接返回）
> 3. 如果class没有加载过，尝试自行加载class

``` c++
static jclass VMClassLoader_findLoadedClass(JNIEnv* env, jclass, jobject javaLoader,
                                            jstring javaName) {
    // 将javaClassLoader对象转化为native
  ScopedFastNativeObjectAccess soa(env);
  ObjPtr<mirror::ClassLoader> loader = soa.Decode<mirror::ClassLoader>(javaLoader);
  ScopedUtfChars name(env, javaName);
    // 前置判空
  if (name.c_str() == nullptr) {
    return nullptr;
  }
    // 获取ClassLinker实例
  ClassLinker* cl = Runtime::Current()->GetClassLinker();

  // Compute hash once.
  // 计算descriptor hash值
  std::string descriptor(DotToDescriptor(name.c_str()));
  const size_t descriptor_hash = ComputeModifiedUtf8Hash(descriptor.c_str());
  // 从通过hash值从classtable中寻找class
  ObjPtr<mirror::Class> c = VMClassLoader::LookupClass(cl,
                                                       soa.Self(),
                                                       descriptor.c_str(),
                                                       descriptor_hash,
                                                       loader);
  // 如果class不为null,并且以及link过
  if (c != nullptr && c->IsResolved()) {
    return soa.AddLocalReference<jclass>(c);
  }
  // If class is erroneous, throw the earlier failure, wrapped in certain cases. See b/28787733.
  // 如果class不为null并且以及初始化 & link过程有任何问题，直接抛出异常
  if (c != nullptr && c->IsErroneous()) {
    cl->ThrowEarlierClassFailure(c);
    Thread* self = soa.Self();
    ObjPtr<mirror::Class> iae_class =
        self->DecodeJObject(WellKnownClasses::java_lang_IllegalAccessError)->AsClass();
    ObjPtr<mirror::Class> ncdfe_class =
        self->DecodeJObject(WellKnownClasses::java_lang_NoClassDefFoundError)->AsClass();
    ObjPtr<mirror::Class> exception = self->GetException()->GetClass();
    if (exception == iae_class || exception == ncdfe_class) {
      self->ThrowNewWrappedException("Ljava/lang/ClassNotFoundException;",
                                     c->PrettyDescriptor().c_str());
    }
    return nullptr;
  }

  // Hard-coded performance optimization: We know that all failed libcore calls to findLoadedClass
  //                                      are followed by a call to the the classloader to actually
  //                                      load the class.
  // 硬编码的优化代码：
  // findLoadedClass失败后紧接着会调用classloader从而加载class.
  if (loader != nullptr) {
    // Try the common case.
    StackHandleScope<1> hs(soa.Self());
    c = VMClassLoader::FindClassInPathClassLoader(cl,
                                                  soa,
                                                  soa.Self(),
                                                  descriptor.c_str(),
                                                  descriptor_hash,
                                                  hs.NewHandle(loader));
    if (c != nullptr) {
      return soa.AddLocalReference<jclass>(c);
    }
  }

  // The class wasn't loaded, yet, and our fast-path did not apply (e.g., we didn't understand the
  // classloader chain).
  // 如果class没有加载好 & 硬编码的优化代码没有起作用（我们不太能确认classloader的结构）  
  return nullptr;
}
```



> FindClassInPathClassLoader（没有做实际的加载动作）

```c++
static ObjPtr<mirror::Class> FindClassInPathClassLoader(ClassLinker* cl,
                                                          ScopedObjectAccessAlreadyRunnable& soa,
                                                          Thread* self,
                                                          const char* descriptor,
                                                          size_t hash,
                                                          Handle<mirror::ClassLoader> class_loader)
      REQUIRES_SHARED(Locks::mutator_lock_) {
    ObjPtr<mirror::Class> result;
    if (cl->FindClassInBaseDexClassLoader(soa, self, descriptor, hash, class_loader, &result)) {
      DCHECK(!self->IsExceptionPending());
      return result;
    }
    if (self->IsExceptionPending()) {
      self->ClearException();
    }
    return nullptr;
  }
};
```





#### FindClassInBaseDexClassLoader



> FindClassInBaseDexClassLoader方法本质上是通过ClassLoader进行类加载，但是不是所有的classloader都会在native层直接执行。
>
> （如果是自定义的ClassLoader, 系统也不知道应该怎么加载。因为你可能会破坏原本既定好的加载机制。）
>
> 只针对于3种不同类型的ClassLoader做了兼容和处理
>
> 1. ClassLoader 为null or BootClassLoader
>
>    a. 直接通过BootClassLoader加载Class
>
> 2. ClassLoader为PathClassLoader, DexClassLoader, InMemoryDexClassLoader
>
>    a. 先通过parent classloader进行递归加载
>
>    b. 逐一使用shared libraries进行递归加载
>
>    c. 使用当前的classloader加载class
>
>    d. 使用shared libraries after加载class
>
> 3. ClassLoader为DelegateLastClassLoader
>
>    a. 首先使用boot class path加载class
>
>    b. 加载使用则使用shared libraries加载
>
>    c. 再通过classloader loader本身的path进行加载
>
>    d. 最终使用parent进行递归加载。
>
> 4. 其他 （对应于自定义的ClassLoader）
>
>    a. 直接返回

![classloader-findclassinpathclassloader](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/classloader-findclassinpathclassloader.png)

​	

```c++
bool ClassLinker::FindClassInBaseDexClassLoader(ScopedObjectAccessAlreadyRunnable& soa,
                                                Thread* self,
                                                const char* descriptor,
                                                size_t hash,
                                                Handle<mirror::ClassLoader> class_loader,
                                                /*out*/ ObjPtr<mirror::Class>* result) {
  // Termination case: boot class loader.
  if (IsBootClassLoader(soa, class_loader.Get())) {
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInBootClassLoaderClassPath(self, descriptor, hash, result), *result, self);
    return true;
  }

  if (IsPathOrDexClassLoader(soa, class_loader) || IsInMemoryDexClassLoader(soa, class_loader)) {
    // For regular path or dex class loader the search order is:
    //    - parent
    //    - shared libraries
    //    - class loader dex files

    // Create a handle as RegisterDexFile may allocate dex caches (and cause thread suspension).
    StackHandleScope<1> hs(self);
    Handle<mirror::ClassLoader> h_parent(hs.NewHandle(class_loader->GetParent()));
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInBaseDexClassLoader(soa, self, descriptor, hash, h_parent, result),
        *result,
        self);
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInSharedLibraries(soa, self, descriptor, hash, class_loader, result),
        *result,
        self);
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInBaseDexClassLoaderClassPath(soa, descriptor, hash, class_loader, result),
        *result,
        self);
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInSharedLibrariesAfter(soa, self, descriptor, hash, class_loader, result),
        *result,
        self);
    // We did not find a class, but the class loader chain was recognized, so we
    // return true.
    return true;
  }

  if (IsDelegateLastClassLoader(soa, class_loader)) {
    // For delegate last, the search order is:
    //    - boot class path
    //    - shared libraries
    //    - class loader dex files
    //    - parent
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInBootClassLoaderClassPath(self, descriptor, hash, result), *result, self);
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInSharedLibraries(soa, self, descriptor, hash, class_loader, result),
        *result,
        self);
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInBaseDexClassLoaderClassPath(soa, descriptor, hash, class_loader, result),
        *result,
        self);
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInSharedLibrariesAfter(soa, self, descriptor, hash, class_loader, result),
        *result,
        self);

    // Create a handle as RegisterDexFile may allocate dex caches (and cause thread suspension).
    StackHandleScope<1> hs(self);
    Handle<mirror::ClassLoader> h_parent(hs.NewHandle(class_loader->GetParent()));
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInBaseDexClassLoader(soa, self, descriptor, hash, h_parent, result),
        *result,
        self);
    // We did not find a class, but the class loader chain was recognized, so we
    // return true.
    return true;
  }

  // Unsupported class loader.
  *result = nullptr;
  return false;
}
```





### Class.classForName



> JNI调用后的方法入口。
>
> 只是做了参数的解析，没做实际的操作。
>
> 核心逻辑在class_linker->FindClass中

```c++
static jclass Class_classForName(JNIEnv* env, jclass, jstring javaName, jboolean initialize,
                                 jobject javaLoader) {
  ScopedFastNativeObjectAccess soa(env);
  ScopedUtfChars name(env, javaName);
  if (name.c_str() == nullptr) {
    return nullptr;
  }

  // We need to validate and convert the name (from x.y.z to x/y/z).  This
  // is especially handy for array types, since we want to avoid
  // auto-generating bogus array classes.
  // 类名称合法性检查
  if (!IsValidBinaryClassName(name.c_str())) {
    soa.Self()->ThrowNewExceptionF("Ljava/lang/ClassNotFoundException;",
                                   "Invalid name: %s", name.c_str());
    return nullptr;
  }

  // 获取native classloader
  std::string descriptor(DotToDescriptor(name.c_str()));
  StackHandleScope<2> hs(soa.Self());
  Handle<mirror::ClassLoader> class_loader(
      hs.NewHandle(soa.Decode<mirror::ClassLoader>(javaLoader)));
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
  // 寻找类  
  Handle<mirror::Class> c(
      hs.NewHandle(class_linker->FindClass(soa.Self(), descriptor.c_str(), class_loader)));
  // 找不到类实例，抛出异常
  if (c == nullptr) {
    ScopedLocalRef<jthrowable> cause(env, env->ExceptionOccurred());
    env->ExceptionClear();
    jthrowable cnfe = reinterpret_cast<jthrowable>(
        env->NewObject(WellKnownClasses::java_lang_ClassNotFoundException,
                       WellKnownClasses::java_lang_ClassNotFoundException_init,
                       javaName,
                       cause.get()));
    if (cnfe != nullptr) {
      // Make sure allocation didn't fail with an OOME.
      env->Throw(cnfe);
    }
    return nullptr;
  }  
  if (initialize) {
    class_linker->EnsureInitialized(soa.Self(), c, true, true);
  }
  return soa.AddLocalReference<jclass>(c.Get());
}
```



### ClassLinker::FindClass

>代码链路其实比较长
>
>我们可以把整个执行流程分为这么几大类
>
>1. 加载基本数据类型
>2. 计算描述符hash值，从缓存中获取class
>3. 加载类型不是数组类型 & classloader == null
>4. 数组类型类加载
>5. 非数组类型类加载
>6. 加载完成将class insert到缓存



> 从如下流程图我们可以很明显知道class for name针对于3种不同的类型进行了单独的处理操作。
>
> 基本类型、数组类型、非数组类型。
>
> 除此之外就是classl table缓存的获取 & 更新（加载前查缓存，加载后更新缓存）

![classloader-classforname](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/classloader-classforname.png)

```c++
ObjPtr<mirror::Class> ClassLinker::FindClass(Thread* self,
                                             const char* descriptor,
                                             Handle<mirror::ClassLoader> class_loader) {
  // 前处理，确保类名正常，参数值正常。
  DCHECK_NE(*descriptor, '\0') << "descriptor is empty string";
  DCHECK(self != nullptr);
  self->AssertNoPendingException();
  self->PoisonObjectPointers();  // For DefineClass, CreateArrayClass, etc...
  // 1. 描述符长度为1，加载基本数据类型
  if (descriptor[1] == '\0') {
    // only the descriptors of primitive types should be 1 character long, also avoid class lookup
    // for primitive classes that aren't backed by dex files.
    return FindPrimitiveClass(descriptor[0]);
  }
  // 2. 计算描述符hash值，从缓存中获取class
  const size_t hash = ComputeModifiedUtf8Hash(descriptor);
  // Find the class in the loaded classes table.
  ObjPtr<mirror::Class> klass = LookupClass(self, descriptor, hash, class_loader.Get());
  if (klass != nullptr) {
    return EnsureResolved(self, descriptor, klass);
  }
  // Class is not yet loaded.
  // 3. 当前需要加载的类不是数组类型 & classloader为null
  // -> 从boot class path中加载class
  if (descriptor[0] != '[' && class_loader == nullptr) {
    // Non-array class and the boot class loader, search the boot class path.
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
  ObjPtr<mirror::Class> result_ptr;
  bool descriptor_equals;
  // 4. 加载数组类型的class
  if (descriptor[0] == '[') {
    result_ptr = CreateArrayClass(self, descriptor, hash, class_loader);
    DCHECK_EQ(result_ptr == nullptr, self->IsExceptionPending());
    DCHECK(result_ptr == nullptr || result_ptr->DescriptorEquals(descriptor));
    descriptor_equals = true;
  } else {
    // 5. 加载非数组类型的class 
    ScopedObjectAccessUnchecked soa(self);
    bool known_hierarchy =
        FindClassInBaseDexClassLoader(soa, self, descriptor, hash, class_loader, &result_ptr);
    if (result_ptr != nullptr) {
      // The chain was understood and we found the class. We still need to add the class to
      // the class table to protect from racy programs that can try and redefine the path list
      // which would change the Class<?> returned for subsequent evaluation of const-class.
      DCHECK(known_hierarchy);
      DCHECK(result_ptr->DescriptorEquals(descriptor));
      descriptor_equals = true;
    } else if (!self->IsExceptionPending()) {
      // Either the chain wasn't understood or the class wasn't found.
      // If there is a pending exception we didn't clear, it is a not a ClassNotFoundException and
      // we should return it instead of silently clearing and retrying.
      //
      // If the chain was understood but we did not find the class, let the Java-side
      // rediscover all this and throw the exception with the right stack trace. Note that
      // the Java-side could still succeed for racy programs if another thread is actively
      // modifying the class loader's path list.

      // The runtime is not allowed to call into java from a runtime-thread so just abort.
      if (self->IsRuntimeThread()) {
        // Oops, we can't call into java so we can't run actual class-loader code.
        // This is true for e.g. for the compiler (jit or aot).
        ObjPtr<mirror::Throwable> pre_allocated =
            Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
        self->SetException(pre_allocated);
        return nullptr;
      }

      // Inlined DescriptorToDot(descriptor) with extra validation.
      //
      // Throw NoClassDefFoundError early rather than potentially load a class only to fail
      // the DescriptorEquals() check below and give a confusing error message. For example,
      // when native code erroneously calls JNI GetFieldId() with signature "java/lang/String"
      // instead of "Ljava/lang/String;", the message below using the "dot" names would be
      // "class loader [...] returned class java.lang.String instead of java.lang.String".
      size_t descriptor_length = strlen(descriptor);
      if (UNLIKELY(descriptor[0] != 'L') ||
          UNLIKELY(descriptor[descriptor_length - 1] != ';') ||
          UNLIKELY(memchr(descriptor + 1, '.', descriptor_length - 2) != nullptr)) {
        ThrowNoClassDefFoundError("Invalid descriptor: %s.", descriptor);
        return nullptr;
      }

      std::string class_name_string(descriptor + 1, descriptor_length - 2);
      std::replace(class_name_string.begin(), class_name_string.end(), '/', '.');
      if (known_hierarchy &&
          fast_class_not_found_exceptions_ &&
          !Runtime::Current()->IsJavaDebuggable()) {
        // For known hierarchy, we know that the class is going to throw an exception. If we aren't
        // debuggable, optimize this path by throwing directly here without going back to Java
        // language. This reduces how many ClassNotFoundExceptions happen.
        self->ThrowNewExceptionF("Ljava/lang/ClassNotFoundException;",
                                 "%s",
                                 class_name_string.c_str());
      } else {
        ScopedLocalRef<jobject> class_loader_object(
            soa.Env(), soa.AddLocalReference<jobject>(class_loader.Get()));
        ScopedLocalRef<jobject> result(soa.Env(), nullptr);
        {
          ScopedThreadStateChange tsc(self, ThreadState::kNative);
          ScopedLocalRef<jobject> class_name_object(
              soa.Env(), soa.Env()->NewStringUTF(class_name_string.c_str()));
          if (class_name_object.get() == nullptr) {
            DCHECK(self->IsExceptionPending());  // OOME.
            return nullptr;
          }
          CHECK(class_loader_object.get() != nullptr);
          result.reset(soa.Env()->CallObjectMethod(class_loader_object.get(),
                                                   WellKnownClasses::java_lang_ClassLoader_loadClass,
                                                   class_name_object.get()));
        }
        if (result.get() == nullptr && !self->IsExceptionPending()) {
          // broken loader - throw NPE to be compatible with Dalvik
          ThrowNullPointerException(StringPrintf("ClassLoader.loadClass returned null for %s",
                                                 class_name_string.c_str()).c_str());
          return nullptr;
        }
        result_ptr = soa.Decode<mirror::Class>(result.get());
        // Check the name of the returned class.
        descriptor_equals = (result_ptr != nullptr) && result_ptr->DescriptorEquals(descriptor);
      }
    } else {
      DCHECK(!MatchesDexFileCaughtExceptions(self->GetException(), this));
    }
  }

  if (self->IsExceptionPending()) {
    // If the ClassLoader threw or array class allocation failed, pass that exception up.
    // However, to comply with the RI behavior, first check if another thread succeeded.
    result_ptr = LookupClass(self, descriptor, hash, class_loader.Get());
    if (result_ptr != nullptr && !result_ptr->IsErroneous()) {
      self->ClearException();
      return EnsureResolved(self, descriptor, result_ptr);
    }
    return nullptr;
  }

  // Try to insert the class to the class table, checking for mismatch.
  // 6. 将加载的类型插入class table缓存中  
  ObjPtr<mirror::Class> old;
  {
    WriterMutexLock mu(self, *Locks::classlinker_classes_lock_);
    ClassTable* const class_table = InsertClassTableForClassLoader(class_loader.Get());
    old = class_table->Lookup(descriptor, hash);
    if (old == nullptr) {
      old = result_ptr;  // For the comparison below, after releasing the lock.
      if (descriptor_equals) {
        class_table->InsertWithHash(result_ptr, hash);
        WriteBarrier::ForEveryFieldWrite(class_loader.Get());
      }  // else throw below, after releasing the lock.
    }
  }
  
  // 后处理，确保缓存正常。  
  if (UNLIKELY(old != result_ptr)) {
    // Return `old` (even if `!descriptor_equals`) to mimic the RI behavior for parallel
    // capable class loaders.  (All class loaders are considered parallel capable on Android.)
    ObjPtr<mirror::Class> loader_class = class_loader->GetClass();
    const char* loader_class_name =
        loader_class->GetDexFile().StringByTypeIdx(loader_class->GetDexTypeIndex());
    LOG(WARNING) << "Initiating class loader of type " << DescriptorToDot(loader_class_name)
        << " is not well-behaved; it returned a different Class for racing loadClass(\""
        << DescriptorToDot(descriptor) << "\").";
    return EnsureResolved(self, descriptor, old);
  }
  if (UNLIKELY(!descriptor_equals)) {
    std::string result_storage;
    const char* result_name = result_ptr->GetDescriptor(&result_storage);
    std::string loader_storage;
    const char* loader_class_name = class_loader->GetClass()->GetDescriptor(&loader_storage);
    ThrowNoClassDefFoundError(
        "Initiating class loader of type %s returned class %s instead of %s.",
        DescriptorToDot(loader_class_name).c_str(),
        DescriptorToDot(result_name).c_str(),
        DescriptorToDot(descriptor).c_str());
    return nullptr;
  }
  // Success.
  return result_ptr;
}
```





### DexFile.defineClassNative

> JNI入口位置
>
> 可以简单总结为以下几个过程
>
> 1. 将DexFile.mCookie转化为Native DexFile数组
>
> 2. 遍历所有的DexFile
>
>    a. 调用OatDexFile::FindClassDef加载Class
>
>    b. 调用ClassLinker::DefineClass定义Class
>
>    c. 调用InsertDexFileInToClassLoader将DexFile插入到ClassLoader中
>
> 上述过程中的一些方法其实在之前FindClassInBaseDexClassLoader中也有用到。
>
> 所以defineClassNative其实不算是新逻辑。



```c++
static jclass DexFile_defineClassNative(JNIEnv* env,
                                        jclass,
                                        jstring javaName,
                                        jobject javaLoader,
                                        jobject cookie,
                                        jobject dexFile) {
  // 1. 将java的DexFile.mCookie转为dex_files & oat_file  
  std::vector<const DexFile*> dex_files;
  const OatFile* oat_file;
  if (!ConvertJavaArrayToDexFiles(env, cookie, /*out*/ dex_files, /*out*/ oat_file)) {
    VLOG(class_linker) << "Failed to find dex_file";
    DCHECK(env->ExceptionCheck());
    return nullptr;
  }

  // 对class name进行前置判空检查
  ScopedUtfChars class_name(env, javaName);
  if (class_name.c_str() == nullptr) {
    VLOG(class_linker) << "Failed to find class_name";
    return nullptr;
  }
  
  // 创建描述符、计算描述符对应的hash值
  const std::string descriptor(DotToDescriptor(class_name.c_str()));
  const size_t hash(ComputeModifiedUtf8Hash(descriptor.c_str()));
  // 2. 遍历所有的dex_files  
  for (auto& dex_file : dex_files) {
    // a. 尝试从oat, dex中寻找class 
    const dex::ClassDef* dex_class_def =
        OatDexFile::FindClassDef(*dex_file, descriptor.c_str(), hash);
    if (dex_class_def != nullptr) {
      ScopedObjectAccess soa(env);
      ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
      StackHandleScope<1> hs(soa.Self());
      // 将java classloader 解析为 native classloader
      Handle<mirror::ClassLoader> class_loader(
          hs.NewHandle(soa.Decode<mirror::ClassLoader>(javaLoader)));
      // 通过dex_file & class_loader获取dex_cache
      ObjPtr<mirror::DexCache> dex_cache =
          class_linker->RegisterDexFile(*dex_file, class_loader.Get());
      // dex_cache异常，直接丢掉  
      if (dex_cache == nullptr) {
        // OOME or InternalError (dexFile already registered with a different class loader).
        soa.Self()->AssertPendingException();
        return nullptr;
      }
      // b. 定义class  
      ObjPtr<mirror::Class> result = class_linker->DefineClass(soa.Self(),
                                                               descriptor.c_str(),
                                                               hash,
                                                               class_loader,
                                                               *dex_file,
                                                               *dex_class_def);
      // Add the used dex file. This only required for the DexFile.loadClass API since normal
      // class loaders already keep their dex files live.
      // c. 将dexFile插入到classLoader中。  
      class_linker->InsertDexFileInToClassLoader(soa.Decode<mirror::Object>(dexFile),
                                                 class_loader.Get());
      // 返回结果  
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











# QA





## 为什么需要两套类加载机制（Java & Native）



> 眼尖的小伙伴或许能发现我们流程图中有两个类加载的逻辑。

![double-load-class-process.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/double-load-class-process.drawio.png)



> 问题来了，为什么需要设计两套呢？
>
> 我自己结合对代码的理解给出一种解释（不一定对）：

***首先先说结论，出现两套的原因是历史原因，最开始类加载是Java代码 ，后续工程师发现Java代码加载速度比较慢，接着就将一些常用的ClassLoader直接在Native实现了一份以提升性能。因此只有部分我们比较熟知的系统ClassLoader的加载逻辑是走的Native实现，其他的都是走的Java实现。***



类加载代码最早是Java代码这点其实不太准确，准确的来说应该是最早是在java层控制类加载的顺序（先从哪里找然后再从哪里找，最后再......）

第二点java代码的执行速度慢这个问题，其实能从代码的注释中发现, FindClassInPathClassLoader是作为一个fast-path, 只有当classloader是系统熟知的一些ClassLoader(PathClassLoader, InMemoryDexClassLoader, BootClassLoader......)才会走对应的逻辑。如果是自定义的ClassLoader将会走Java逻辑代码。

```c++
static jclass VMClassLoader_findLoadedClass(JNIEnv* env, jclass, jobject javaLoader,
                                            jstring javaName) {
  // .......
 
  // Hard-coded performance optimization: We know that all failed libcore calls to findLoadedClass
  //                                      are followed by a call to the the classloader to actually
  //                                      load the class.
  if (loader != nullptr) {
    // Try the common case.
    StackHandleScope<1> hs(soa.Self());
    c = VMClassLoader::FindClassInPathClassLoader(cl,
                                                  soa,
                                                  soa.Self(),
                                                  descriptor.c_str(),
                                                  descriptor_hash,
                                                  hs.NewHandle(loader));
    if (c != nullptr) {
      return soa.AddLocalReference<jclass>(c);
    }
  }

  // The class wasn't loaded, yet, and our fast-path did not apply (e.g., we didn't understand the
  // classloader chain).
  return nullptr;
    
}
```



## DexElement, DexFile, DexCahe,  ClassTable 是什么

refs: https://juejin.cn/post/7047680282463305735?searchId=202503301638213703DE9FA4841F24C422

1. DexElement是DexFile的Java表示形式(通过JNI持有native引用。)

```c++
// TODO(calin): clean up the unused parameters (here and in libcore).
static jobject DexFile_openDexFileNative(JNIEnv* env,
                                         jclass,
                                         jstring javaSourceName,
                                         [[maybe_unused]] jstring javaOutputName,
                                         [[maybe_unused]] jint flags,
                                         jobject class_loader,
                                         jobjectArray dex_elements) {
  ScopedUtfChars sourceName(env, javaSourceName);
  if (sourceName.c_str() == nullptr) {
    return nullptr;
  }

  if (isReadOnlyJavaDclChecked() && access(sourceName.c_str(), W_OK) == 0) {
    LOG(ERROR) << "Attempt to load writable dex file: " << sourceName.c_str();
    if (isReadOnlyJavaDclEnforced(env)) {
      ScopedLocalRef<jclass> se(env, env->FindClass("java/lang/SecurityException"));
      std::string message(
          StringPrintf("Writable dex file '%s' is not allowed.", sourceName.c_str()));
      env->ThrowNew(se.get(), message.c_str());
      return nullptr;
    }
  }

  std::vector<std::string> error_msgs;
  const OatFile* oat_file = nullptr;
    // 加载dex文件
  std::vector<std::unique_ptr<const DexFile>> dex_files =
      Runtime::Current()->GetOatFileManager().OpenDexFilesFromOat(sourceName.c_str(),
                                                                  class_loader,
                                                                  dex_elements,
                                                                  /*out*/ &oat_file,
                                                                  /*out*/ &error_msgs);
    // 将dexFiles, oat_file转为为handle返回给java层。
  return CreateCookieFromOatFileManagerResult(env, dex_files, oat_file, error_msgs);
}
```



2. DexFile是”dex文件“的对象形式

![image-20250330164133409](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250330164133409.png)

3. DexCache保存dex解析过程中的成员变量，方法，类型，字符串信息。(本文未作讲解)

4. ClassTable是ClassLoader的类缓存表，用于缓存已经加载类，在类加载过程中用于快速返回结果

> ClassLoader.loadClass会通过调用VMClassLoader.findLoadedClass寻找已经加载的所有类的信息。
>
> 而findLoadedClass会调用LookupClass从classTable中寻找当前已经加载的类信息。从而快速返回已加载的类。

![image-20250330164802902](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250330164802902.png)



# 未完待续......



Class加载原理分析
