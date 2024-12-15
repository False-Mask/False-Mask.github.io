---
title: dex-load-process
date: 2024-12-14 23:43:15
tags:
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/38cfed3e911a43d89c6b9376fbd5a421~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp
---



# pre



对一些环境信息做阐述和说明

1.本文分析的是AOSP android-13.0.0_r78的流程（aosp_redfin_userdebug）

2.本文使用设备为pixel5







# Dex加载过程解析



# install time



> 安装过程中PMS会做一些操作将我们的apk的Manifest进行解析



具体的分析思路如下：

1.准备一个demo app.（其实什么demo都可以）

2.在PMS上打断点

com.android.server.pm.pkg.parsing.ParsingPackageUtils#parsePackage

![image-20241215142720227](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241215142720227.png)

3.安装APP

4.等待断点停下



## PMS解析



> 如下是解析XML的堆栈

```Plaint Text
parsePackage:372, ParsingPackageUtils (com.android.server.pm.pkg.parsing)
parsePackage:168, PackageParser2 (com.android.server.pm.parsing)
parsePackage:150, PackageParser2 (com.android.server.pm.parsing)
getPackageArchiveInfo:624, AppIntegrityManagerServiceImpl (com.android.server.integrity)
handleIntegrityVerification:300, AppIntegrityManagerServiceImpl (com.android.server.integrity)
-$$Nest$mhandleIntegrityVerification:-1, AppIntegrityManagerServiceImpl (com.android.server.integrity)
lambda$onReceive$0:183, AppIntegrityManagerServiceImpl$1 (com.android.server.integrity)
$r8$lambda$D-6BanMaetK6OzqqHwb9xKEKfbs:-1, AppIntegrityManagerServiceImpl$1 (com.android.server.integrity)
run:-1, AppIntegrityManagerServiceImpl$1$$ExternalSyntheticLambda0 (com.android.server.integrity)
handleCallback:942, Handler (android.os)
dispatchMessage:99, Handler (android.os)
loopOnce:201, Looper (android.os)
loop:288, Looper (android.os)
run:67, HandlerThread (android.os)
```



> 解析文件，分了三个case
>
> 1.系统framework-res
>
> 2.输入文件夹（需要解析多个apk）split apk
>
> 3.输入文件单apk

```java
public ParseResult<ParsingPackage> parsePackage(ParseInput input, File packageFile, int flags,
        List<File> frameworkSplits) {
    // 1. 解析framework res文件
    if (((flags & PARSE_FRAMEWORK_RES_SPLITS) != 0)
            && frameworkSplits.size() > 0
            && packageFile.getAbsolutePath().endsWith("/framework-res.apk")) {
        return parseClusterPackage(input, packageFile, frameworkSplits, flags);
    } else if (packageFile.isDirectory()) { // 2. 如果输入是文件夹，则认为是split apk，
        return parseClusterPackage(input, packageFile, /* frameworkSplits= */null, flags);
    } else { // 3. 如果传入的是一个单个文件，则只解析一个apk
        return parseMonolithicPackage(input, packageFile, flags);
    }
}
```

> 解析分为如下过程
>
> 1.简单解析
>
> 2.构建 split 依赖树

```java
private ParseResult<ParsingPackage> parseClusterPackage(ParseInput input, File packageDir,
        List<File> frameworkSplits, int flags) {
    // 1. 简单解析（好像解析的内容也不简单 haha～）
    // 只会解析部分的xml内容其中包含如下
    
    // manifest tag：  
    // installLocation，versionCode，versionCodeMajor，revisionCode，coreApp，isolatedSplits，isFeatureSplit，isSplitRequired，configForSplit
    
    // application tag：
    // debuggable，multiArch，use32bitAbi，extractNativeLibs，useEmbeddedDex，rollbackDataPolicy，permission
    
    // package-verifier：
    // ......
    
    // profileable tag:
    // ......
    
    // receiver tag:
    // ......
    
    // overlay tag：
    // requiredSystemPropertyName，requiredSystemPropertyValue，targetPackage，isStatic，priority
    
    // use-split tag：
    // name
    
    // uses-sdk tag：
    // minSdkVersion， targetSdkVersion
    final ParseResult<PackageLite> liteResult =
                ApkLiteParseUtils.parseClusterPackageLite(input, packageDir, frameworkSplits,
                        liteParseFlags);
    
    
    // 2.构建 split 依赖树
    // Build the split dependency tree.
    SparseArray<int[]> splitDependencies = null;
    final SplitAssetLoader assetLoader;
    if (lite.isIsolatedSplits() && !ArrayUtils.isEmpty(lite.getSplitNames())) {
        try {
            splitDependencies = SplitAssetDependencyLoader.createDependenciesFromPackage(lite);
            assetLoader = new SplitAssetDependencyLoader(lite, splitDependencies, flags);
        } catch (SplitAssetDependencyLoader.IllegalDependencyException e) {
            return input.error(INSTALL_PARSE_FAILED_BAD_MANIFEST, e.getMessage());
        }
    } else {
        assetLoader = new DefaultSplitAssetLoader(lite, flags);
    }
    
}
```





# runtime



> 运行期间就使用tiktok为例做一次dex加载的分析。



调试(需要userdebug的包)

1.设置TikTok为debug状态

```shell
adb shell am set-debug-app -w com.ss.android.ugc.trill
```

2.打断点

可以将断点打在dalvik.system.DexFile#openDexFile（看整体的运行逻辑）

3.attach Process



> 如下是断点停下来后的，函数调用栈

```shell
openDexFile:371, DexFile (dalvik.system)
<init>:113, DexFile (dalvik.system)
<init>:86, DexFile (dalvik.system)
loadDexFile:438, DexPathList (dalvik.system)
makeDexElements:397, DexPathList (dalvik.system)
<init>:166, DexPathList (dalvik.system)
<init>:160, BaseDexClassLoader (dalvik.system)
<init>:130, BaseDexClassLoader (dalvik.system)
<init>:146, PathClassLoader (dalvik.system)
createClassLoader:93, ClassLoaderFactory (com.android.internal.os)
createClassLoader:134, ClassLoaderFactory (com.android.internal.os)
getClassLoader:126, ApplicationLoaders (android.app)
getClassLoaderWithSharedLibraries:61, ApplicationLoaders (android.app)
getSharedLibraryClassLoaderWithSharedLibraries:89, ApplicationLoaders (android.app)
createSharedLibraryLoader:745, LoadedApk (android.app)
createSharedLibrariesLoaders:792, LoadedApk (android.app)
createOrUpdateClassLoaderLocked:1022, LoadedApk (android.app)
getClassLoader:1126, LoadedApk (android.app)
getResources:1374, LoadedApk (android.app)
createAppContext:3090, ContextImpl (android.app)
createAppContext:3082, ContextImpl (android.app)
handleBindApplication:6696, ActivityThread (android.app)
-$$Nest$mhandleBindApplication:-1, ActivityThread (android.app)
handleMessage:2132, ActivityThread$H (android.app)
dispatchMessage:106, Handler (android.os)
loopOnce:201, Looper (android.os)
loop:288, Looper (android.os)
main:7918, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:548, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:936, ZygoteInit (com.android.internal.os)
```



> 简单分析了下调用栈，发现：
>
> 1.openDexFile是由new PathClassLoader -> new DexPathList -> DexPathList.makeDexElements一层层调用。
>
> 2.Android的ClassLoader加载有两个过程，首先是加载系统共享库ClassLoader,再则才是App Apk的ClassLoader

```java
// 1. makeDexElements过程（其实也就是针对于安装路径的所有apk做openDexFile操作，并将结果保存到DexFile中）

private static Element[] makeDexElements(List<File> files, File optimizedDirectory,
        List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
  Element[] elements = new Element[files.size()];
  int elementsPos = 0;
  /*
   * Open all files and load the (direct or contained) dex files up front.
   */
  for (File file : files) {
      if (file.isDirectory()) {
          // element path 为 文件夹
          // ......
      } else if (file.isFile()) {
          String name = file.getName();

          DexFile dex = null;
          if (name.endsWith(DEX_SUFFIX)) {
              // end with .dex
              // .....
          } else {
              // 加载dex并保存在数组中
              try {
                  dex = loadDexFile(file, optimizedDirectory, loader, elements);
              } catch (IOException suppressed) {
                  /*
                   * IOException might get thrown "legitimately" by the DexFile constructor if
                   * the zip file turns out to be resource-only (that is, no classes.dex file
                   * in it).
                   * Let dex == null and hang on to the exception to add to the tea-leaves for
                   * when findClass returns null.
                   */
                  suppressedExceptions.add(suppressed);
              }

              if (dex == null) {
                  elements[elementsPos++] = new Element(file);
              } else {
                  elements[elementsPos++] = new Element(dex, file);
              }
          }
          
      }
  }
  return elements;
}


// 2.ClassLoader加载过程
@GuardedBy("mLock")
private void createOrUpdateClassLoaderLocked(List<String> addedPaths) {

    // 创建系统的classloader，pair的低一个参数是标准的共享库
    // 第二个参数是app dex加载完之后需要加载的库
    //  the first is for standard shared libraries
    //  and the second is list for shared libraries that code should be loaded after the dex
    Pair<List<ClassLoader>, List<ClassLoader>> sharedLibraries =
                createSharedLibrariesLoaders(mApplicationInfo.sharedLibraryInfos, isBundledApp,
                        librarySearchPath, libraryPermittedPath);


    // 创建app自身的classloader
    mDefaultClassLoader = ApplicationLoaders.getDefault().getClassLoaderWithSharedLibraries(
                zip, mApplicationInfo.targetSdkVersion, isBundledApp, librarySearchPath,
                libraryPermittedPath, mBaseClassLoader,
                mApplicationInfo.classLoaderName, sharedLibraries.first, nativeSharedLibraries,
                sharedLibraries.second);
    
    // 创建ClassLoader
    if (mClassLoader == null) {
            mClassLoader = mAppComponentFactory.instantiateClassLoader(mDefaultClassLoader,
                    new ApplicationInfo(mApplicationInfo));
    }
        
        
        
}

```



## shared library



> 核心代码如下

```java
private Pair<List<ClassLoader>, List<ClassLoader>> createSharedLibrariesLoaders(
    	// sharedLibrary是由PMS解析通过IPC传入的。
        List<SharedLibraryInfo> sharedLibraries,
        boolean isBundledApp, String librarySearchPath, String libraryPermittedPath) {
    if (sharedLibraries == null || sharedLibraries.isEmpty()) {
        return new Pair<>(null, null);
    }

    // if configured to do so, shared libs are split into 2 collections: those that are
    // on the class path before the applications code, which is standard, and those
    // specified to be loaded after the applications code.
    HashSet<String> libsToLoadAfter = new HashSet<>();
    Resources systemR = Resources.getSystem();
    Collections.addAll(libsToLoadAfter, systemR.getStringArray(
            R.array.config_sharedLibrariesLoadedAfterApp));

    // 需要在app apk加载之前加载的库集合
    // 通常是5个
    // 1. /system/framework/android.test.base.jar
    // 2. /system/framework/android.test.mock.jar
    // 3. /system/framework/org.apache.http.legacy.jar
    // 4. /system/framework/android.test.runner.jar
    // 5. libOpenCL-pixel.so
    List<ClassLoader> loaders = new ArrayList<>();
    // 需要在app apk加载之后加载的库集合。通常是空的。
    List<ClassLoader> after = new ArrayList<>();
    for (SharedLibraryInfo info : sharedLibraries) {
        if (info.isNative()) {
            // Native shared lib doesn't contribute to the native lib search path. Its name is
            // sent to libnativeloader and then the native shared lib is exported from the
            // default linker namespace.
            continue;
        }
        if (info.isSdk()) {
            // SDKs are not loaded automatically.
            continue;
        }
        if (libsToLoadAfter.contains(info.getName())) {
            if (DEBUG) {
                Slog.v(ActivityThread.TAG,
                        info.getName() + " will be loaded after application code");
            }
            after.add(createSharedLibraryLoader(
                    info, isBundledApp, librarySearchPath, libraryPermittedPath));
        } else {
            loaders.add(createSharedLibraryLoader(
                    info, isBundledApp, librarySearchPath, libraryPermittedPath));
        }
    }
    return new Pair<>(loaders, after);
}
```



createSharedLibararyLoader过程如下: 

其实很好理解，就只是创建了一个ClassLoader,ClassLoader在创建的同时会做一系列的LoadDex操作。（通过JNI处理）

![load-shared-lib.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/load-shared-lib.drawio.png)





##  load app dex



> 堆栈如下

```java
openDexFile:371, DexFile (dalvik.system)
<init>:113, DexFile (dalvik.system)
<init>:86, DexFile (dalvik.system)
loadDexFile:438, DexPathList (dalvik.system)
makeDexElements:397, DexPathList (dalvik.system)
<init>:166, DexPathList (dalvik.system)
<init>:160, BaseDexClassLoader (dalvik.system)
<init>:130, BaseDexClassLoader (dalvik.system)
<init>:146, PathClassLoader (dalvik.system)
createClassLoader:93, ClassLoaderFactory (com.android.internal.os)
createClassLoader:134, ClassLoaderFactory (com.android.internal.os)
getClassLoader:126, ApplicationLoaders (android.app)
getClassLoaderWithSharedLibraries:61, ApplicationLoaders (android.app)
createOrUpdateClassLoaderLocked:1034, LoadedApk (android.app)
getClassLoader:1126, LoadedApk (android.app)
getResources:1374, LoadedApk (android.app)
createAppContext:3090, ContextImpl (android.app)
createAppContext:3082, ContextImpl (android.app)
handleBindApplication:6696, ActivityThread (android.app)
-$$Nest$mhandleBindApplication:-1, ActivityThread (android.app)
handleMessage:2132, ActivityThread$H (android.app)
dispatchMessage:106, Handler (android.os)
loopOnce:201, Looper (android.os)
loop:288, Looper (android.os)
main:7918, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:548, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:936, ZygoteInit (com.android.internal.os)
```



> load操作起点如下

```java
mDefaultClassLoader = ApplicationLoaders.getDefault().getClassLoaderWithSharedLibraries(
        zip, mApplicationInfo.targetSdkVersion, isBundledApp, librarySearchPath,
        libraryPermittedPath, mBaseClassLoader,
        mApplicationInfo.classLoaderName, sharedLibraries.first, nativeSharedLibraries,
        sharedLibraries.second);
mAppComponentFactory = createAppFactory(mApplicationInfo, mDefaultClassLoader);
```



> 可以发现从一开始的函数调用就和shared library一样了.
>
> 后续流程就不帖了，很明确的是他们的流程是一样的

```java
 ClassLoader getClassLoaderWithSharedLibraries(
            String zip, int targetSdkVersion, boolean isBundled,
            String librarySearchPath, String libraryPermittedPath,
            ClassLoader parent, String classLoaderName,
            List<ClassLoader> sharedLibraries, List<String> nativeSharedLibraries,
            List<ClassLoader> sharedLibrariesLoadedAfterApp) {
        // For normal usage the cache key used is the same as the zip path.
        return getClassLoader(zip, targetSdkVersion, isBundled, librarySearchPath,
                              libraryPermittedPath, parent, zip, classLoaderName, sharedLibraries,
                              nativeSharedLibraries, sharedLibrariesLoadedAfterApp);
    }
```





## QA





### 系统怎么知道Split Apk有那些的？



> 其实很关键的一点就是path.
>
> app apk加载的时候会有一个很长的zip string（他是app所有apk的集合）

```java
mDefaultClassLoader = ApplicationLoaders.getDefault().getClassLoaderWithSharedLibraries(
        zip, mApplicationInfo.targetSdkVersion, isBundledApp, librarySearchPath,
        libraryPermittedPath, mBaseClassLoader,
        mApplicationInfo.classLoaderName, sharedLibraries.first, nativeSharedLibraries,
        sharedLibraries.second);
```



> zip string其实是是makePath时创建的，其中是由base.apk + split apk路径集合拼接而来
>
> base apk很好理解，就是base.apk的路径
>
> split的apk是如何解析的呢？

```java
makePaths(mActivityThread, isBundledApp, mApplicationInfo, zipPaths, libPaths);

public static void makePaths(ActivityThread activityThread,
                                 boolean isBundledApp,
                                 ApplicationInfo aInfo,
                                 List<String> outZipPaths,
                                 List<String> outLibPaths) {
    
    // base apk
     outZipPaths.add(appDir);
 
    // split apk
     // Do not load all available splits if the app requested isolated split loading.
     if (aInfo.splitSourceDirs != null && !aInfo.requestsIsolatedSplitLoading()) {
         Collections.addAll(outZipPaths, aInfo.splitSourceDirs);
     }
    
    
}
```





> 其实很简单就是在PMS解析的apk的时候会通过baseDir.listFiles寻找路径下的所有apk

android.content.pm.parsing.ApkLiteParseUtils#parseClusterPackageLite

```java
 public static ParseResult<PackageLite> parseClusterPackageLite(ParseInput input,
            File packageDirOrApk, List<File> frameworkSplits, int flags) {

	// ......
	
    // get all apk list
	files = packageDirOrApk.listFiles();
	
	  for (File file : files) {
          // parse all apks
            if (isApkFile(file)) {
                final ParseResult<ApkLite> result = parseApkLite(input, file, flags);
                // ......
                final ApkLite lite = result.getResult();
                // ......
                apks.put(lite.getSplitName(), lite) != null
            }

	  }
            
}
```



> 最后回到问题上
>
> 系统是根据base.apk的路径，通过listFiles过滤*.apk的所有文件。（简单来说split.apk和base.apk处于同一路径）





### Split Apk总是在base.apk的时候加载，有办法让他不加载吗



> 先说答案，可以。



> 首先我们回顾一下问题“系统怎么知道Split Apk有那些的？“
>
> Split Apk路径是由PMS通过listFiles再通过IPC传递给app进程。
>
> app进程在通过makePaths决定app apk需要加载哪些。



> 我们再仔细看一下makePaths方法

```java
makePaths(mActivityThread, isBundledApp, mApplicationInfo, zipPaths, libPaths);

public static void makePaths(ActivityThread activityThread,
                                 boolean isBundledApp,
                                 ApplicationInfo aInfo,
                                 List<String> outZipPaths,
                                 List<String> outLibPaths) {
    
    // base apk
     outZipPaths.add(appDir);
 
    // split apk
     // Do not load all available splits if the app requested isolated split loading.
     if (aInfo.splitSourceDirs != null && !aInfo.requestsIsolatedSplitLoading()) {
         Collections.addAll(outZipPaths, aInfo.splitSourceDirs);
     }
    
    
}
```



> 发现了吗？add Split Apk之前有一个判断。
>
> aInfo.splitSourceDirs != null && !aInfo.requestsIsolatedSplitLoading()。
>
> 其中splitSourceDirs在有split apk的情况下一般不会为null的。
>
> 但是aInfo.requestsIsolatedSplitLoading就不一定了。
>
> 也就是requestsIsolatedSplitLoading可以控制是否加载split apk
>
> 如果这里是true那么。split apk就不会添加



> 最后经过简单的搜索和分析我们得知isolated split loading可以通过
>
> isolatedSplits=true设置

```java
 public boolean requestsIsolatedSplitLoading() {
        return (privateFlags & ApplicationInfo.PRIVATE_FLAG_ISOLATED_SPLIT_LOADING) != 0;
 }


final boolean isolatedSplits = sa.getBoolean(
        com.android.internal.R.styleable.AndroidManifest_isolatedSplits, false);
if (isolatedSplits) {
    pkg.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_ISOLATED_SPLIT_LOADING;
}
```



> 具体isolatedSplits是干嘛的，就不细致分析了

感兴趣的可见[Chrome通过Isolated Splits优化启动耗时](https://chromium.googlesource.com/chromium/src/+/main/docs/android_isolated_splits.md)



# 总结





1.Dex加载其中需要两部分结合：PMS解析Manifest + 运行时信息读取

2.其中PMS的解析主要发生在安装期间，他会针对于Manifest做一个精简的解析

3.运行期间PMS会将ApplicationInfo进行详细解析并通过Binder传递给App进程

4.app运行期间会依据ApplicationInfo中的信息进行Dex的加载

5.Dex的加载分为两个过程：一是shared Library（系统库），二是app apk的加载。

