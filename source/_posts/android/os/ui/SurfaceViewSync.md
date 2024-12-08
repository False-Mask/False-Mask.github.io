---
title: SurfaceViewSync.md
tags:
  - android
cover: >-
  https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/surfaceview.drawio.png
date: 2024-11-30 18:12:04
---



# Android13 SurfaceView Sync机制



## 引入





![surfaceview.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/surfaceview.drawio.png)



1.当我是一名初级Android开发者的时候，有一个概念一直灌输给我——Android是单线程渲染，有且只有主线程能渲染UI.

> 实际上并不是如此，除开普通的普通的View如TextView,ImageView .
>
> Android其实是支持子线程渲染的——SurfaceView.

2.SurfaceView的渲染帧不跟随View渲染，也就是说不归ViewRootImpl管。单独的Surface,单独渲染

> 特立独行是有代价的，SurfaceView的异步绘制会带来一些UI呈现的问题。
>
> 比如View渲染和SurfaceView的渲染不同步，只要SurfaceView or View渲染任意一个渲染快了就会导致UI呈现上的不一致。
>
> 
>
> 假如有一个需求场景：简单来说是一个视频播放的页面，其中包含一些普通的控件 & 用于播放视频的SurfacView.具体的需求如下
>
> ***需要播放器和页面中的其他控件一起渲染***。因为SurfaceView**渲染早了会导致只有一个光秃秃的视频在播**，**渲染晚了SurfaceView会有黑屏**。这都是不好的用户体验。

3.是否有一种同步机制能让View和SurfaceView的帧进行同步呢？

> 答案是有的，也就是Surface的Sync机制。







## 是什么？



***根据前面的内容我们不难了解。SurfaceSync机制是用于同步ViewRootImpl & SurfaceView的帧渲染的。***

***他是用于解决SurfaceView异步渲染以后导致的渲染不同步的问题。***

但是光有概念，不是很贴切？还是很陌生？

这不AOSP内有一个demo可以去验证它的一些作用。



aosp分支：android-13.0.0_r78

代码地址：[frameworks/base/tests/SurfaceViewSyncTest/SurfaceViewSyncActivity.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/tests/SurfaceViewSyncTest/src/com/android/test/SurfaceViewSyncActivity.java)

```java
/**
 * Test app that allows the user to resize the SurfaceView and have the new buffer sync with the
 * main window. This tests that {@link SurfaceSyncer} is working correctly.
 */
public class SurfaceViewSyncActivity extends Activity implements SurfaceHolder.Callback {
    private static final String TAG = "SurfaceViewSyncActivity";

    private SurfaceView mSurfaceView;
    private boolean mLastExpanded = true;

    private RenderingThread mRenderingThread;

    private final SurfaceSyncer mSurfaceSyncer = new SurfaceSyncer();

    private Button mExpandButton;
    private Switch mEnableSyncSwitch;

    private int mLastSyncId = -1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_surfaceview_sync);
        mSurfaceView = findViewById(R.id.surface_view);
        mSurfaceView.getHolder().addCallback(this);

        WindowManager windowManager = getWindowManager();
        WindowMetrics metrics = windowManager.getCurrentWindowMetrics();
        Rect bounds = metrics.getBounds();

        LinearLayout container = findViewById(R.id.container);
        mExpandButton = findViewById(R.id.expand_sv);
        mEnableSyncSwitch = findViewById(R.id.enable_sync_switch);
        mExpandButton.setOnClickListener(view -> updateSurfaceViewSize(bounds, container));

        mRenderingThread = new RenderingThread(mSurfaceView.getHolder());
    }

    private void updateSurfaceViewSize(Rect bounds, View container) {
        if (mLastSyncId >= 0) {
            return;
        }

        final float height;
        if (mLastExpanded) {
            height = bounds.height() / 2f;
            mExpandButton.setText("EXPAND SV");
        } else {
            height = bounds.height() / 1.5f;
            mExpandButton.setText("COLLAPSE SV");
        }
        mLastExpanded = !mLastExpanded;

        if (mEnableSyncSwitch.isChecked()) {
            mLastSyncId = mSurfaceSyncer.setupSync(() -> { });
            mSurfaceSyncer.addToSync(mLastSyncId, container);
        }

        ViewGroup.LayoutParams svParams = mSurfaceView.getLayoutParams();
        svParams.height = (int) height;
        mSurfaceView.setLayoutParams(svParams);
    }

    @Override
    public void surfaceCreated(@NonNull SurfaceHolder holder) {
        final Canvas canvas = holder.lockCanvas();
        canvas.drawARGB(255, 255, 0, 0);
        holder.unlockCanvasAndPost(canvas);
        mRenderingThread.startRendering();
        mRenderingThread.renderFrame(null, mSurfaceView.getWidth(), mSurfaceView.getHeight());
    }

    @Override
    public void surfaceChanged(@NonNull SurfaceHolder holder, int format, int width, int height) {
        if (mEnableSyncSwitch.isChecked()) {
            if (mLastSyncId < 0) {
                mRenderingThread.renderFrame(null, width, height);
                return;
            }
            mSurfaceSyncer.addToSync(mLastSyncId, mSurfaceView, frameCallback ->
                    mRenderingThread.renderFrame(frameCallback, width, height));
            mSurfaceSyncer.markSyncReady(mLastSyncId);
            mLastSyncId = -1;
        } else {
            mRenderingThread.renderFrame(null, width, height);
        }
    }

    @Override
    public void surfaceDestroyed(@NonNull SurfaceHolder holder) {
        mRenderingThread.stopRendering();
    }

    private static class RenderingThread extends HandlerThread {
        private final SurfaceHolder mSurfaceHolder;
        private Handler mHandler;
        private SurfaceSyncer.SurfaceViewFrameCallback mFrameCallback;
        private final Point mSurfaceSize = new Point();

        int mColorValue = 0;
        int mColorDelta = 10;
        private final Paint mPaint = new Paint();

        RenderingThread(SurfaceHolder holder) {
            super("RenderingThread");
            mSurfaceHolder = holder;
            mPaint.setColor(Color.BLACK);
            mPaint.setTextSize(100);
        }

        public void renderFrame(SurfaceSyncer.SurfaceViewFrameCallback frameCallback, int width,
                int height) {
            if (mHandler != null) {
                mHandler.post(() -> {
                    mFrameCallback = frameCallback;
                    mSurfaceSize.set(width, height);
                    mRunnable.run();
                });
            }
        }

        private final Runnable mRunnable = new Runnable() {
            @Override
            public void run() {
                if (mFrameCallback != null) {
                    mFrameCallback.onFrameStarted();
                }

                try {
                    // Long delay from start to finish to mimic slow draw
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                }

                mColorValue += mColorDelta;
                if (mColorValue > 245 || mColorValue < 10) {
                    mColorDelta *= -1;
                }

                Canvas c = mSurfaceHolder.lockCanvas();
                c.drawRGB(255, 0, 0);
                c.drawText("RENDERED CONTENT", 0, mSurfaceSize.y / 2, mPaint);
                mSurfaceHolder.unlockCanvasAndPost(c);
                mFrameCallback = null;
            }
        };

        public void startRendering() {
            start();
            mHandler = new Handler(getLooper());
        }

        public void stopRendering() {
            if (mHandler != null) {
                mHandler.removeCallbacks(mRunnable);
            }
        }
    }
}
```



[APK安装路径](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/SurfaceViewSyncTest.apk)

> note：
>
> 这个这个只针对于Android13的设备有效，其他设备会由于系统类找不到直接崩溃。
>
> 这里有个名为SurfaceSyncer的系统核心类。在android13以前是没有这个类的。
>
> android13以后被重命名为SurfaceSyncGroup。
>
> 因此该APK只在android13的设备上生效。



演示使用视频。

<iframe src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/sync_demo.mp4" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen
scrolling="no"
border="0"
frameborder="no"
framespacing="0"
height="500px"
></iframe>









## 怎么实现？



> 可以中前面的代码看出用到了一个很关键的类SurfaceSyncer

![surfacesyncer.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/surfacesyncer.drawio.png)





## 流程解析



1.setupSync



> setupSync在代码中有两个实现
>
> 虽然最后都会调用到setupSync(@NonNull Consumer<Transaction> syncRequestComplete)         ：）



> 如下代码容易理解，就是创建了一个map entry put到类mSyncSets（虽然叫set但是是一个map）中。
>
> - 其中这个map的key是一个自增的int值
>
> - value是一个set（set里面的每一个元素都是通过addToSync传入的）其中包含了syncRequestComplete回掉
> - 于此同时map中的entry会在每次sync结束之后remove掉。

```java
// SurfaceSyncer.java


@GuardedBy("mSyncSetLock")
private final SparseArray<SyncSet> mSyncSets = new SparseArray<>();

/**
 * Starts a sync and will automatically apply the final, merged transaction.
 *
 * @param onComplete The runnable that is invoked when the sync has completed. This will run on
 *                   the same thread that the sync was started on.
 * @return The syncId for the newly created sync.
 * @see #setupSync(Consumer)
 */
    public int setupSync(@Nullable Runnable onComplete) {
        Handler handler = new Handler(Looper.myLooper());
        return setupSync(transaction -> {
            transaction.apply();
            if (onComplete != null) {
                handler.post(onComplete);
            }
        });
    }

/**
 * Starts a sync.
 *
 * @param syncRequestComplete The complete callback that contains the syncId and transaction
 *                            with all the sync data merged.
 * @return The syncId for the newly created sync.
 * @hide
 * @see #setupSync(Runnable)
 */
    public int setupSync(@NonNull Consumer<Transaction> syncRequestComplete) {
        synchronized (mSyncSetLock) {
            final int syncId = mIdCounter++;
            if (DEBUG) {
                Log.d(TAG, "setupSync " + syncId);
            }
            SyncSet syncSet = new SyncSet(syncId, transaction -> {
                synchronized (mSyncSetLock) {
                    mSyncSets.remove(syncId);
                }
                syncRequestComplete.accept(transaction);
            });
            mSyncSets.put(syncId, syncSet);
            return syncId;
        }
    }
```





2.addToSync

> addToSync一共有三个重载方法，其中两个都是包装的api方法。实际核心的方法为 addToSync(int syncId, @NonNull SyncTarget syncTarget)



> 其中addToSync方法的调用很简单
>
> 1. getAndValidateSyncSet获取SyncSet
> 1. 添加callback到SyncSet中

```java
 /**
     * Add a SurfaceView to a sync set. This is different than {@link #addToSync(int, View)} because
     * it requires the caller to notify the start and finish drawing in order to sync.
     *
     * @param syncId The syncId to add an entry to.
     * @param surfaceView The SurfaceView to add to the sync.
     * @param frameCallbackConsumer The callback that's invoked to allow the caller to notify the
     *                              Syncer when the SurfaceView has started drawing and finished.
     *
     * @return true if the SurfaceView was successfully added to the SyncSet, false otherwise.
     */
    @UiThread
    public boolean addToSync(int syncId, SurfaceView surfaceView,
            Consumer<SurfaceViewFrameCallback> frameCallbackConsumer) {
        return addToSync(syncId, new SurfaceViewSyncTarget(surfaceView, frameCallbackConsumer));
    }

    /**
     * Add a View's rootView to a sync set.
     *
     * @param syncId The syncId to add an entry to.
     * @param view The view where the root will be add to the sync set
     *
     * @return true if the View was successfully added to the SyncSet, false otherwise.
     */
    @UiThread
    public boolean addToSync(int syncId, @NonNull View view) {
        ViewRootImpl viewRoot = view.getViewRootImpl();
        if (viewRoot == null) {
            return false;
        }
        return addToSync(syncId, viewRoot.mSyncTarget);
    }

    /**
     * Add a {@link SyncTarget} to a sync set. The sync set will wait for all
     * SyncableSurfaces to complete before notifying.
     *
     * @param syncId                 The syncId to add an entry to.
     * @param syncTarget A SyncableSurface that implements how to handle syncing
     *                               buffers.
     *
     * @return true if the SyncTarget was successfully added to the SyncSet, false otherwise.
     */
    public boolean addToSync(int syncId, @NonNull SyncTarget syncTarget) {
        SyncSet syncSet = getAndValidateSyncSet(syncId);
        if (syncSet == null) {
            return false;
        }
        if (DEBUG) {
            Log.d(TAG, "addToSync id=" + syncId);
        }
        return syncSet.addSyncableSurface(syncTarget);
    }
```



3.markSyncReady

> markSyncReady的方法调用比较简单～
>
> 1.设置标记位为ready
>
> 2.回调方法（syncTarget.onSyncComplete，mSyncRequestCompleteCallback）

```java
void markSyncReady() {
    synchronized (mLock) {
        mSyncReady = true;
        checkIfSyncIsComplete();
    }
}
```





4. merge

> merge所做的只是将两个SyncSet做合并
>
> 1. 语句syncId取出SyncSet（包含当前的syncId & 待合并的syncSet）
> 2. 调用syncSet.merge进行SyncSet合并

```java
/**
     * Merge another SyncSet into the specified syncId.
     * @param syncId The current syncId to merge into
     * @param otherSyncId The other syncId to be merged
     * @param otherSurfaceSyncer The other SurfaceSyncer where the otherSyncId is from
     */
    public void merge(int syncId, int otherSyncId, SurfaceSyncer otherSurfaceSyncer) {
        SyncSet syncSet;
        synchronized (mSyncSetLock) {
            syncSet = mSyncSets.get(syncId);
        }

        SyncSet otherSyncSet = otherSurfaceSyncer.getAndValidateSyncSet(otherSyncId);
        if (otherSyncSet == null) {
            return;
        }

        if (DEBUG) {
            Log.d(TAG,
                    "merge id=" + otherSyncId + " from=" + otherSurfaceSyncer + " into id=" + syncId
                            + " from" + this);
        }
        syncSet.merge(otherSyncSet);
    }
```



> SyncSet.merge
>
> 1. 将待合并的SyncSet添加进mergeSyncSet。
> 2. 更新待添加SyncSet的Callback

```java
/**
 * Merge a SyncSet into this SyncSet. Since SyncSets could still have pending SyncTargets,
 * we need to make sure those can still complete before the mergeTo syncSet is considered
 * complete.
 *
 * We keep track of all the merged SyncSets until they are marked as done, and then they
 * are removed from the set. This SyncSet is not considered done until all the merged
 * SyncSets are done.
 *
 * When the merged SyncSet is complete, it will invoke the original syncRequestComplete
 * callback but send an empty transaction to ensure the changes are applied early. This
 * is needed in case the original sync is relying on the callback to continue processing.
 *
 * @param otherSyncSet The other SyncSet to merge into this one.
 */
        public void merge(SyncSet otherSyncSet) {
            synchronized (mLock) {
                mMergedSyncSets.add(otherSyncSet);
            }
            otherSyncSet.updateCallback(transaction -> {
                synchronized (mLock) {
                    mMergedSyncSets.remove(otherSyncSet);
                    mTransaction.merge(transaction);
                    checkIfSyncIsComplete();
                }
            });
        }
```



> 更新callback
>
> 将oldCallback做了替换。
>
> 其实本质上就了如下操作
>
> 1.包装oldCallback通过空Transaction回调
>
> 2.实际的Transition回调给

```java
public void updateCallback(Consumer<Transaction> transactionConsumer) {
            synchronized (mLock) {
                if (mFinished) {
                    Log.e(TAG, "Attempting to merge SyncSet " + mSyncId + " when sync is"
                            + " already complete");
                    transactionConsumer.accept(new Transaction());
                }

                final Consumer<Transaction> oldCallback = mSyncRequestCompleteCallback;
                mSyncRequestCompleteCallback = transaction -> {
                    oldCallback.accept(new Transaction());
                    transactionConsumer.accept(transaction);
                };
            }
        }
```







## 总结



1.在Andorid13上Surface Sync通过一个名为SurfaceSyncer的API提供

2.SurfaceSyncer有3个比较重要的API

  a.setupSync开启一个sync

  b.addToSync添加syncTarget到map中（添加sync的元素会同步帧渲染）

  c.markSyncReady标记sync为完成状态（等到syncTarget均完成渲染统一进行合并上屏）

3.关于SurfaceSyncer的回调

  a.setupSync(syncRequestComplete,onComplete)这两个回掉会在markSyncReady回调

  b.SyncTarget其中onReadyToSync会在addToSync成功添加到SyncSet中回调，而onSyncComplete会在markSyncReady中回调

4.其他注意事项

  a.SurfaceSyncer这个API只针对于Android13,这个版本，在此版本之前没有Sync的概念，在此版本之后API名称变为了SurfaceSyncGroup

  b.SurfaceSyncer API是一个hidden api，并非public可以通过反射调用。android14的SurfaceSyncGroup则是一个public api





# 源码中的运用



> 首先可以确认framework源码中主要有两处在使用SurfaceSyncer
>
> ViewRootImpl，SurfaceView

![surfacesyncer-use.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/surfacesyncer-usage.drawio.png)







## SurfaceView



> SurfaceView主要的使用位置为handleSyncNoBuffer。
>
> 根据上述的流程图可得知是调用时机主要在画面渲染的最早时刻。
>
> 即ViewRootImpl调用PreDraw



> SurfaceSyncer API添加同步逻辑
>
> 1.setupSync 添加sync complete回调，onDrawFinished主要是用于requestTransparentRegion挖孔
>
> 2.addToSync添加SyncAdd成功的回掉，其中通过redrawNeededAsync回调SurfaceHolder Callback.surfaceRedrawNeeded



> SurfaceSyncerStarted调用逻辑。
>
> 通过将所有未处理完的syncId和ViewRootImpl的SyncSet合并

```java
public class SurfaceView extends View implements ViewRootImpl.SurfaceChangedCallback {


    private final SurfaceSyncer mSurfaceSyncer = new SurfaceSyncer();
    
    private void handleSyncNoBuffer(SurfaceHolder.Callback[] callbacks) {
        // 
        final int syncId = mSurfaceSyncer.setupSync(this::onDrawFinished);

        mSurfaceSyncer.addToSync(syncId, syncBufferCallback -> redrawNeededAsync(callbacks,
                () -> {
                    syncBufferCallback.onBufferReady(null);
                    synchronized (mSyncIds) {
                        mSyncIds.remove(syncId);
                    }
                }));

        mSurfaceSyncer.markSyncReady(syncId);
        synchronized (mSyncIds) {
            mSyncIds.add(syncId);
        }
    }
    
    
    @Override
    public void surfaceSyncStarted() {
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot != null) {
            synchronized (mSyncIds) {
                for (int syncId : mSyncIds) {
                    viewRoot.mergeSync(syncId, mSurfaceSyncer);
                }
            }
        }
    }
    

}
```





## ViewRootImpl



> ViewRootImpl使用位置主要是如下几处
>
> 1.preDraw以后确认需要渲染下一帧，通过creareSyncIfNeeded创建一次Sync
>
> 2.mereSync，给外部类如SurfaceView合并SyncSet.
>
> 3.在performTraversal函数调用结尾的位置回调markSyncReady

```JAVA
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks,
        AttachedSurfaceControl {


	private void performTraversals() {
     	// pre draw
        boolean cancelDueToPreDrawListener = mAttachInfo.mTreeObserver.dispatchOnPreDraw();
        boolean cancelAndRedraw = cancelDueToPreDrawListener
                 || (cancelDraw && mDrewOnceForSync);
        // 确认下一帧需要渲染，创建一次Sync
        if (!cancelAndRedraw) {
            createSyncIfNeeded();
            mDrewOnceForSync = true;
        }
        
        // ......
        
        // 标记Sync结束,同步完成
        if (!cancelAndRedraw) {
            mReportNextDraw = false;
            mLastReportNextDrawReason = null;
            mSyncBufferCallback = null;
            mSyncBuffer = false;
            if (isInLocalSync()) {
                mSurfaceSyncer.markSyncReady(mSyncId);
                mSyncId = UNSET_SYNC_ID;
                mLocalSyncState = LOCAL_SYNC_NONE;
            }
        }
        
        
    }
            
            
    
    private void createSyncIfNeeded() {
        // Started a sync already or there's nothing needing to sync
        if (isInLocalSync() || !mReportNextDraw) {
            return;
        }

        final int seqId = mSyncSeqId;
        mLocalSyncState = LOCAL_SYNC_PENDING;
        mSyncId = mSurfaceSyncer.setupSync(transaction -> {
            mLocalSyncState = LOCAL_SYNC_RETURNED;
            // Callback will be invoked on executor thread so post to main thread.
            mHandler.postAtFrontOfQueue(() -> {
                mLocalSyncState = LOCAL_SYNC_MERGED;
                mSurfaceChangedTransaction.merge(transaction);
                reportDrawFinished(seqId);
            });
        });
        if (DEBUG_BLAST) {
            Log.d(mTag, "Setup new sync id=" + mSyncId);
        }
        mSurfaceSyncer.addToSync(mSyncId, mSyncTarget);
        // 回调SurfaceHolde.Callback方法
        notifySurfaceSyncStarted();
    }
            
   
   // 通常留给SurfaceView回调用以合并SyncSet         
   void mergeSync(int syncId, SurfaceSyncer otherSyncer) {
        if (!isInLocalSync()) {
            return;
        }
        mSurfaceSyncer.merge(mSyncId, syncId, otherSyncer);
    }


        
}
```







## 总结



1.SurfaceSyncer在Framework的使用场景比较单一，主要是SurfaceView & ViewRootImpl中用于同步Surface的绘制内容。

2.SurfaceView中使用到的主要场景主要是在ViewRootImpl preDraw以前创建Sync,并通过merge与ViewRootImpl的SurfaceSyncer进行同步

3.ViewRootImpl的使用场景也类似，在preDraw以后创建Sync，并回调SurfaceView进行Sync的合并，在ViewRootImpl performTraversal结尾标记Sync完成。





# 最后





小小总结一下本篇文章的内容：

1.SurfaceSyncer并不是公开的API,目前暂时只是Framework的internal API

2.SurfaceSyncer的实现在Android 14被重命名为类SurfaceSyncGroup。

3.SurfaceSyncer主要是一种同步机制，用于同步SurfaceView & ViewRootImpl的绘制不同步的问题





 

最最后送给读者：

> Go give yourself a break yeah～
>
> Remember it's not a race～
>
> Everyone's got these days～
>
> Remember you'll be okey hmm～
>
> ​	——我在听《The Race》  Chiris James











