---
title: window
tags:
cover:
---



# Window





## Window创建(APP进程)



> 调用堆栈

```plaintText
attach:8850, Activity (android.app)
performLaunchActivity:3944, ActivityThread (android.app)
handleLaunchActivity:4173, ActivityThread (android.app)
execute:114, LaunchActivityItem (android.app.servertransaction)
executeNonLifecycleItem:231, TransactionExecutor (android.app.servertransaction)
executeTransactionItems:152, TransactionExecutor (android.app.servertransaction)
execute:93, TransactionExecutor (android.app.servertransaction)
handleMessage:2595, ActivityThread$H (android.app)
dispatchMessage:107, Handler (android.os)
loopOnce:232, Looper (android.os)
loop:317, Looper (android.os)
main:8592, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:580, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:878, ZygoteInit (com.android.internal.os)
```



> 在Activity创建的时候创建Window对象。

```java
// Activity.java

final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
            IBinder shareableActivityToken, IBinder initialCallerInfoAccessToken) {
        // ......

    	// 1. 创建window & 配置window
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(mWindowControllerCallback);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        // ......

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
        mWindow.setPreferMinimalPostProcessing(
                (info.flags & ActivityInfo.FLAG_PREFER_MINIMAL_POST_PROCESSING) != 0);

        getAutofillClientController().onActivityAttached(application);
        setContentCaptureOptions(application.getContentCaptureOptions());

        if (android.security.Flags.contentUriPermissionApis()) {
            mInitialCaller = new ComponentCaller(getActivityToken(), initialCallerInfoAccessToken);
            mCaller = mInitialCaller;
        }
    }
```





## 创建WindowState(WMS进程)



> 调用WindowSession
>
> 将Window注册到WindowManagerService

```plaintText
setView:1476, ViewRootImpl (android.view)
addView:441, WindowManagerGlobal (android.view)
addView:158, WindowManagerImpl (android.view)
handleResumeActivity:5330, ActivityThread (android.app)
execute:57, ResumeActivityItem (android.app.servertransaction)
execute:60, ActivityTransactionItem (android.app.servertransaction)
executeLifecycleItem:282, TransactionExecutor (android.app.servertransaction)
executeTransactionItems:150, TransactionExecutor (android.app.servertransaction)
execute:93, TransactionExecutor (android.app.servertransaction)
handleMessage:2595, ActivityThread$H (android.app)
dispatchMessage:107, Handler (android.os)
loopOnce:232, Looper (android.os)
loop:317, Looper (android.os)
main:8592, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:580, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:878, ZygoteInit (com.android.internal.os)
```

> 调用WindowSession
>
> 将Window注册到WindowManagerService

```java
// ViewRootImpl.java


final W mWindow;
public final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();



public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
 
    // ......
    
    
    res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId,
                            mInsetsController.getRequestedVisibleTypes(), inputChannel, mTempInsets,
                            mTempControls, attachedFrame, compatScale);
    
    // ......
    
    
}
```



> 

```java
// com.android.server.wm.Session	
@Override
    public int addToDisplayAsUser(IWindow window, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, int userId, @InsetsType int requestedVisibleTypes,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl.Array outActiveControls, Rect outAttachedFrame,
            float[] outSizeCompatScale) {
        return mService.addWindow(this, window, attrs, viewVisibility, displayId, userId,
                requestedVisibleTypes, outInputChannel, outInsetsState, outActiveControls,
                outAttachedFrame, outSizeCompatScale);
    }



// com.android.server.wm.WindowManagerService
    public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
            int displayId, int requestUserId, @InsetsType int requestedVisibleTypes,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl.Array outActiveControls, Rect outAttachedFrame,
            float[] outSizeCompatScale) {
        
        
        // ......
        // 1. 创建WindowState实例
        final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], attrs, viewVisibility, session.mUid, userId,
                    session.mCanAddInternalSystemWindow);
        
        // 2. 将WindowState实例添加到WMS缓存列表中
        // mWindowMap存储了所有的客户端的连接，
        // key为客户端IWindow binder对象，value为WindowState.
        win.mSession.onWindowAdded(win);
        mWindowMap.put(client.asBinder(), win);
```



> relayout

```plaintText
relayoutWindow:8897, ViewRootImpl (android.view)
performTraversals:3470, ViewRootImpl (android.view)
doTraversal:2718, ViewRootImpl (android.view)
run:9937, ViewRootImpl$TraversalRunnable (android.view)
run:1406, Choreographer$CallbackRecord (android.view)
run:1415, Choreographer$CallbackRecord (android.view)
doCallbacks:1015, Choreographer (android.view)
doFrame:945, Choreographer (android.view)
run:1389, Choreographer$FrameDisplayEventReceiver (android.view)
handleCallback:959, Handler (android.os)
dispatchMessage:100, Handler (android.os)
loopOnce:232, Looper (android.os)
loop:317, Looper (android.os)
main:8592, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:580, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:878, ZygoteInit (com.android.internal.os)
```





```java
// android.view.ViewRootImpl
private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {
    // ......
     relayoutResult = mWindowSession.relayout(mWindow, params,
                    requestedWidth, requestedHeight, viewVisibility,
                    insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, mRelayoutSeq,
                    mLastSyncSeqId, mTmpFrames, mPendingMergedConfiguration, mSurfaceControl,
                    mTempInsets, mTempControls, mRelayoutBundle);
    // ......
}


// com.android.server.wm.Session
public int relayout(IWindow window, WindowManager.LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewFlags, int flags, int seq,
            int lastSyncSeqId, ClientWindowFrames outFrames,
            MergedConfiguration mergedConfiguration, SurfaceControl outSurfaceControl,
            InsetsState outInsetsState, InsetsSourceControl.Array outActiveControls,
            Bundle outSyncSeqIdBundle) {
        if (false) Slog.d(TAG_WM, ">>>>>> ENTERED relayout from "
                + Binder.getCallingPid());
        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, mRelayoutTag);
        int res = mService.relayoutWindow(this, window, attrs,
                requestedWidth, requestedHeight, viewFlags, flags, seq,
                lastSyncSeqId, outFrames, mergedConfiguration, outSurfaceControl, outInsetsState,
                outActiveControls, outSyncSeqIdBundle);
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        if (false) Slog.d(TAG_WM, "<<<<<< EXITING relayout to "
                + Binder.getCallingPid());
        return res;
    }

// com.android.server.wm.WindowManagerService

public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags, int seq,
            int lastSyncSeqId, ClientWindowFrames outFrames,
            MergedConfiguration outMergedConfiguration, SurfaceControl outSurfaceControl,
            InsetsState outInsetsState, InsetsSourceControl.Array outActiveControls,
            Bundle outSyncIdBundle) {

    // ......
    result = createSurfaceControl(outSurfaceControl, result, win, winAnimator);
    
    // ......	
    
}


// com.android.server.wm.WindowStateAnimator
WindowSurfaceController createSurfaceLocked() {
 
    // ......
    mSurfaceController = new WindowSurfaceController(attrs.getTitle().toString(), format,
                    flags, this, attrs.type);
    
    // ......
    
}


// com.android.server.wm.WindowSurfaceController
WindowSurfaceController(String name, int format, int flags, WindowStateAnimator animator,
            int windowType) {
        mAnimator = animator;

        title = name;

        mService = animator.mService;
        final WindowState win = animator.mWin;
        mWindowType = windowType;
        mWindowSession = win.mSession;

        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "new SurfaceControl");
        mSurfaceControl = win.makeSurface()
                .setParent(win.getSurfaceControl())
                .setName(name)
                .setFormat(format)
                .setFlags(flags)
                .setMetadata(METADATA_WINDOW_TYPE, windowType)
                .setMetadata(METADATA_OWNER_UID, mWindowSession.mUid)
                .setMetadata(METADATA_OWNER_PID, mWindowSession.mPid)
                .setCallsite("WindowSurfaceController")
                .setBLASTLayer().build();

        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    }
```



## 创建SurfaceControl(WMS进程)



> 继续上文的分析流程	

```java
mSurfaceControl = win.makeSurface()
                .setParent(win.getSurfaceControl())
                .setName(name)
                .setFormat(format)
                .setFlags(flags)
                .setMetadata(METADATA_WINDOW_TYPE, windowType)
                .setMetadata(METADATA_OWNER_UID, mWindowSession.mUid)
                .setMetadata(METADATA_OWNER_PID, mWindowSession.mPid)
                .setCallsite("WindowSurfaceController")
                .setBLASTLayer().build();
```



> 调用parent.makeChildSurface
>
> 直到DisplayContent.java

``` java
// WindowContainer
Builder makeSurface() {
        final WindowContainer p = getParent();
        return p.makeChildSurface(this);
    }

Builder makeChildSurface(WindowContainer child) {
        final WindowContainer p = getParent();
        // Give the parent a chance to set properties. In hierarchy v1 we rely
        // on this to set full-screen dimensions on all our Surface-less Layers.
        return p.makeChildSurface(child)
                .setParent(mSurfaceControl);
    }

// DisplayContent
SurfaceControl.Builder makeChildSurface(WindowContainer child) {
    	// 获取SurfaceSession对象 
        SurfaceSession s = child != null ? child.getSession() : getSession();
    	// 利用SurfaceSession创建SurfaceControl.Builder
        final SurfaceControl.Builder b = mWmService.makeSurfaceBuilder(s).setContainerLayer();
        if (child == null) {
            return b;
        }

        // WARNING: it says `mSurfaceControl` below, but this CHANGES meaning after construction!
        // DisplayAreas are added in `configureSurface()` *before* `mSurfaceControl` gets replaced
        // with a wrapper or magnification surface so they end up in the right place; however,
        // anything added or reparented to "the display" *afterwards* needs to be reparented to
        // `getWindowinglayer()` (unless it's an overlay DisplayArea).
        return b.setName(child.getName())
                .setParent(mSurfaceControl);
    }
```



> 调用build创建SurfaceControl native句柄

```java
// android.view.SurfaceControl.Builder 
public SurfaceControl build() {
            if (mWidth < 0 || mHeight < 0) {
                throw new IllegalStateException(
                        "width and height must be positive or unset");
            }
            if ((mWidth > 0 || mHeight > 0) && (isEffectLayer() || isContainerLayer())) {
                throw new IllegalStateException(
                        "Only buffer layers can set a valid buffer size.");
            }

            if (mName == null) {
                Log.w(TAG, "Missing name for SurfaceControl", new Throwable());
            }

            if ((mFlags & FX_SURFACE_MASK) == FX_SURFACE_NORMAL) {
                setBLASTLayer();
            }

            return new SurfaceControl(
                    mSession, mName, mWidth, mHeight, mFormat, mFlags, mParent, mMetadata,
                    mLocalOwnerView, mCallsite);
        }


private SurfaceControl(SurfaceSession session, String name, int w, int h, int format, int flags,
            SurfaceControl parent, SparseIntArray metadata, WeakReference<View> localOwnerView,
            String callsite)
                    throws OutOfResourcesException, IllegalArgumentException {
        if (name == null) {
            throw new IllegalArgumentException("name must not be null");
        }

        mName = name;
        mWidth = w;
        mHeight = h;
        mLocalOwnerView = localOwnerView;
        Parcel metaParcel = Parcel.obtain();
        long nativeObject = 0;
        try {
            if (metadata != null && metadata.size() > 0) {
                metaParcel.writeInt(metadata.size());
                for (int i = 0; i < metadata.size(); ++i) {
                    metaParcel.writeInt(metadata.keyAt(i));
                    metaParcel.writeByteArray(
                            ByteBuffer.allocate(4).order(ByteOrder.nativeOrder())
                                    .putInt(metadata.valueAt(i)).array());
                }
                metaParcel.setDataPosition(0);
            }
            // 调用native方法创建SurfaceControl实例
            nativeObject = nativeCreate(session, name, w, h, format, flags,
                    parent != null ? parent.mNativeObject : 0, metaParcel);
        } finally {
            metaParcel.recycle();
        }
        if (nativeObject == 0) {
            throw new OutOfResourcesException(
                    "Couldn't allocate SurfaceControl native object");
        }
        assignNativeObject(nativeObject, callsite);
    }
```



> JNI代码分析
>
> frameworks/base/core/jni/android_view_SurfaceControl.cpp
>
> 并调用SurfaceComposerClient创建/初始化配置 SurfaceControl对象
>
> 

``` c++
static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
        jstring nameStr, jint w, jint h, jint format, jint flags, jlong parentObject,
        jobject metadataParcel) {
    ScopedUtfChars name(env, nameStr);
    sp<SurfaceComposerClient> client;
    // 1. 通过SurfaceSession获取client对象
    if (sessionObj != NULL) {
        client = android_view_SurfaceSession_getClient(env, sessionObj);
    } else {
        client = SurfaceComposerClient::getDefault();
    }
    // 2. 解析函数参数
    SurfaceControl *parent = reinterpret_cast<SurfaceControl*>(parentObject);
    sp<SurfaceControl> surface;
    LayerMetadata metadata;
    Parcel* parcel = parcelForJavaObject(env, metadataParcel);
    if (parcel && !parcel->objectsCount()) {
        status_t err = metadata.readFromParcel(parcel);
        if (err != NO_ERROR) {
          jniThrowException(env, "java/lang/IllegalArgumentException",
                            "Metadata parcel has wrong format");
        }
    }

    sp<IBinder> parentHandle;
    if (parent != nullptr) {
        parentHandle = parent->getHandle();
    }
	
    status_t err = client->createSurfaceChecked(String8(name.c_str()), w, h, format, &surface,
                                                flags, parentHandle, std::move(metadata));
    if (err == NAME_NOT_FOUND) {
        jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
        return 0;
    } else if (err != NO_ERROR) {
        jniThrowException(env, OutOfResourcesException, statusToString(err).c_str());
        return 0;
    }
	// 返回SurfaceControl native句柄
    surface->incStrong((void *)nativeCreate);
    return reinterpret_cast<jlong>(surface.get());
}
```



> frameworks/native/libs/gui/SurfaceComposerClient.cpp
>
> 1. 通过binder调用Client.cpp创建Surface
> 2. 并创建SurfaceControl native对象，并且将binder调用的结果中的handle,layerId保存入SurfaceControl实例中

```c++
status_t SurfaceComposerClient::createSurfaceChecked(const String8& name, uint32_t w, uint32_t h,
                                                     PixelFormat format,
                                                     sp<SurfaceControl>* outSurface, int32_t flags,
                                                     const sp<IBinder>& parentHandle,
                                                     LayerMetadata metadata,
                                                     uint32_t* outTransformHint) {
    sp<SurfaceControl> sur;
    status_t err = mStatus;

    if (mStatus == NO_ERROR) {
        gui::CreateSurfaceResult result;
        binder::Status status = mClient->createSurface(std::string(name.c_str()), flags,
                                                       parentHandle, std::move(metadata), &result);
        err = statusTFromBinderStatus(status);
        if (outTransformHint) {
            *outTransformHint = result.transformHint;
        }
        ALOGE_IF(err, "SurfaceComposerClient::createSurface error %s", strerror(-err));
        if (err == NO_ERROR) {
            *outSurface = new SurfaceControl(this, result.handle, result.layerId,
                                             toString(result.layerName), w, h, format,
                                             result.transformHint, flags);
        }
    }
    return err;
}
```





## Layer创建(SurfaceFlinger进程)



> frameworks/native/services/surfaceflinger/Client.cpp
>
> 1.调用SurfaceFlinger创建Layer

```c++
binder::Status Client::createSurface(const std::string& name, int32_t flags,
                                     const sp<IBinder>& parent, const gui::LayerMetadata& metadata,
                                     gui::CreateSurfaceResult* outResult) {
    // We rely on createLayer to check permissions.
    sp<IBinder> handle;
    LayerCreationArgs args(mFlinger.get(), sp<Client>::fromExisting(this), name.c_str(),
                           static_cast<uint32_t>(flags), std::move(metadata));
    args.parentHandle = parent;
    const status_t status = mFlinger->createLayer(args, *outResult);
    return binderStatusFromStatusT(status);
}
```



> frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
>
> 1.调用createBufferStateLayer创建Layer对象
>
> 2.调用addClientLayer将La ye

```c++
status_t SurfaceFlinger::createLayer(LayerCreationArgs& args, gui::CreateSurfaceResult& outResult) {
    status_t result = NO_ERROR;

    sp<Layer> layer;

    switch (args.flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceBufferQueue:
        case ISurfaceComposerClient::eFXSurfaceContainer:
        case ISurfaceComposerClient::eFXSurfaceBufferState:
            args.flags |= ISurfaceComposerClient::eNoColorFill;
            [[fallthrough]];
        case ISurfaceComposerClient::eFXSurfaceEffect: {
            result = createBufferStateLayer(args, &outResult.handle, &layer);
            std::atomic<int32_t>* pendingBufferCounter = layer->getPendingBufferCounter();
            if (pendingBufferCounter) {
                std::string counterName = layer->getPendingBufferCounterName();
                mBufferCountTracker.add(outResult.handle->localBinder(), counterName,
                                        pendingBufferCounter);
            }
        } break;
        default:
            result = BAD_VALUE;
            break;
    }

    if (result != NO_ERROR) {
        return result;
    }

    args.addToRoot = args.addToRoot && callingThreadHasUnscopedSurfaceFlingerAccess();
    // We can safely promote the parent layer in binder thread because we have a strong reference
    // to the layer's handle inside this scope.
    sp<Layer> parent = LayerHandle::getLayer(args.parentHandle.promote());
    if (args.parentHandle != nullptr && parent == nullptr) {
        ALOGE("Invalid parent handle %p", args.parentHandle.promote().get());
        args.addToRoot = false;
    }

    uint32_t outTransformHint;
    result = addClientLayer(args, outResult.handle, layer, parent, &outTransformHint);
    if (result != NO_ERROR) {
        return result;
    }

    outResult.transformHint = static_cast<int32_t>(outTransformHint);
    outResult.layerId = layer->sequence;
    outResult.layerName = String16(layer->getDebugName());
    return result;
}
```



## BLASTBufferQueue创建(App进程)



> 由于

```java
// android.view.ViewRootImpl

public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags, int seq,
            int lastSyncSeqId, ClientWindowFrames outFrames,
            MergedConfiguration outMergedConfiguration, SurfaceControl outSurfaceControl,
            InsetsState outInsetsState, InsetsSourceControl.Array outActiveControls,
            Bundle outSyncIdBundle) {

    // ......
    result = createSurfaceControl(outSurfaceControl, result, win, winAnimator);
    
    
    // ......	
    updateBlastSurfaceIfNeeded();
    
}


// 将利用SurfaceControl创建BlastBufferQueue
void updateBlastSurfaceIfNeeded() {
        if (!mSurfaceControl.isValid()) {
            return;
        }
		// 如果SurfaceControl发生变化， 更新BlastBufferQueue
        if (mBlastBufferQueue != null && mBlastBufferQueue.isSameSurfaceControl(mSurfaceControl)) {
            mBlastBufferQueue.update(mSurfaceControl,
                mSurfaceSize.x, mSurfaceSize.y,
                mWindowAttributes.format);
            return;
        }

        // If the SurfaceControl has been updated, destroy and recreate the BBQ to reset the BQ and
        // BBQ states.
        if (mBlastBufferQueue != null) {
            mBlastBufferQueue.destroy();
        }
    	// 创建BlastBufferQueue 
        mBlastBufferQueue = new BLASTBufferQueue(mTag, mSurfaceControl,
                mSurfaceSize.x, mSurfaceSize.y, mWindowAttributes.format);
        mBlastBufferQueue.setTransactionHangCallback(sTransactionHangCallback);
    	// 创建Surface 
        Surface blastSurface = mBlastBufferQueue.createSurface();
        // Only call transferFrom if the surface has changed to prevent inc the generation ID and
        // causing EGL resources to be recreated.
        mSurface.transferFrom(blastSurface);
    }
```



> BLastBufferQueue

``` java
public BLASTBufferQueue(String name, SurfaceControl sc, int width, int height,
            @PixelFormat.Format int format) {
    	// 1. 调用构造函数创建BufferQueue
        this(name, true /* updateDestinationFrame */);
        // 2. 更新
        update(sc, width, height, format);
    }

```



> 1.创建BufferQueue
>
> 初始化了native BufferQueue的成员变量如BBQBufferQueueProducer，BufferQueueConsumer，BLASTBufferItemConsumer

``` java
// android.graphics.BLASTBufferQueue.java
public BLASTBufferQueue(String name, boolean updateDestinationFrame) {
        mNativeObject = nativeCreate(name, updateDestinationFrame);
    }

// frameworks/base/core/jni/android_graphics_BLASTBufferQueue.cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz, jstring jName,
                          jboolean updateDestinationFrame) {
    ScopedUtfChars name(env, jName);
    sp<BLASTBufferQueue> queue = new BLASTBufferQueue(name.c_str(), updateDestinationFrame);
    queue->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(queue.get());
}


BLASTBufferQueue::BLASTBufferQueue(const std::string& name, bool updateDestinationFrame)
      : mSurfaceControl(nullptr),
        mSize(1, 1),
        mRequestedSize(mSize),
        mFormat(PIXEL_FORMAT_RGBA_8888),
        mTransactionReadyCallback(nullptr),
        mSyncTransaction(nullptr),
        mUpdateDestinationFrame(updateDestinationFrame) {
    createBufferQueue(&mProducer, &mConsumer);
    // since the adapter is in the client process, set dequeue timeout
    // explicitly so that dequeueBuffer will block
    mProducer->setDequeueTimeout(std::numeric_limits<int64_t>::max());

    // safe default, most producers are expected to override this
    mProducer->setMaxDequeuedBufferCount(2);
    mBufferItemConsumer = new BLASTBufferItemConsumer(mConsumer,
                                                      GraphicBuffer::USAGE_HW_COMPOSER |
                                                              GraphicBuffer::USAGE_HW_TEXTURE,
                                                      1, false, this);
    static std::atomic<uint32_t> nextId = 0;
    mProducerId = nextId++;
    mName = name + "#" + std::to_string(mProducerId);
    auto consumerName = mName + "(BLAST Consumer)" + std::to_string(mProducerId);
    mQueuedBufferTrace = "QueuedBuffer - " + mName + "BLAST#" + std::to_string(mProducerId);
    mBufferItemConsumer->setName(String8(consumerName.c_str()));
    mBufferItemConsumer->setFrameAvailableListener(this);

    ComposerServiceAIDL::getComposerService()->getMaxAcquiredBufferCount(&mMaxAcquiredBuffers);
    mBufferItemConsumer->setMaxAcquiredBufferCount(mMaxAcquiredBuffers);
    mCurrentMaxAcquiredBufferCount = mMaxAcquiredBuffers;
    mNumAcquired = 0;
    mNumFrameAvailable = 0;

    TransactionCompletedListener::getInstance()->addQueueStallListener(
            [&](const std::string& reason) {
                std::function<void(const std::string&)> callbackCopy;
                {
                    std::unique_lock _lock{mMutex};
                    callbackCopy = mTransactionHangCallback;
                }
                if (callbackCopy) callbackCopy(reason);
            },
            this);

    BQA_LOGV("BLASTBufferQueue created");
}
```



> 2.关联SurfaceControl
>
> 主要是将surfaceControl native handle设置到BLASTBufferQueue的成员中

```java
// android.graphics.BLASTBufferQueue
// 调用jni
public void update(SurfaceControl sc, int width, int height, @PixelFormat.Format int format) {
        nativeUpdate(mNativeObject, sc.mNativeObject, width, height, format);
    }

// frameworks/base/core/jni/android_graphics_BLASTBufferQueue.cpp
// jni方法，内部直接调用BLASTBufferQueue
static void nativeUpdate(JNIEnv* env, jclass clazz, jlong ptr, jlong surfaceControl, jlong width,
                         jlong height, jint format) {
    sp<BLASTBufferQueue> queue = reinterpret_cast<BLASTBufferQueue*>(ptr);
    queue->update(reinterpret_cast<SurfaceControl*>(surfaceControl), width, height, format);
}

// frameworks/native/libs/gui/BLASTBufferQueue.cpp
// 
void BLASTBufferQueue::update(const sp<SurfaceControl>& surface, uint32_t width, uint32_t height,
                              int32_t format) {
    LOG_ALWAYS_FATAL_IF(surface == nullptr, "BLASTBufferQueue: mSurfaceControl must not be NULL");

    std::lock_guard _lock{mMutex};
    if (mFormat != format) {
        mFormat = format;
        mBufferItemConsumer->setDefaultBufferFormat(convertBufferFormat(format));
    }

    const bool surfaceControlChanged = !SurfaceControl::isSameSurface(mSurfaceControl, surface);
    if (surfaceControlChanged && mSurfaceControl != nullptr) {
        BQA_LOGD("Updating SurfaceControl without recreating BBQ");
    }
    bool applyTransaction = false;

    // Always update the native object even though they might have the same layer handle, so we can
    // get the updated transform hint from WM.
    mSurfaceControl = surface;
    SurfaceComposerClient::Transaction t;
    if (surfaceControlChanged) {
        t.setFlags(mSurfaceControl, layer_state_t::eEnableBackpressure,
                   layer_state_t::eEnableBackpressure);
        applyTransaction = true;
    }
    mTransformHint = mSurfaceControl->getTransformHint();
    mBufferItemConsumer->setTransformHint(mTransformHint);
    BQA_LOGV("update width=%d height=%d format=%d mTransformHint=%d", width, height, format,
             mTransformHint);

    ui::Size newSize(width, height);
    if (mRequestedSize != newSize) {
        mRequestedSize.set(newSize);
        mBufferItemConsumer->setDefaultBufferSize(mRequestedSize.width, mRequestedSize.height);
        if (mLastBufferInfo.scalingMode != NATIVE_WINDOW_SCALING_MODE_FREEZE) {
            // If the buffer supports scaling, update the frame immediately since the client may
            // want to scale the existing buffer to the new size.
            mSize = mRequestedSize;
            if (mUpdateDestinationFrame) {
                t.setDestinationFrame(mSurfaceControl, Rect(newSize));
                applyTransaction = true;
            }
        }
    }
    if (applyTransaction) {
        // All transactions on our apply token are one-way. See comment on mAppliedLastTransaction
        t.setApplyToken(mApplyToken).apply(false, true);
    }
}
```





## Surface创建



> 1.利用BufferQueue创建Surface
>
> 2.创建Java Surface对，并将Surface handle实例写入

```java
// android.view.ViewRootImpl
// 调用mBlastBufferQueue.createSurface创建Surfaec对象
void updateBlastSurfaceIfNeeded() {
        if (!mSurfaceControl.isValid()) {
            return;
        }
		// 如果SurfaceControl发生变化， 更新BlastBufferQueue
        if (mBlastBufferQueue != null && mBlastBufferQueue.isSameSurfaceControl(mSurfaceControl)) {
            mBlastBufferQueue.update(mSurfaceControl,
                mSurfaceSize.x, mSurfaceSize.y,
                mWindowAttributes.format);
            return;
        }

        // If the SurfaceControl has been updated, destroy and recreate the BBQ to reset the BQ and
        // BBQ states.
        if (mBlastBufferQueue != null) {
            mBlastBufferQueue.destroy();
        }
    	// 创建BlastBufferQueue 
        mBlastBufferQueue = new BLASTBufferQueue(mTag, mSurfaceControl,
                mSurfaceSize.x, mSurfaceSize.y, mWindowAttributes.format);
        mBlastBufferQueue.setTransactionHangCallback(sTransactionHangCallback);
    	// 创建Surface 
        Surface blastSurface = mBlastBufferQueue.createSurface();
        // Only call transferFrom if the surface has changed to prevent inc the generation ID and
        // causing EGL resources to be recreated.
        mSurface.transferFrom(blastSurface);
    }

// android.graphics.BLASTBufferQueue
// 通过JNI创建surface
public Surface createSurface() {
        return nativeGetSurface(mNativeObject, false /* includeSurfaceControlHandle */);
    }


// frameworks/base/core/jni/android_graphics_BLASTBufferQueue.cpp
// 转接到surface_createFromSurface
static jobject nativeGetSurface(JNIEnv* env, jclass clazz, jlong ptr,
                                jboolean includeSurfaceControlHandle) {
    sp<BLASTBufferQueue> queue = reinterpret_cast<BLASTBufferQueue*>(ptr);
    return android_view_Surface_createFromSurface(env,
                                                  queue->getSurface(includeSurfaceControlHandle));
}


// frameworks/base/core/jni/android_view_Surface.cpp
// 创建Surface对象 & 直接返回
jobject android_view_Surface_createFromSurface(JNIEnv* env, const sp<Surface>& surface) {
    jobject surfaceObj = env->NewObject(gSurfaceClassInfo.clazz,
            gSurfaceClassInfo.ctor, (jlong)surface.get());
    if (surfaceObj == NULL) {
        if (env->ExceptionCheck()) {
            ALOGE("Could not create instance of Surface from IGraphicBufferProducer.");
            LOGE_EX(env);
            env->ExceptionClear();
        }
        return NULL;
    }
    surface->incStrong(&sRefBaseOwner);
    return surfaceObj;
}
```



> 1.利用BufferQueue创建Surface

```C++
sp<Surface> BLASTBufferQueue::getSurface(bool includeSurfaceControlHandle) {
    std::lock_guard _lock{mMutex};
    sp<IBinder> scHandle = nullptr;
    if (includeSurfaceControlHandle && mSurfaceControl) {
        scHandle = mSurfaceControl->getHandle();
    }
    return new BBQSurface(mProducer, true, scHandle, this);
}
```



> 2.创建java对象，并将Surface handle实例写入

```c++
// android.view.ViewRootImpl
// 调用mBlastBufferQueue.createSurface创建Surfaec对象
void updateBlastSurfaceIfNeeded() {
        // ......
    
    	// 创建Surface 
        Surface blastSurface = mBlastBufferQueue.createSurface();
        // Only call transferFrom if the surface has changed to prevent inc the generation ID and
        // causing EGL resources to be recreated.
        mSurface.transferFrom(blastSurface);
    }
```



> 创建Java Surface对象

```c++

// android.graphics.BLASTBufferQueue
// 通过JNI创建surface
public Surface createSurface() {
        return nativeGetSurface(mNativeObject, false /* includeSurfaceControlHandle */);
    }

// frameworks/base/core/jni/android_view_Surface.cpp
// 创建Surface对象 & 直接返回
jobject android_view_Surface_createFromSurface(JNIEnv* env, const sp<Surface>& surface) {
    jobject surfaceObj = env->NewObject(gSurfaceClassInfo.clazz,
            gSurfaceClassInfo.ctor, (jlong)surface.get());
    if (surfaceObj == NULL) {
        if (env->ExceptionCheck()) {
            ALOGE("Could not create instance of Surface from IGraphicBufferProducer.");
            LOGE_EX(env);
            env->ExceptionClear();
        }
        return NULL;
    }
    surface->incStrong(&sRefBaseOwner);
    return surfaceObj;
}
```



> 将Surface handle对象写入

```java
mSurface.transferFrom(blastSurface);

@Deprecated
@UnsupportedAppUsage
public void transferFrom(Surface other) {
    if (other == null) {
        throw new IllegalArgumentException("other must not be null");
    }
    // 将当前surface对象的native指针设置为other的native handle
    // 将other的native handle设置为null。
    if (other != this) {
        final long newPtr;
        synchronized (other.mLock) {
            newPtr = other.mNativeObject;
            other.setNativeObjectLocked(0);
        }

        synchronized (mLock) {
            if (mNativeObject != 0) {
                nativeRelease(mNativeObject);
            }
            setNativeObjectLocked(newPtr);
        }
    }
}
```







# Window数据结构





##  WindowContainer



> Window的直接容器或者间接容器



```java
/**
 * Defines common functionality for classes that can hold windows directly or through their
 * children in a hierarchy form.
 * The test class is {@link WindowContainerTests} which must be kept up-to-date and ran anytime
 * changes are made to this class.
 * 直接包含或者孩子节点包含Window的类型
 */
 
 class WindowContainer<E extends WindowContainer> extends ConfigurationContainer<E>
        implements Comparable<WindowContainer>, Animatable, SurfaceFreezer.Freezable,
        InsetsControlTarget {
            
            
   /**
     * The parent of this window container.
     * For removing or setting new parent {@link #setParent} should be used, because it also
     * performs configuration updates based on new parent's settings.
     */
    private WindowContainer<WindowContainer> mParent = null;
            
            
    // List of children for this window container. List is in z-order as the children appear on
    // screen with the top-most window container at the tail of the list.
    protected final WindowList<E> mChildren = new WindowList<E>();
 
            
            
 }
```





## RootWindowContainer



> Window结构的跟节点

```java
/** Root {@link WindowContainer} for the device. */
class RootWindowContainer extends WindowContainer<DisplayContent>
        implements DisplayManager.DisplayListener {
    
}    


```



> 父类

``` java
/**
 * Root of a {@link DisplayArea} hierarchy. It can be either the {@link DisplayContent} as the root
 * of the whole logical display, or a {@link DisplayAreaGroup} as the root of a partition of the
 * logical display.
 */
class RootDisplayArea extends DisplayArea.Dimmable {
 
     /** {@link Feature} that are supported in this {@link DisplayArea} hierarchy. */
    List<DisplayAreaPolicyBuilder.Feature> mFeatures;
    
}
```







### DisplayContent



> 一块屏幕对于的class, andorid支持多屏幕。印日RootWindowContainer包含多个DisplayContent

```java
/** Root {@link WindowContainer} for the device. */
class DisplayContent extends RootDisplayArea implements WindowManagerPolicy.DisplayContentInfo {
 
    
    
    
    
}

```







## DisplayArea



### TaskDisplayArea

```
/**
 * {@link DisplayArea} that represents a section of a screen that contains app window containers.
 *
 * The children can be either {@link Task} or {@link TaskDisplayArea}.
 */
```



### Dimmable





### Tokens











# WindowState











