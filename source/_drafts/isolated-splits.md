---
title: Isolated Splits原理剖析
tags:
cover:
---



# Isolated Split 原理剖析 



# 起源



最开始了解Isolated Splits是在[Chromium Docs](https://chromium.googlesource.com/chromium/src/+/main/docs/android_isolated_splits.md)了解到的。（篇幅很短）

一句话总结就是——减少了应用冷启耗时

![image-20241222232056777](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241222232056777.png)





# API分析



## createContextForSplit



Trigger:

1. handleReceiver
2. handleCreateService
3. installProvider

作用：

1.为split创建独立的context对象

2.classloader是独立的

3.resource是独立的



- 核心流程

```java
public Context createContextForSplit(String splitName) throws NameNotFoundException {
    if (!mPackageInfo.getApplicationInfo().requestsIsolatedSplitLoading()) {
        // All Splits are always loaded.
        return this;
    }
	// 获取classloader
    final ClassLoader classLoader = mPackageInfo.getSplitClassLoader(splitName);
    final String[] paths = mPackageInfo.getSplitPaths(splitName);
	// 创建context
    final ContextImpl context = new ContextImpl(this, mMainThread, mPackageInfo, mParams,
            mAttributionSource.getAttributionTag(),
            mAttributionSource.getNext(),
            splitName, mToken, mUser, mFlags, classLoader, null);
	// 获取 & 设置resource
    context.setResources(ResourcesManager.getInstance().getResources(
            mToken,
            mPackageInfo.getResDir(),
            paths,
            mPackageInfo.getOverlayDirs(),
            mPackageInfo.getOverlayPaths(),
            mPackageInfo.getApplicationInfo().sharedLibraryFiles,
            mForceDisplayOverrideInResources ? getDisplayId() : null,
            null,
            mPackageInfo.getCompatibilityInfo(),
            classLoader,
            mResources.getLoaders()));
    return context;
}
```

- classloader创建

```java
// LoadedApk.java
ClassLoader getSplitClassLoader(String splitName) throws NameNotFoundException {
    // 正常case下mSplitLoader不为null.
    // 除非 aInfo.requestsIsolatedSplitLoading() && !ArrayUtils.isEmpty(mSplitNames) == false
    // if (aInfo.requestsIsolatedSplitLoading() && !ArrayUtils.isEmpty(mSplitNames)) {
    //        mSplitLoader = new SplitDependencyLoaderImpl(aInfo.splitDependencies);
    //    }
    if (mSplitLoader == null) {
        return mClassLoader;
    }
    return mSplitLoader.getClassLoaderForSplit(splitName);
}

// SplitDependencyLoaderImpl.java
 ClassLoader getClassLoaderForSplit(String splitName) throws NameNotFoundException {
     // 加载Split.apk
    final int idx = ensureSplitLoaded(splitName);
    synchronized (mLock) {
        // 从缓存中获取classloader
        return mCachedClassLoaders[idx];
    }
 }

// 创建split loader
private int ensureSplitLoaded(String splitName) throws NameNotFoundException {
    int idx = 0;
    if (splitName != null) {
        // 搜索split 位置
        idx = Arrays.binarySearch(mSplitNames, splitName);
        if (idx < 0) {
            throw new PackageManager.NameNotFoundException(
                    "Split name '" + splitName + "' is not installed");
        }
        idx += 1;
    }
    loadDependenciesForSplit(idx);
    return idx;
}

protected void loadDependenciesForSplit(@IntRange(from = 0) int splitIdx) throws E {
        // Quick check before any allocations are done.
        if (isSplitCached(splitIdx)) {
            return;
        }

        // Special case the base, since it has no dependencies.
        if (splitIdx == 0) {
            final int[] configSplitIndices = collectConfigSplitIndices(0);
            constructSplit(0, configSplitIndices, -1);
            return;
        }

        // Build up the dependency hierarchy.
        final IntArray linearDependencies = new IntArray();
        linearDependencies.add(splitIdx);

        // Collect all the dependencies that need to be constructed.
        // They will be listed from leaf to root.
        while (true) {
            // Only follow the first index into the array. The others are config splits and
            // get loaded with the split.
            final int[] deps = mDependencies.get(splitIdx);
            if (deps != null && deps.length > 0) {
                splitIdx = deps[0];
            } else {
                splitIdx = -1;
            }

            if (splitIdx < 0 || isSplitCached(splitIdx)) {
                break;
            }

            linearDependencies.add(splitIdx);
        }

        // Visit each index, from right to left (root to leaf).
        int parentIdx = splitIdx;
        for (int i = linearDependencies.size() - 1; i >= 0; i--) {
            final int idx = linearDependencies.get(i);
            // 计算dependencies 
            final int[] configSplitIndices = collectConfigSplitIndices(idx);
            // 构造classloader
            constructSplit(idx, configSplitIndices, parentIdx);
            parentIdx = idx;
        }
    }
```





- resource创建

```java
// ContextImpl.java
context.setResources(ResourcesManager.getInstance().getResources(
        mToken, // 不确定
        mPackageInfo.getResDir(), // 看样子是base包的位置
        paths, // 资源路径
        mPackageInfo.getOverlayDirs(), // overlay 文件路径
        mPackageInfo.getOverlayPaths(), // overlay 路径
        mPackageInfo.getApplicationInfo().sharedLibraryFiles, // 共享库路径
        mForceDisplayOverrideInResources ? getDisplayId() : null,
        null,
        mPackageInfo.getCompatibilityInfo(), // 屏幕信息
        classLoader, // classloader
        mResources.getLoaders() // resource loader
        ));


// ResourcesManager.java
public Resources getResources(
            @Nullable IBinder activityToken,
            @Nullable String resDir,
            @Nullable String[] splitResDirs,
            @Nullable String[] legacyOverlayDirs,
            @Nullable String[] overlayPaths,
            @Nullable String[] libDirs,
            @Nullable Integer overrideDisplayId,
            @Nullable Configuration overrideConfig,
            @NonNull CompatibilityInfo compatInfo,
            @Nullable ClassLoader classLoader,
            @Nullable List<ResourcesLoader> loaders) {
        try {
            Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "ResourcesManager#getResources");
            final ResourcesKey key = new ResourcesKey(
                    resDir,
                    splitResDirs,
                    combinedOverlayPaths(legacyOverlayDirs, overlayPaths),
                    libDirs,
                    overrideDisplayId != null ? overrideDisplayId : INVALID_DISPLAY,
                    overrideConfig,
                    compatInfo,
                    loaders == null ? null : loaders.toArray(new ResourcesLoader[0]));
            classLoader = classLoader != null ? classLoader : ClassLoader.getSystemClassLoader();

            // Preload the ApkAssets required by the key to prevent performing heavy I/O while the
            // ResourcesManager lock is held.
            final ApkAssetsSupplier assetsSupplier = createApkAssetsSupplierNotLocked(key);

            if (overrideDisplayId != null) {
                rebaseKeyForDisplay(key, overrideDisplayId);
            }

            Resources resources;
            if (activityToken != null) {
                Configuration initialOverrideConfig = new Configuration(key.mOverrideConfiguration);
                rebaseKeyForActivity(activityToken, key, overrideDisplayId != null);
                resources = createResourcesForActivity(activityToken, key, initialOverrideConfig,
                        overrideDisplayId, classLoader, assetsSupplier);
            } else {
                resources = createResources(key, classLoader, assetsSupplier);
            }
            return resources;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
        }
    }
```



## other



### ContentProvider

> 1.如果没有开启Isolate模式，基本上**不会调用到createContextForSplit**，ContentProvider实例默认使用**ApplicationContext ClassLoader反射创建**
>
> 2.如果开启Isolate模式
>
> ​	a.provider只在split apk manifest中声明（虽然默认会合并到base.apk）那么会调用createContextForSplit创建Context.
>
> ​	b.如果provider在base.apk中申明，那么默认不会调用createContextForSplit,创建等流程使用Application Context ClassLoader反射。但是大概率会ClassNotFound.（如果没有做其他特殊处理）

```java
private ContentProviderHolder installProvider(Context context,
        ContentProviderHolder holder, ProviderInfo info,
        boolean noisy, boolean noReleaseNeeded, boolean stable) {
    ContentProvider localProvider = null;
    IContentProvider provider;

     // a
     // 如果contentProvider不是在base.apk 中申明
     if (info.splitName != null) {
            try {
                // 会单独创建一个Context
                // 包含ClassLoader & Resource
                c = c.createContextForSplit(info.splitName);
            } catch (NameNotFoundException e) {
                throw new RuntimeException(e);
            }
       }
        if (info.attributionTags != null && info.attributionTags.length > 0) {
            final String attributionTag = info.attributionTags[0];
            c = c.createAttributionContext(attributionTag);
        }

        try {
            final java.lang.ClassLoader cl = c.getClassLoader();
            LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
            if (packageInfo == null) {
                // System startup case.
                packageInfo = getSystemContext().mPackageInfo;
            }
            // 如果a处分支没有进入，则通过application Context对应的ClassLoader反射创建Provider实例。（isolated模式没做特殊处理可能会ClassNotFound）
            // 如果a处分支进入，则通过isolated context对应的ClassLoader反射创建Provider实例。（一般不会ClassNotFound）
            localProvider = packageInfo.getAppFactory()
                    .instantiateProvider(cl, info.name);
            provider = localProvider.getIContentProvider();
            if (provider == null) {
                Slog.e(TAG, "Failed to instantiate class " +
                      info.name + " from sourceDir " +
                      info.applicationInfo.sourceDir);
                return null;
            }
            if (DEBUG_PROVIDER) Slog.v(
                TAG, "Instantiating local provider " + info.name);
            // XXX Need to create the correct context for this provider.
            localProvider.attachInfo(c, info);
        } 
    } 

	// ......


    return retHolder;
}
```





### activity



> 如下可看出，针对于isolatedSplit,activity会单独获取一个ClassLoader & splitDirs

```java
static ContextImpl createActivityContext(ActivityThread mainThread,
        LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
        Configuration overrideConfiguration) {

    
    String[] splitDirs = packageInfo.getSplitResDirs();
    ClassLoader classLoader = packageInfo.getClassLoader();

    if (packageInfo.getApplicationInfo().requestsIsolatedSplitLoading()) {
        Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "SplitDependencies");
        try {
            classLoader = packageInfo.getSplitClassLoader(activityInfo.splitName);
            splitDirs = packageInfo.getSplitPaths(activityInfo.splitName);
        } catch (NameNotFoundException e) {
            // Nothing above us can handle a NameNotFoundException, better crash.
            throw new RuntimeException(e);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
        }
    }
    
    // ......
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, ContextParams.EMPTY,
                attributionTag, null, activityInfo.splitName, activityToken, null, 0, classLoader,
                null);
    
    // ......
    context.setResources(resourcesManager.createBaseTokenResources(activityToken,
                packageInfo.getResDir(),
                splitDirs,
                packageInfo.getOverlayDirs(),
                packageInfo.getOverlayPaths(),
                packageInfo.getApplicationInfo().sharedLibraryFiles,
                displayId,
                overrideConfiguration,
                compatInfo,
                classLoader,
                packageInfo.getApplication() == null ? null
                        : packageInfo.getApplication().getResources().getLoaders()));
        context.mDisplay = resourcesManager.getAdjustedDisplay(displayId,
                context.getResources());
    
    
}
```







### service



> 同installProvider
>
> 只是额外创建了split ClassLoader

```java
private void handleCreateService(CreateServiceData data) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

        Application app = packageInfo.makeApplicationInner(false, mInstrumentation);

        final java.lang.ClassLoader cl;
        // 如果split不为null则为split创建单独的classLoader
        if (data.info.splitName != null) {
            cl = packageInfo.getSplitClassLoader(data.info.splitName);
        } else {
            cl = packageInfo.getClassLoader();
        }
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
        ContextImpl context = ContextImpl.getImpl(service
                .createServiceBaseContext(this, packageInfo));
        // 如果service Split不为null,则创建新的Context
        if (data.info.splitName != null) {
            context = (ContextImpl) context.createContextForSplit(data.info.splitName);
        }
        if (data.info.attributionTags != null && data.info.attributionTags.length > 0) {
            final String attributionTag = data.info.attributionTags[0];
            context = (ContextImpl) context.createAttributionContext(attributionTag);
        }
        // Service resources must be initialized with the same loaders as the application
        // context.
        context.getResources().addLoaders(
                app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

        context.setOuterContext(service);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        service.onCreate();
        mServicesData.put(data.token, data);
        mServices.put(data.token, service);
        try {
            ActivityManager.getService().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(service, e)) {
            throw new RuntimeException(
                "Unable to create service " + data.info.name
                + ": " + e.toString(), e);
        }
    }
}
```





### Receiver



> 同installProvider

```java
private void handleReceiver(ReceiverData data) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();

    String component = data.intent.getComponent().getClassName();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);

    IActivityManager mgr = ActivityManager.getService();

    Application app;
    BroadcastReceiver receiver;
    ContextImpl context;
    try {
        app = packageInfo.makeApplicationInner(false, mInstrumentation);
        context = (ContextImpl) app.getBaseContext();
        // 如果splitName不为null,则为split创建单独的context
        if (data.info.splitName != null) {
            context = (ContextImpl) context.createContextForSplit(data.info.splitName);
        }
        if (data.info.attributionTags != null && data.info.attributionTags.length > 0) {
            final String attributionTag = data.info.attributionTags[0];
            context = (ContextImpl) context.createAttributionContext(attributionTag);
        }
        java.lang.ClassLoader cl = context.getClassLoader();
        data.intent.setExtrasClassLoader(cl);
        data.intent.prepareToEnterProcess(
                isProtectedComponent(data.info) || isProtectedBroadcast(data.intent),
                context.getAttributionSource());
        data.setExtrasClassLoader(cl);
        receiver = packageInfo.getAppFactory()
                .instantiateReceiver(cl, data.info.name, data.intent);
    } catch (Exception e) {
        if (DEBUG_BROADCAST) Slog.i(TAG,
                "Finishing failed broadcast to " + data.intent.getComponent());
        data.sendFinished(mgr);
        throw new RuntimeException(
            "Unable to instantiate receiver " + component
            + ": " + e.toString(), e);
    }

    try {
        if (localLOGV) Slog.v(
            TAG, "Performing receive of " + data.intent
            + ": app=" + app
            + ", appName=" + app.getPackageName()
            + ", pkg=" + packageInfo.getPackageName()
            + ", comp=" + data.intent.getComponent().toShortString()
            + ", dir=" + packageInfo.getAppDir());

        sCurrentBroadcastIntent.set(data.intent);
        receiver.setPendingResult(data);
        receiver.onReceive(context.getReceiverRestrictedContext(),
                data.intent);
    } catch (Exception e) {
        if (DEBUG_BROADCAST) Slog.i(TAG,
                "Finishing failed broadcast to " + data.intent.getComponent());
        data.sendFinished(mgr);
        if (!mInstrumentation.onException(receiver, e)) {
            throw new RuntimeException(
                "Unable to start receiver " + component
                + ": " + e.toString(), e);
        }
    } finally {
        sCurrentBroadcastIntent.set(null);
    }

    if (receiver.getPendingResult() != null) {
        data.finish();
    }
}
```







# refs



[Chromium Docs Isolated Splits](https://chromium.googlesource.com/chromium/src/+/main/docs/android_isolated_splits.md)

