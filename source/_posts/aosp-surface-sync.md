---
title: AOSP SurfaceSyncGroup
tags:
  - aosp
  - SurfaceView
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-SurfaceSyncGroup.drawio.png
date: 2026-02-22 21:02:36
---

# Android SurfaceSyncGroup

![aosp-SurfaceSyncGroup.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-SurfaceSyncGroup.drawio.png)

# 关于

本文主要是用于基于 Android14.0.0_r73 用于分析 SurfaceSyncGroup 的逻辑

1.ViewRootImpl作为 Root，SurfaceView 作为 child
2.Root 接受所有ready signal，最后将所有的Transaction 进行合并进行原子提交
3.通过 add 方法添加pending syncs数目，构建Sync 树结构
4.调用markSyncReady减少pending syncs 数目

# SurfaceSyncGroup 案例分析

[SurfaceViewSyncTest案例分析](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r73:frameworks/base/tests/SurfaceViewSyncTest/src/com/android/test/SurfaceViewSyncActivity.java)

核心逻辑如下：
```java
// 1. 创建 SurfaceSyncGroup
new SurfaceSyncGroup(TAG);
mSyncGroup.add(container.getRootSurfaceControl(), null /* runnable */);

// 2. 添加 SurfaceView 作为 child sync group
mSyncGroup.add(mSurfaceView, frameCallback ->
                    mRenderingThread.renderFrame(frameCallback, width, height));
                    
// 3. 将主线程的VRI 的 SurfaceControl 添加到 SyncGroup 中
mSyncGroup.add(container.getRootSurfaceControl(), null /* runnable */);

// 4. 标记完成
x.markSyncReady();
xx.markSyncReady();
```


## 创建 SurfaceSyncGroup


创建 SurfaceSyncGroup 对象，主要是部分变量的初始化。

```java
// 1. 外部调用处
new SurfaceSyncGroup(TAG)

// 2. 构造函数
public SurfaceSyncGroup(@NonNull String name) {

	this(name, transaction -> {
		if (transaction != null) {
			// ......
			transaction.apply();
		}
	});
    
}

// .......
public SurfaceSyncGroup(String name, Consumer<Transaction> transactionReadyConsumer) {
        // sCounter is a way to give the SurfaceSyncGroup a unique name even if the name passed in
        // is not.
        // Avoid letting the count get too big so just reset to 0. It's unlikely that we'll have
        // more than MAX_COUNT active syncs that have overlapping names
        // 默认counter的最高限制为 100 次
        if (sCounter.get() >= MAX_COUNT) {
            sCounter.set(0);
        }

        mName = name + "#" + sCounter.getAndIncrement();
        mTrackName = "SurfaceSyncGroup " + name;

		// 该 Callback 会在SurfaceSyncGroup.checkIfSyncIsComplete中被回调。
        mTransactionReadyConsumer = (transaction) -> {
            // ......

			// 调用transactionReadyConsumer
            transactionReadyConsumer.accept(transaction);
            synchronized (mLock) {
                // If there's a registered listener with WMS, that means we aren't actually complete
                // until WMS notifies us that the parent has completed.
                // 逐一调用 SyncCompleteCallback
                // mSyncCompleteCallbacks通过 SurfaceSyncGroup.addSyncCompleteCallback添加。
                if (mSurfaceSyncGroupCompletedListener == null) {
                    invokeSyncCompleteCallbacks();
                }
            }
        };

	    // ......
    }
```

## 添加 SurfaceView

SurfaceSyncGroup 的 add 方法中会传入一个frameCallbackConsumer。其中 frameCallbackConsumer 会在入参中传入SurfaceViewFrameCallback，而外部可以在渲染开始前调用SurfaceViewFrameCallback.onFrameStarted()这样画面将跟随 SurfaceSyncGroup一起渲染。
```java
public boolean add(SurfaceView surfaceView,
            Consumer<SurfaceViewFrameCallback> frameCallbackConsumer) {
        SurfaceSyncGroup surfaceSyncGroup = new SurfaceSyncGroup(surfaceView.getName());
        if (add(surfaceSyncGroup.mISurfaceSyncGroup, false /* parentSyncGroupMerge */,
                null /* runnable */)) {
            frameCallbackConsumer.accept(() -> surfaceView.syncNextFrame(transaction -> {
                surfaceSyncGroup.addTransaction(transaction);
                surfaceSyncGroup.markSyncReady();
            }));
            return true;
        }
        return false;
    }
```

- 一层调用：将 SurfaceView 作为参数添加
其中frameCallbackConsumer将会回调一个 SurfaceFrameCallback 接口参数。外部可以调用SurfaceViewFrameCallback.onFrameStarted进行回调。
```java
// 1. 调用 add 方法
mSyncGroup.add(mSurfaceView, frameCallback ->
                    mRenderingThread.renderFrame(frameCallback, width, height));

public boolean add(SurfaceView surfaceView,
            Consumer<SurfaceViewFrameCallback> frameCallbackConsumer) {
	    // 1. 创建 SurfaceSyncGroup 对象
        SurfaceSyncGroup surfaceSyncGroup = new SurfaceSyncGroup(surfaceView.getName());
        // 2. 将SyncGroup.mISurfaceSyncGroup添加到SurfaceSyncGroup，返回是否创建成功的状态。
        if (add(surfaceSyncGroup.mISurfaceSyncGroup, false /* parentSyncGroupMerge */,
                null /* runnable */)) {
            // 如果 add 正常则回调 frameCallback（SurfaceView开始绘制 Callback）
            // 调用syncNextFrame抓取 BLASTBufferQueue 一帧并将这一帧的 Buffer 转为Transaction。
            frameCallbackConsumer.accept(() -> surfaceView.syncNextFrame(transaction -> {
	            // 添加 Transaction 并调用 markSyncReady
                surfaceSyncGroup.addTransaction(transaction);
                surfaceSyncGroup.markSyncReady();
            }));
            return true;
        }
        return false;
    }
```


- 二层调用：将 ISurfaceSyncGroup 作为参数添加
```java
// SufaceSyncGroup.java
public boolean add(ISurfaceSyncGroup surfaceSyncGroup, boolean parentSyncGroupMerge,  
        @Nullable Runnable runnable) { 
 
    // ......
    // 1. 如果已经同步完成了直接返回添加失败。
    synchronized (mLock) {  
        if (mSyncReady) {  
            // ...... 
            return false;  
        }  
    }  
    
	// 2. 回调 Runnable（Callback为函数入参）
    if (runnable != null) {  
        runnable.run();  
    }  
    
	// 3. 将 ISurfaceSyncGroup 添加到 SurfaceSyncGroup 内。
	// 3.1 如果是 LocalBinder 则调用 addLocalSync 将surfaceSyncGroup添加到 parent中。
    if (isLocalBinder(surfaceSyncGroup.asBinder())) {  
        boolean didAddLocalSync = addLocalSync(surfaceSyncGroup, parentSyncGroupMerge);  
        // ......
        return didAddLocalSync;  
    }  
	// 3.2 通过 IPC 将 ISurfaceSyncGroup 添加到  WMS 中的 SurfaceSyncGroup 中
    synchronized (mLock) {  
	    // 如果之前没有向 WindowManager 中添加 Sync，则添加WindowManager
        if (!mHasWMSync) {  
             mSurfaceSyncGroupCompletedListener = new ISurfaceSyncGroupCompletedListener.Stub() {  
                @Override  
                public void onSurfaceSyncGroupComplete() {  
                    synchronized (mLock) {  
                        invokeSyncCompleteCallbacks();  
                    }  
                }  
            };  
            if (!addSyncToWm(mToken, false /* parentSyncGroupMerge */,  
                    mSurfaceSyncGroupCompletedListener)) {  
                mSurfaceSyncGroupCompletedListener = null;  
                // ......
                return false;  
            }  
            mHasWMSync = true;  
        }  
    }  

	// 4. ISurfaceSyncGroup 添加完成，进行 Callback 回调。
    try {  
        surfaceSyncGroup.onAddedToSyncGroup(mToken, parentSyncGroupMerge);  
    } catch (RemoteException e) {  
        // ...... 
        return false;  
    }  
  
    // ......  
    return true;  
}
```



### LocalBinder

将childSyncToken添加到 SurfaceSyncGroup 中

```java
// SurfaceSyncGroup.java
private boolean addLocalSync(ISurfaceSyncGroup childSyncToken, boolean parentSyncGroupMerge) {
		// ......
		// 将 ISurfaceSyncGroup 转化为 SurfaceSyncGroup 
        SurfaceSyncGroup childSurfaceSyncGroup = getSurfaceSyncGroup(childSyncToken);
        if (childSurfaceSyncGroup == null) {
            // ......
            return false;
        }

        // ......
        // 创建 Callback
        ITransactionReadyCallback callback =
                createTransactionReadyCallback(parentSyncGroupMerge);

        if (callback == null) {
            return false;
        }
		// 将 SurfaceSyncGroup 的 Callback 进行 reset（后续需要调用使用这个 Callback，用于调用 parent）
        childSurfaceSyncGroup.setTransactionCallbackFromParent(mISurfaceSyncGroup, callback);
        // ......
        return true;
    }

// 创建 Callback
public ITransactionReadyCallback createTransactionReadyCallback(boolean parentSyncGroupMerge) {
        // ......
        // 创建ITransactionReadyCallback
        ITransactionReadyCallback transactionReadyCallback =
                new ITransactionReadyCallback.Stub() {
                    @Override
                    public void onTransactionReady(Transaction t) {
                        synchronized (mLock) {
                            if (t != null) {
                                t.sanitize(Binder.getCallingPid(), Binder.getCallingUid());
                                // When an older parent sync group is added due to a child syncGroup
                                // getting added to multiple groups, we need to maintain merge order
                                // so the older parentSyncGroup transactions are overwritten by
                                // anything in the newer parentSyncGroup.
                                if (parentSyncGroupMerge) {
                                    t.merge(mTransaction);
                                }
                                mTransaction.merge(t);
                            }
                            mPendingSyncs.remove(this);
                            // ......
                            checkIfSyncIsComplete();
                        }
                    }
                };

		// 将SyncCallbacks 添加到 mPendingSyncs
        synchronized (mLock) {
            if (mSyncReady) {
                // ......
                return null;
            }
            mPendingSyncs.add(transactionReadyCallback);
            // ......
        }

        // 同步超时机制，超时默认为 1s，超过则取消同步（防止）
        addTimeout();

        return transactionReadyCallback;
    }

// 在添加ISurfaceSyncGroup的同时，将SurfaceSyncGroup 进行字段设置。
private void setTransactionCallbackFromParent(ISurfaceSyncGroup parentSyncGroup,
            ITransactionReadyCallback transactionReadyCallback) {
            
        // ......
        // 同步超时机制
        addTimeout();

        boolean finished = false;
        Runnable addedToSyncListener = null;
        synchronized (mLock) {
	        // 同步状态
            if (mFinished) {
                finished = true;
            } else {
                // ......
                // 如果之前关联过 parent，并且 parentSync 跟之前关联的不一样
                if (mParentSyncGroup != null && mParentSyncGroup != parentSyncGroup) {
                    // ......
                    try {
                        parentSyncGroup.addToSync(mParentSyncGroup,
                                true /* parentSyncGroupMerge */);
                    } catch (RemoteException e) {
                    }
                }
                // .......
                Consumer<Transaction> lastCallback = mTransactionReadyConsumer;
                mParentSyncGroup = parentSyncGroup;
                // Callback 以往的mTransactionReadyConsumer、并执行transactionReadyCallback 
                mTransactionReadyConsumer = (transaction) -> {
                    // ......
                    lastCallback.accept(null);

                    try {
                        transactionReadyCallback.onTransactionReady(transaction);
                    } catch (RemoteException e) {
                        transaction.apply();
                    }
                    // ......
                };
                addedToSyncListener = mAddedToSyncListener;
            }
        }

        // ......
        if (finished) {
	        // 执行完毕执行transactionReadyCallback
            try {
                transactionReadyCallback.onTransactionReady(null);
            } catch (RemoteException e) {
            }
        } else if (addedToSyncListener != null) {
	        // 执行addedToSyncListener Callback
            addedToSyncListener.run();
        }
        // ......
    }
```


### Remote Binder

用于将 SurfaceSyncGroup 添加到 WMS 中，需要借助 IPC 进行调用
```java
synchronized (mLock) {  
	    // 如果之前没有向 WindowManager 中添加 Sync，则添加WindowManager
        if (!mHasWMSync) {  
             mSurfaceSyncGroupCompletedListener = new ISurfaceSyncGroupCompletedListener.Stub() {  
                @Override  
                public void onSurfaceSyncGroupComplete() {  
                    synchronized (mLock) {  
                        invokeSyncCompleteCallbacks();  
                    }  
                }  
            };  
            if (!addSyncToWm(mToken, false /* parentSyncGroupMerge */,  
                    mSurfaceSyncGroupCompletedListener)) {  
                mSurfaceSyncGroupCompletedListener = null;  
                // ......
                return false;  
            }  
            mHasWMSync = true;  
        }  
    } 

private boolean addSyncToWm(IBinder token, boolean parentSyncGroupMerge,
            @Nullable ISurfaceSyncGroupCompletedListener surfaceSyncGroupCompletedListener) {
        try {
            // ......
            AddToSurfaceSyncGroupResult addToSyncGroupResult = new AddToSurfaceSyncGroupResult();
            if (!WindowManagerGlobal.getWindowManagerService().addToSurfaceSyncGroup(token,
                    parentSyncGroupMerge, surfaceSyncGroupCompletedListener,
                    addToSyncGroupResult)) {
                // ......
                return false;
            }

            setTransactionCallbackFromParent(addToSyncGroupResult.mParentSyncGroup,
                    addToSyncGroupResult.mTransactionReadyCallback);
        } catch (RemoteException e) {
            // ......
            return false;
        }
        // ......
        return true;
    }
```


## 添加AttachedSurfaceControl

该过程会将 ViewRootImpl 的值添加到 SurfaceSyncGroup 的节点中。
```java
// 此处的 AttachedSurfaceControl 为 ViewRootImpl
@UiThread
public boolean add(@Nullable AttachedSurfaceControl attachedSurfaceControl,
		@Nullable Runnable runnable) {
	// 1. 边界 case 过滤
	if (attachedSurfaceControl == null) {
		return false;
	}
	// 2. 确保当前窗口有一个活跃的同步组，并利用“暂停重绘”机制（Paused For Sync）来强制等待所有关联的 Surface 完成绘制。
	SurfaceSyncGroup surfaceSyncGroup = attachedSurfaceControl.getOrCreateSurfaceSyncGroup();
	if (surfaceSyncGroup == null) {
		return false;
	}
	// 3. 将 surfaceSyncGroup 添加到SurfaceSyncGroup 中
	return add(surfaceSyncGroup, runnable);
}
```


### getOrCreateSurfaceSyncGroup

每调用一次getOrCreateSurfaceSyncGroup就会累加mNumPausedForSync，同时每当一个 Sync 添加完成则进行递减，知道递减为 0 则触发 Traversal 进行执行，如果超时(超时时间为1s)也恢复 Traversal 执行。
```java
public SurfaceSyncGroup getOrCreateSurfaceSyncGroup() {
        boolean newSyncGroup = false;
        
        // 创建mActiveSurfaceSyncGroup对象
        if (mActiveSurfaceSyncGroup == null) {
            mActiveSurfaceSyncGroup = new SurfaceSyncGroup(mTag);
            // 设置 add 监听
            mActiveSurfaceSyncGroup.setAddedToSyncListener(() -> {
                Runnable runnable = () -> {
                    // 只有>0 才进行递减，防止timeout reset 的时候mNumPausedForSync变成负数。
                    if (mNumPausedForSync > 0) {
                        mNumPausedForSync--;
                    }
                    // 如果所有的 Sync 已经添加完成，则进行主线程渲染。
                    if (mNumPausedForSync == 0) {
                        mHandler.removeMessages(MSG_PAUSED_FOR_SYNC_TIMEOUT);
                        if (!mIsInTraversal) {
                            scheduleTraversals();
                        }
                    }
                };

				// 切换到主线程执行。
                if (Thread.currentThread() == mThread) {
                    runnable.run();
                } else {
                    mHandler.post(runnable);
                }
            });
            newSyncGroup = true;
        }
		// ......

		// 累加 Sync 数
        mNumPausedForSync++;

		// 超时解除同步
        mHandler.removeMessages(MSG_PAUSED_FOR_SYNC_TIMEOUT);
        mHandler.sendEmptyMessageDelayed(MSG_PAUSED_FOR_SYNC_TIMEOUT,
                1000 * Build.HW_TIMEOUT_MULTIPLIER);
        return mActiveSurfaceSyncGroup;
    };
```


### add

向当前的SurfaceSyncGroup中添加一个子 SurfaceSyncGroup
```java
public boolean add(@NonNull SurfaceSyncGroup surfaceSyncGroup,
            @Nullable Runnable runnable) {
        // 进一步调用 add 方法用于将ISurfaceSyncGroup添加到 SurfaceSyncGroup
        return add(surfaceSyncGroup.mISurfaceSyncGroup, false /* parentSyncGroupMerge */,
                runnable);
}


public boolean add(ISurfaceSyncGroup surfaceSyncGroup, boolean parentSyncGroupMerge,
            @Nullable Runnable runnable);
```



### markSyncReady

注册 Callback
```java
registerCallbacksForSync:12083, ViewRootImpl (android.view)
draw:5302, ViewRootImpl (android.view)
performDraw:4975, ViewRootImpl (android.view)
performTraversals:4093, ViewRootImpl (android.view)
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

为 Sync 注册registerRtFrameCallback监听。与此同时会通过 BLASTBufferQueue 注册syncNextTransaction 的 Callback hook 首帧转为 Transaction 通过 Transaction.apply 传给 SurfaceFlinger。
```java
// ViewRootImpl.java
private void registerCallbacksForSync(boolean syncBuffer,
            final SurfaceSyncGroup surfaceSyncGroup) {
		// ......

		// merge Transaction
        final Transaction t;
        if (mHasPendingTransactions) {
            t = new Transaction();
            t.merge(mPendingTransaction);
        } else {
            t = null;
        }

		
		// 向 ThreadedRenderer 中注册 Rt FrameCallback 方法
        mAttachInfo.mThreadedRenderer.registerRtFrameCallback(new FrameDrawingCallback() {
            @Override
            public void onFrameDraw(long frame) {
            }

            @Override
            public HardwareRenderer.FrameCommitCallback onFrameDraw(int syncResult, long frame) {
                // ......
                if (t != null) {
                    mergeWithNextTransaction(t, frame);
                }

                // 如果 syncResults 是SYNC_LOST_SURFACE_REWARD_IF_FOUND、SYNC_CONTEXT_IS_STOPPED
                // 则表明没有任何需要渲染的内容，不需要设置任何的 commit Callback
                // 需要只要调用 pendingDrawFinished
                if ((syncResult
                        & (SYNC_LOST_SURFACE_REWARD_IF_FOUND | SYNC_CONTEXT_IS_STOPPED)) != 0) {
                    surfaceSyncGroup.addTransaction(
                            mBlastBufferQueue.gatherPendingTransactions(frame));
                    surfaceSyncGroup.markSyncReady();
                    return null;
                }
                // ......

                if (syncBuffer) {
	                // 调用 syncNextTransaction 同步下一帧 Transaction。
                    boolean result = mBlastBufferQueue.syncNextTransaction(transaction -> {
                        Runnable timeoutRunnable = () -> Log.e(mTag,
                                "Failed to submit the sync transaction after 4s. Likely to ANR "
                                        + "soon");
                        mHandler.postDelayed(timeoutRunnable, 4000L * Build.HW_TIMEOUT_MULTIPLIER);
                        transaction.addTransactionCommittedListener(mSimpleExecutor,
                                () -> mHandler.removeCallbacks(timeoutRunnable));
                        surfaceSyncGroup.addTransaction(transaction);
                        surfaceSyncGroup.markSyncReady();
                    });
                    if (!result) {
                        // syncNextTransaction can only return false if something is already trying
                        // to sync the same frame in the same BBQ. That shouldn't be possible, but
                        // if it did happen, invoke markSyncReady so the active SSG doesn't get
                        // stuck.
                        // ......
                        surfaceSyncGroup.markSyncReady();
                    }
                }

                return didProduceBuffer -> {
                    // ......

                    // 如果 Frame 没有绘制，则 clear 下一帧 Transaction以便不要影响下一次绘制。
                    if (!didProduceBuffer) {
                        mBlastBufferQueue.clearSyncTransaction();

                        surfaceSyncGroup.addTransaction(
                                mBlastBufferQueue.gatherPendingTransactions(frame));
                        surfaceSyncGroup.markSyncReady();
                        return;
                    }

                    // 如果我们没有 request 同步 Buffer，我们就没有必要调用syncNextTransaction
                    // 只需要调用markSyncReady即可
                    if (!syncBuffer) {
                        surfaceSyncGroup.markSyncReady();
                    }
                };
            }
        });
    }

/**
 * 用于注册一个 Callback 用于在下一帧RenderThread中进行回调。
 **/
void registerRtFrameCallback(@NonNull FrameDrawingCallback callback) {
        if (mNextRtFrameCallbacks == null) {
            mNextRtFrameCallbacks = new ArrayList<>();
        }
        mNextRtFrameCallbacks.add(callback);
    }
```



## SurfaceSyncGroup.markSyncReady


将 SurfaceSyncGroup 标记为完成。
```java
public void markSyncReady() {
        // ......
        synchronized (mLock) {
	        // 如果sync添加到了 wms 中，则调用 WMS 的 markReady 方法 
            if (mHasWMSync) {
                try {
                    WindowManagerGlobal.getWindowManagerService().markSurfaceSyncGroupReady(mToken);
                } catch (RemoteException e) {
                }
            }
            mSyncReady = true;
            // 标记完成状态 & 执行Transaction
            checkIfSyncIsComplete();
        }
    }
```


 标记完成状态，执行 Transaction。
```java
private void checkIfSyncIsComplete() {
        if (mFinished) {
            // ......
            mTransaction.apply();
            return;
        }

        // ......

        if (!mSyncReady || !mPendingSyncs.isEmpty()) {
            // ......
            return;
        }

        // ......
        mTransactionReadyConsumer.accept(mTransaction);
        mFinished = true;
        if (mTimeoutAdded) {
            mHandler.removeCallbacksAndMessages(this);
        }
    }
```

另外当 child SurfaceSyncGroup 调用 markReady 后，回调到 parent SurfaceSyncGroup checkIfSyncIsComplete. 其中 child 到 parent 的回调主要是通过 mTransactionReadyConsumer 连接.

# ViewRootImpl Transaction 流程

该流程主要是分析 VRI 中SyncGroup执行逻辑。
```java
private void performTraversals() {

	// View measure操作
	performMeasure(...);
	// View layout操作
	performLayout(...);
	
	// 判断是否需要拦截绘制逻辑
	boolean cancelDueToPreDrawListener = mAttachInfo.mTreeObserver.dispatchOnPreDraw();
	boolean cancelAndRedraw = cancelDueToPreDrawListener
			 || (cancelDraw && mDrewOnceForSync);
	
	// 绘制前：开启 Sync
	if (!cancelAndRedraw) {
		// A sync was already requested before the WMS requested sync. This means we need to
		// sync the buffer, regardless if WMS wants to sync the buffer.
		if (mActiveSurfaceSyncGroup != null) {
			mSyncBuffer = true;
		}

		createSyncIfNeeded();
		notifyDrawStarted(isInWMSRequestedSync());
		mDrewOnceForSync = true;

		// If the active SSG is also requesting to sync a buffer, the following needs to happen
		// 1. Ensure we keep track of the number of active syncs to know when to disable RT
		//    RT animations that conflict with syncing a buffer.
		// 2. Add a safeguard SSG to prevent multiple SSG that sync buffers from being submitted
		//    out of order.
		if (mActiveSurfaceSyncGroup != null && mSyncBuffer) {
			updateSyncInProgressCount(mActiveSurfaceSyncGroup);
			safeguardOverlappingSyncs(mActiveSurfaceSyncGroup);
		}
	}
	
	// View draw 操作
	performDraw(...);
	
	
	// 绘制后：关闭 Sync
	if (!cancelAndRedraw) {
		mReportNextDraw = false;
		mLastReportNextDrawReason = null;
		mActiveSurfaceSyncGroup = null;
		mHasPendingTransactions = false;
		mSyncBuffer = false;
		if (isInWMSRequestedSync()) {
			mWmsRequestSyncGroup.markSyncReady();
			mWmsRequestSyncGroup = null;
			mWmsRequestSyncGroupState = WMS_SYNC_NONE;
		}
	}

}
```



### Pre0 触发前置条件

触发的前置条件是调用reportNextDraw传入一个 reason 对象（string 类型，标记触发同步的原因）将mReportNextDraw标记为 true，否则不会触发 VRI 的Sync 同步逻辑

```java
private void reportNextDraw(String reason) {
        if (DEBUG_BLAST) {
            Log.d(mTag, "reportNextDraw " + Debug.getCallers(5));
        }
        mReportNextDraw = true;
        mLastReportNextDrawReason = reason;
    }
```


### VRI开始同步

该方法主要有如下逻辑：
1. 创建 Root SurfaceSyncGroup，并将 VRI SurfaceSyncGroup 添加到Root中
2. 回调 vriDrawStarted，将 SurfaceView 的所有SurfaceSyncGroup 添加到 VRI 的 SurfaceSyncGroup 中。
```java
if (!cancelAndRedraw) {
	if (mActiveSurfaceSyncGroup != null) {
		mSyncBuffer = true;
	}
	// 创建 SurfaceSyncGroup 
	createSyncIfNeeded();
	// 回调notifyDrawStarted.vriDrawStarted
	// 其实就是将外部如 SurfaceView 的SurfaceSyncGroup 通过viewRoot.addToSync添加到 VRI 的SyncGroup 中。
	notifyDrawStarted(isInWMSRequestedSync());
	mDrewOnceForSync = true;

	if (mActiveSurfaceSyncGroup != null && mSyncBuffer) {
		updateSyncInProgressCount(mActiveSurfaceSyncGroup);
		safeguardOverlappingSyncs(mActiveSurfaceSyncGroup);
	}
}
```

- createSyncIfNeeded
该方法做了两件事：
1. 创建顶层的 SurfaceSyncGroup 即 mWmsRequestSyncGroup属性
2. 将 VRI 做为 child 添加到顶层的 SurfaceSyncGroup 中。

```java
// 用于创建
private void createSyncIfNeeded() {
        // WMS requested sync already started or there's nothing needing to sync
        // 如果已经创建了 WMS or mReportNextDraw为 false 直接返回
        // Note：此处可以发现若 pre0 前置条件没有被触发mReportNextDraw为 false，那么 CreateSync 会失败。
        if (isInWMSRequestedSync() || !mReportNextDraw) {
            return;
        }

        final int seqId = mSyncSeqId;
        mWmsRequestSyncGroupState = WMS_SYNC_PENDING;
        mWmsRequestSyncGroup = new SurfaceSyncGroup("wmsSync-" + mTag, t -> {
            mWmsRequestSyncGroupState = WMS_SYNC_MERGED;
            // See b/286355097. If the current process is not system, then invoking finishDraw on
            // any thread is fine since once it calls into system process, finishDrawing will run
            // on a different thread. However, when the current process is system, the finishDraw in
            // system server will be run on the current thread, which could result in a deadlock.
            if (mWindowSession instanceof Binder) {
                // The transaction should be copied to a local reference when posting onto a new
                // thread because up until now the SSG is holding a lock on the transaction. Once
                // the call jumps onto a new thread, the lock is no longer held and the transaction
                // send back may be modified or used again.
                Transaction transactionCopy = new Transaction();
                transactionCopy.merge(t);
                mHandler.postAtFrontOfQueue(() -> reportDrawFinished(transactionCopy, seqId));
            } else {
                reportDrawFinished(t, seqId);
            }
        });
        if (DEBUG_BLAST) {
            Log.d(mTag, "Setup new sync=" + mWmsRequestSyncGroup.getName());
        }
		
		// 添加 ViewRootImpl 中
        mWmsRequestSyncGroup.add(this, null /* runnable */);
    }
```

- notifyDrawStarted

通过mSurfaceChangedCallbacks 回调，给外部一个添加 VRI SyncGroup 的时机。
```java
// ViewRootImpl.java
private void notifyDrawStarted(boolean isWmSync) {
        for (int i = 0; i < mSurfaceChangedCallbacks.size(); i++) {
            mSurfaceChangedCallbacks.get(i).vriDrawStarted(isWmSync);
        }
    }
```

另外补充一点mSurfaceChangedCallbacks主要来看是 SurfaceView 添加的，其中添加时机主要是在 SurfaceView.onAttachedToWindow 
```java
// SurfaceView.java
protected void onAttachedToWindow() {
        super.onAttachedToWindow();

        getViewRootImpl().addSurfaceChangedCallback(this);
        mWindowStopped = false;

        mViewVisibility = getVisibility() == VISIBLE;
        updateRequestedVisibility();

        mAttachedToWindow = true;
        mParent.requestTransparentRegion(SurfaceView.this);
        if (!mGlobalListenersAdded) {
            ViewTreeObserver observer = getViewTreeObserver();
            observer.addOnScrollChangedListener(mScrollChangedListener);
            observer.addOnPreDrawListener(mDrawListener);
            mGlobalListenersAdded = true;
        }
    }
```

最后还有一点补充是目前源码中vriDrawStarted的主要实现方法也是在 SurfaceView 中, 具体逻辑主要是在 VRI 绘制以前添加一个 Sync 用于做画面的同步，防止闪黑的问题。具体实现逻辑如下：
```java
// SurfaceView.java
public void vriDrawStarted(boolean isWmSync) {
		// 获取 ViewRootImpl.java
        ViewRootImpl viewRoot = getViewRootImpl();
        // 如果正处于 WindowManager 的变化时机（relayout 时机）则将 SurfaceView 的 SyncGroup 也添加到 VRI 中
        // 以便后续能一起上屏渲染。
        
        synchronized (mSyncGroups) {
            if (isWmSync && viewRoot != null) {
                for (SurfaceSyncGroup syncGroup : mSyncGroups) {
                    viewRoot.addToSync(syncGroup);
                }
            }
            mSyncGroups.clear();
        }
    }
```


### VRI结束同步

主要用于重置变量，关闭 VRI。

```java
	if (!cancelAndRedraw) {
		// 重置变量状态
		mReportNextDraw = false;
		mLastReportNextDrawReason = null;
		mActiveSurfaceSyncGroup = null;
		mHasPendingTransactions = false;
		mSyncBuffer = false;
		// 如果创建成功了 SyncGroup
		if (isInWMSRequestedSync()) {
			// 兜底标记一次完成再置空
			mWmsRequestSyncGroup.markSyncReady();
			mWmsRequestSyncGroup = null;
			mWmsRequestSyncGroupState = WMS_SYNC_NONE;
		}
	}
```


### 标记结束

标记结束即SurfaceSyncGroup.markSyncReady
下面主要针对两处”标记结束“逻辑进行分析介绍： VRI、SurfaceView

#### VRI

结束标记是需要在绘制完成后调用的，因此核心就是需要知道什么时候绘制完成，而 VRI 的绘制完成是通过注册回调，通过 RenderThread 进行通知。其中回调的注册调用堆栈如下：
```plaintText
registerCallbacksForSync:12099, ViewRootImpl (android.view)
draw:5302, ViewRootImpl (android.view)
performDraw:4975, ViewRootImpl (android.view)
performTraversals:4093, ViewRootImpl (android.view)
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
run:67, KwaiExcludeUncaughtExceptionHandler$1 (com.yxcorp.gifshow.apm.crash)
handleCallback:959, Handler (android.os)
dispatchMessage:100, Handler (android.os)
loopOnce:232, Looper (android.os)
loop:317, Looper (android.os)
main:8592, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:580, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:878, ZygoteInit (com.android.internal.os)
```

注册的代码如下：
```java
// ViewRootImpl.java
private void registerCallbacksForSync(boolean syncBuffer,
            final SurfaceSyncGroup surfaceSyncGroup) {
		// ......

		// merge Transaction
        final Transaction t;
        if (mHasPendingTransactions) {
            t = new Transaction();
            t.merge(mPendingTransaction);
        } else {
            t = null;
        }

		
		// 向 ThreadedRenderer 中注册 Rt FrameCallback 方法
        mAttachInfo.mThreadedRenderer.registerRtFrameCallback(new FrameDrawingCallback() {
            @Override
            public void onFrameDraw(long frame) {
            }

            @Override
            public HardwareRenderer.FrameCommitCallback onFrameDraw(int syncResult, long frame) {
                // ......
                if (t != null) {
                    mergeWithNextTransaction(t, frame);
                }

                // 如果 syncResults 是SYNC_LOST_SURFACE_REWARD_IF_FOUND、SYNC_CONTEXT_IS_STOPPED
                // 则表明没有任何需要渲染的内容，不需要设置任何的 commit Callback
                // 需要只要调用 pendingDrawFinished
                if ((syncResult
                        & (SYNC_LOST_SURFACE_REWARD_IF_FOUND | SYNC_CONTEXT_IS_STOPPED)) != 0) {
                    surfaceSyncGroup.addTransaction(
                            mBlastBufferQueue.gatherPendingTransactions(frame));
                    surfaceSyncGroup.markSyncReady();
                    return null;
                }
                // ......

                if (syncBuffer) {
	                // 调用 syncNextTransaction 同步下一帧 Transaction。
                    boolean result = mBlastBufferQueue.syncNextTransaction(transaction -> {
                        Runnable timeoutRunnable = () -> Log.e(mTag,
                                "Failed to submit the sync transaction after 4s. Likely to ANR "
                                        + "soon");
                        mHandler.postDelayed(timeoutRunnable, 4000L * Build.HW_TIMEOUT_MULTIPLIER);
                        transaction.addTransactionCommittedListener(mSimpleExecutor,
                                () -> mHandler.removeCallbacks(timeoutRunnable));
                        surfaceSyncGroup.addTransaction(transaction);
                        surfaceSyncGroup.markSyncReady();
                    });
                    if (!result) {
                        // syncNextTransaction can only return false if something is already trying
                        // to sync the same frame in the same BBQ. That shouldn't be possible, but
                        // if it did happen, invoke markSyncReady so the active SSG doesn't get
                        // stuck.
                        // ......
                        surfaceSyncGroup.markSyncReady();
                    }
                }

                return didProduceBuffer -> {
                    // ......

                    // 如果 Frame 没有绘制，则 clear 下一帧 Transaction以便不要影响下一次绘制。
                    if (!didProduceBuffer) {
                        mBlastBufferQueue.clearSyncTransaction();

                        surfaceSyncGroup.addTransaction(
                                mBlastBufferQueue.gatherPendingTransactions(frame));
                        surfaceSyncGroup.markSyncReady();
                        return;
                    }

                    // 如果我们没有 request 同步 Buffer，我们就没有必要调用syncNextTransaction
                    // 只需要调用markSyncReady即可
                    if (!syncBuffer) {
                        surfaceSyncGroup.markSyncReady();
                    }
                };
            }
        });
    }

/**
 * 用于注册一个 Callback 用于在下一帧RenderThread中进行回调。
 **/
void registerRtFrameCallback(@NonNull FrameDrawingCallback callback) {
        if (mNextRtFrameCallbacks == null) {
            mNextRtFrameCallbacks = new ArrayList<>();
        }
        mNextRtFrameCallbacks.add(callback);
    }
```
整体的逻辑如下：
1. ViewRootImpl 在调用 draw绘制前通过向mThreadedRenderer设置 Callback，RenderThread 在特定的时机获取 Callback 并执行回调。
2. 在RenderThread 回调中调用mBlastBufferQueue.syncNextTransaction拦截下一次 Transaction 操作。并将 Transaction 进行合并提交到mActiveSurfaceSyncGroup中，标记完成。

#### SurfaceView


当SurfaceView对应 Surface 的部分元数据信息发生变化，并且 VRI 处于请求同步信号的过程中则向流程中新增同步信号并进行同步处理，等待绘制完成信号。
```java
protected void updateSurface() {
	if (creating || formatChanged || sizeChanged || visibleChanged  
        || alphaChanged || windowVisibleChanged || positionChanged  
        || layoutSizeChanged || hintChanged || relativeZChanged || !mAttachedToWindow  
        || surfaceLifecycleStrategyChanged || hdrHeadroomChanged) {
        
        final boolean redrawNeeded = sizeChanged || creating || hintChanged
                        || (mVisible && !mDrawFinished) || alphaChanged || relativeZChanged;
        boolean shouldSyncBuffer = redrawNeeded && viewRoot.wasRelayoutRequested()
                        && viewRoot.isInWMSRequestedSync();
        
        // 抓取一个 Transaction
        if (shouldSyncBuffer) {
			syncBufferTransactionCallback = new SyncBufferTransactionCallback();
			mBlastBufferQueue.syncNextTransaction(
					false /* acquireSingleBuffer */,
					syncBufferTransactionCallback::onTransactionReady);
		}
		
		// 依据状态回调 SurfaceSyncGroup
		if (redrawNeeded) {
			// ......
			// 获取 Callback
			if (callbacks == null) {
				callbacks = getSurfaceCallbacks();
			}
			// 执行 surfaceRedraw 回调
			if (shouldSyncBuffer) {
				// 执行 SurfaceHolder.Callback2#surfaceRedrawNeededAsync，并且在所有 Callback 执行后进行等待。
				handleSyncBufferCallback(callbacks, syncBufferTransactionCallback);
			} else {
				// 执行 SurfaceHolder.Callback2#surfaceRedrawNeededAsync，Callback 执行完成后不需要等待。
				handleSyncNoBuffer(callbacks);
			}
		}
                
                
    }

}
```


当需要同步 Buffer 时，向 VRI 添加 SurfaceSyncGroup 并回调redrawNeededAsync Callback。当 redrawNeededAsync 中的所有 Callback 执行完成 Callback 回调。
```java
private void handleSyncBufferCallback(SurfaceHolder.Callback[] callbacks,
            SyncBufferTransactionCallback syncBufferTransactionCallback) {
		// 创建 SurfaceSyncGroup 并添加到 ViewRootImpl 中
        final SurfaceSyncGroup surfaceSyncGroup = new SurfaceSyncGroup(getName());
        getViewRootImpl().addToSync(surfaceSyncGroup);
        // 调用 surfaceHolder.Callback2.redrawNeededAsync
        // 只有当所有的redrawNeededAsync Callback 回调完成才标记为完成状态
        redrawNeededAsync(callbacks, () -> {
            Transaction t = null;
            if (mBlastBufferQueue != null) {
		        // 只有当BlastBufferQueue.syncNextTransaction(..., false)才需要调用这个方法，用于停止向 Transaction 中 acquire Buffer。
                mBlastBufferQueue.stopContinuousSyncTransaction();
                // 等待mBlastBufferQueue.syncNextTransaction Callback 执行完成。
                t = syncBufferTransactionCallback.waitForTransaction();
            }

            surfaceSyncGroup.addTransaction(t);
            surfaceSyncGroup.markSyncReady();
            onDrawFinished();
        });
    }
```

# QA


## SurfaceSyncGroup 的作用是啥？

用于实现多个 Surface 之间的状态同步，以免多 Surface 渲染过程中出现先后渲染的问题。



## SurfaceSyncGroup & SurfaceView.applyTransactionOnDraw有什么区别？


1.SurfaceSyncGroup 主要是**用于做 Surface & Surface 间的同步**，他是一种**强保障机制**。只要是被添加到 SurfaceSyncGroup 中的 Surface 会强制进行同步，**只有当 Surface 均完成渲染/触发超时机制画面才会进行更新，否则画面会停留在上一帧**。
```java
// VRI Surface 同步逻辑
// ViewRootImpl.java 
private void registerCallbacksForSync(boolean syncBuffer,
            final SurfaceSyncGroup surfaceSyncGroup) {
	mAttachInfo.mThreadedRenderer.registerRtFrameCallback(new FrameDrawingCallback() {
            @Override
            public void onFrameDraw(long frame) {
            }

            @Override
            public HardwareRenderer.FrameCommitCallback onFrameDraw(int syncResult, long frame) {
                // ......
                if (syncBuffer) {
	                // 通过BLASTBufferQueue拦截 Transaction，并将 SurfaceSyncGroup 标记为结束
                    boolean result = mBlastBufferQueue.syncNextTransaction(transaction -> {
                        Runnable timeoutRunnable = () -> Log.e(mTag,
                                "Failed to submit the sync transaction after 4s. Likely to ANR "
                                        + "soon");
                        mHandler.postDelayed(timeoutRunnable, 4000L * Build.HW_TIMEOUT_MULTIPLIER);
                        transaction.addTransactionCommittedListener(mSimpleExecutor,
                                () -> mHandler.removeCallbacks(timeoutRunnable));
                        surfaceSyncGroup.addTransaction(transaction);
                        surfaceSyncGroup.markSyncReady();
                    });
                    if (!result) {
                        // .......
                        surfaceSyncGroup.markSyncReady();
                    }
                }

                return didProduceBuffer -> {
                    // ......
                };
            }
        });            
}

// SurfaceView Surface 同步逻辑
// SurfaceView.java
private void handleSyncBufferCallback(SurfaceHolder.Callback[] callbacks,
            SyncBufferTransactionCallback syncBufferTransactionCallback) {

        final SurfaceSyncGroup surfaceSyncGroup = new SurfaceSyncGroup(getName());
        getViewRootImpl().addToSync(surfaceSyncGroup);
        // 执行SurfaceHolder.Callback2#surfaceRedrawNeeded Callback
        // 并调用 SurfaceSyncGroup 标记结束
        redrawNeededAsync(callbacks, () -> {
            Transaction t = null;
            if (mBlastBufferQueue != null) {
                mBlastBufferQueue.stopContinuousSyncTransaction();
                t = syncBufferTransactionCallback.waitForTransaction();
            }

            surfaceSyncGroup.addTransaction(t);
            surfaceSyncGroup.markSyncReady();
            onDrawFinished();
        });
    }
```
2.applyTransactionOnDraw主要是用于 SurfaceView 的更新同步，他是一种**弱保障机制**。SurfaceView 会通过该方法**合并到 VRI 的下一帧**，不存在阻塞等待的问题，**主要用于修复 SurfaceView 渲染快于 VRI 导致的残影/黑屏/闪烁问题**。。
```java
// SurfaceView.java
public boolean applyTransactionOnDraw(@NonNull SurfaceControl.Transaction t) {
        if (mRemoved || !isHardwareEnabled()) {
            logAndTrace("applyTransactionOnDraw applyImmediately");
            t.apply();
        } else {
            Trace.instant(Trace.TRACE_TAG_VIEW, "applyTransactionOnDraw-" + mTag);
            // Copy and clear the passed in transaction for thread safety. The new transaction is
            // accessed on the render thread.
            mPendingTransaction.merge(t);
            mHasPendingTransactions = true;
        }
        return true;
    }
```




# refs

https://juejin.cn/post/7434179604774322216#heading-0
