---
title: Android View渲染原理
tags:
- aosp
cover:
---



# Android View渲染原理



本文章属于专栏 Android 渲染原理分析 





# 环境



> 源码版本Android-14.0.0_r73



# pre



> 本文主要分析的是普通View,比如ImageView or Button, Text等的渲染原理，
>
> TextureView or SurfaceView等控件不在此范围内



# 起点



> 有过一定android开发经验的同学可能知道，在Android中View的渲染主要是由View去绘制实现的。
>
> 各种各样的View会对应于各个样式的UI,通过ViewTree组织在一起。共同构成了界面。
>
> 其中每个View的核心渲染逻辑主要在draw方法里面。
>
> 





# ViewRootImpl.performTraversal



> 在ViewRootImpl中的performTraversal方法中会对ViewTree进行测量、布局和绘制操作。
>
> 其中我们比较关注的绘制操作就在这里。
>
> （下方代码是做了省略的，实际代码肯定不止这么一点）

``` java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks,
        AttachedSurfaceControl {

            
          private void performTraversals() {
           
              // 1. 布局测量 
              measureHierarchy(host, lp,
                        mView.getContext().getResources(), desiredWindowWidth, desiredWindowHeight,
                        shouldOptimizeMeasure);
              
              // 2. 布局
              performLayout(lp, mWidth, mHeight);
              
              // 3. 绘制
              performDraw(mActiveSurfaceSyncGroup)
              
              
          }
            
            
}
```



# ViewRootImpl.draw



> 在performDraw中会调用draw方法，draw方法会进一步处理绘制逻辑

``` java
private boolean performDraw(@Nullable SurfaceSyncGroup surfaceSyncGroup) {
       
    // ......
    usingAsyncReport = draw(fullRedrawNeeded, surfaceSyncGroup, mSyncBuffer);
    // ......
}
```



> draw方法中主要依据是否开启硬件加速做了一个分叉
>
> 由于android api 14及以上默认开启，因此后续主要分析“硬件加速”的渲染逻辑。

``` java
private boolean draw(boolean fullRedrawNeeded, @Nullable SurfaceSyncGroup activeSyncGroup,
            boolean syncBuffer) {
 
    if (isHardwareEnabled()) {
        // 硬件加速-GPU渲染加速
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        
    } else {
        // 软件渲染-CPU渲染
        drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets);
    }
    
    
}
```



# ThreadedRenderer.draw



> 这个方法是硬件加速渲染的分叉点。（只有硬件加速会走这个逻辑）
>
> 方法主要是做绘制操作。



> 从下方代码中可以看出核心逻辑其实就是
>
> 1.updateRootDisplayList——获取displayList对象
>
> 2.syncAndDrawFrame——将renderNode同步到render thread 并 请求上屏

```java
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    attachInfo.mViewRootImpl.mViewFrameInfo.markDrawStart();

    // 1. 获取displayList
    updateRootDisplayList(view, callbacks);

    // register animating rendernodes which started animating prior to renderer
    // creation, which is typical for animators started prior to first draw
    // 注册动画用的rendernodes（过多关注）
    if (attachInfo.mPendingAnimatingRenderNodes != null) {
        final int count = attachInfo.mPendingAnimatingRenderNodes.size();
        for (int i = 0; i < count; i++) {
            registerAnimatingRenderNode(
                    attachInfo.mPendingAnimatingRenderNodes.get(i));
        }
        attachInfo.mPendingAnimatingRenderNodes.clear();
        // We don't need this anymore as subsequent calls to
        // ViewRootImpl#attachRenderNodeAnimator will go directly to us.
        attachInfo.mPendingAnimatingRenderNodes = null;
    }

    // 获取frameInfo对象
    // Note：这个对象没啥特殊的，只是一个用来记录绘制过程中每个阶段耗时的数据类
    // 数据结构上是一个数组，index对应绘制阶段，value对应该阶段的时间戳
    final FrameInfo frameInfo = attachInfo.mViewRootImpl.getUpdatedFrameInfo();

    // 2. 从函数名称不难理解sync + drawFrmae ———— 即将renderNode同步到render thread 并 请求上屏
    int syncResult = syncAndDrawFrame(frameInfo);
    if ((syncResult & SYNC_LOST_SURFACE_REWARD_IF_FOUND) != 0) {
        Log.w("HWUI", "Surface lost, forcing relayout");
        // We lost our surface. For a relayout next frame which should give us a new
        // surface from WindowManager, which hopefully will work.
        attachInfo.mViewRootImpl.mForceNextWindowRelayout = true;
        attachInfo.mViewRootImpl.requestLayout();
    }
    if ((syncResult & SYNC_REDRAW_REQUESTED) != 0) {
        attachInfo.mViewRootImpl.invalidate();
    }
}
```



## updateRootDisplayList



> 该方法主要用来更新view节点的renderNode节点
>
> 但是核心逻辑不在这个方法中，因此我们需要进一步查看updateViewTreeDisplayList的代码逻辑

``` java
// ThreadedRenderer.java

private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
        // 1. 更新viewTree的renderNode
        updateViewTreeDisplayList(view);

        // Consume and set the frame callback after we dispatch draw to the view above, but before
        // onPostDraw below which may reset the callback for the next frame.  This ensures that
        // updates to the frame callback during scroll handling will also apply in this frame.
    	// 2. 设置相关监听用的(不是重点)
        if (mNextRtFrameCallbacks != null) {
            final ArrayList<FrameDrawingCallback> frameCallbacks = mNextRtFrameCallbacks;
            mNextRtFrameCallbacks = null;
            setFrameCallback(new FrameDrawingCallback() {
                @Override
                public void onFrameDraw(long frame) {
                }

                @Override
                public FrameCommitCallback onFrameDraw(int syncResult, long frame) {
                    ArrayList<FrameCommitCallback> frameCommitCallbacks = new ArrayList<>();
                    for (int i = 0; i < frameCallbacks.size(); ++i) {
                        FrameCommitCallback frameCommitCallback = frameCallbacks.get(i)
                                .onFrameDraw(syncResult, frame);
                        if (frameCommitCallback != null) {
                            frameCommitCallbacks.add(frameCommitCallback);
                        }
                    }

                    if (frameCommitCallbacks.isEmpty()) {
                        return null;
                    }

                    return didProduceBuffer -> {
                        for (int i = 0; i < frameCommitCallbacks.size(); ++i) {
                            frameCommitCallbacks.get(i).onFrameCommit(didProduceBuffer);
                        }
                    };
                }
            });
        }

    	// 3. 根节点发生变化，重新生成rootNode
        if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
            RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
            try {
                final int saveCount = canvas.save();
                canvas.translate(mInsetLeft, mInsetTop);
                callbacks.onPreDraw(canvas);

                canvas.enableZ();
                // 生产
                canvas.drawRenderNode(view.updateDisplayListIfDirty());
                canvas.disableZ();

                callbacks.onPostDraw(canvas);
                canvas.restoreToCount(saveCount);
                mRootNodeNeedsUpdate = false;
            } finally {
                mRootNode.endRecording();
            }
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
```



> 这个方法十分的简单，由于当前类还是在ThreadRenderer因此需要初始化跟节点的一些状态信息。
>
> 再调用updateDisplayListIfDirty更新ViewTree的displayList列表。
>
> 依旧重点不在方法本身，而是view.updateDisplayListIfDirty这个方法

``` java
private void updateViewTreeDisplayList(View view) {
        view.mPrivateFlags |= View.PFLAG_DRAWN;
    	// 用于标记是否需要重新生成displayList
        view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
                == View.PFLAG_INVALIDATED;
        view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    	// 借助view自身逻辑去更新displayList
        view.updateDisplayListIfDirty();
        view.mRecreateDisplayList = false;
    }
```



### createDisplayList



1.RootRender触发重建

```java
private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
    updateViewTreeDisplayList(view);

    // ......

    // 如果rootNode需要重构, 则调用canvas.drawRenderNode添加RenderNode
    if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
        RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
        try {
            final int saveCount = canvas.save();
            canvas.translate(mInsetLeft, mInsetTop);
            callbacks.onPreDraw(canvas);

            canvas.enableZ();
            canvas.drawRenderNode(view.updateDisplayListIfDirty());
            canvas.disableZ();

            callbacks.onPostDraw(canvas);
            canvas.restoreToCount(saveCount);
            mRootNodeNeedsUpdate = false;
        } finally {
            mRootNode.endRecording();
        }
    }
    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
}
```

2. ViewGroup触发renderNode重建

``` java
// View.java

boolean draw(@NonNull Canvas canvas, ViewGroup parent, long drawingTime) {
	
    RenderNode renderNode = null;
	if (drawingWithRenderNode) {
        // Delay getting the display list until animation-driven alpha values are
        // set up and possibly passed on to the view
        renderNode = updateDisplayListIfDirty();
        if (!renderNode.hasDisplayList()) {
            // Uncommon, but possible. If a view is removed from the hierarchy during the call
            // to getDisplayList(), the display list will be marked invalid and we should not
            // try to use it again.
            renderNode = null;
            drawingWithRenderNode = false;
        }
    }
    
    // 将view property内容写入到 RenderNode中
    if (drawingWithRenderNode) {
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        // 将节点添加到RenderNode中
        ((RecordingCanvas) canvas).drawRenderNode(renderNode);
    }

}
```



创建RenderNode结构的逻辑

ViewGroupA

\- ViewGroupB

​	\- ViewA

\- ViewGroupC

``` java
// 1. 创建Canvas
public RenderNode updateDisplayListIfDirty() {
    
    renderNode.beginRecording();

    dispatchDraw();

    renderNode.endRecording();
   
}

boolean draw(@NonNull Canvas canvas, ViewGroup parent, long drawingTime) {
 	
    // 递归！！
    RenderNode renderNode = updateDisplayListIfDirty();
    // 将RenderNode插入到当前RenderNode scope的child节点中。
    ((RecordingCanvas) canvas).drawRenderNode(renderNode);

}
```



```java
public final class RecordingCanvas extends BaseRecordingCanvas {

    // ......
    
	@Override
    public void drawRenderNode(@NonNull RenderNode renderNode) {
        nDrawRenderNode(mNativeCanvasWrapper, renderNode.mNativeRenderNode);
    }
    
    private static native void nDrawRenderNode(long renderer, long renderNode);
    
    // ......
    
}
```







### updateDisplayList







### View.updateDisplayListIfDirty



> 重点来了，这个方法会**递归**遍历所有的View节点，依次更新每一个View节点所对应的RenderNode节点的DisplayList内容。
>
> 代码逻辑如下

- 没有attach ThreadRenderer -> 不更新DisplayList
- View缓存有效 & 当前RenderNode节点有DisplayList & 不需要重建displayList节点 -> 不更新DisplayList
- other -> 更新DisplayList
  - 当前renderNode节点包含displayList节点 & 不需要重新重建 -> 分发给child重建displayList（递归）
  - 需要重新重建 & 当前view的layerType为LAYER_TYPE_SOFTWARE -> 使用Bitmap缓存/不重建displayList
  - 需要重新重建 & 当前view的layerType不为LAYER_TYPE_SOFTWARE依据ViewGroup是否为PFLAG_SKIP_DRAW分发方法
    - 是PFLAG_SKIP_DRAW(父View不需要绘制) -> 调用dispatchDraw记录子View （递归）
    - 不是PFLAG_SKIP_DRAW(父View需要参与绘制) -> 调用draw方法记录父View + 子View的绘制操作。（递归）

``` java
public RenderNode updateDisplayListIfDirty() {
        final RenderNode renderNode = mRenderNode;
    	// 没有attach ThreadRenderer不更新RenderNode 
        if (!canHaveDisplayList()) {
            // can't populate RenderNode, don't try
            return renderNode;
        }
		
    	// view缓存失效 or 当前没有displayList or 需要重建displayList
        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
                || !renderNode.hasDisplayList()
                || (mRecreateDisplayList)) {
            // Don't need to recreate the display list, just need to tell our
            // children to restore/recreate theirs
            // 有displayList并且不需要重建 -> 调用dispatchGetDisplayList更新子view的displayList
            if (renderNode.hasDisplayList()
                    && !mRecreateDisplayList) {
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                dispatchGetDisplayList();

                return renderNode; // no work needed
            }

            // If we got here, we're recreating it. Mark it as such to ensure that
            // we copy in child display lists into ours in drawChild()
            // 执行到这表明displayList需要重建
            mRecreateDisplayList = true;

            int width = mRight - mLeft;
            int height = mBottom - mTop;
            int layerType = getLayerType();

            // Hacky hack: Reset any stretch effects as those are applied during the draw pass
            // instead of being "stateful" like other RenderNode properties
            // 清空状态（无需关注）
            renderNode.clearStretch();

            // 开始记录
            final RecordingCanvas canvas = renderNode.beginRecording(width, height);

            try {
                // layerType为LAYER_TYPE_SOFTWARE使用缓存
                if (layerType == LAYER_TYPE_SOFTWARE) {
                    buildDrawingCache(true);
                    Bitmap cache = getDrawingCache(true);
                    if (cache != null) {
                        canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                    }
                } else { // 其他case
                    computeScroll();

                    canvas.translate(-mScrollX, -mScrollY);
                    mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                    mPrivateFlags &= ~PFLAG_DIRTY_MASK;

                    // Fast path for layouts with no backgrounds
                    // 父view无绘制内容，调用dispachDraw绘制子view
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        dispatchDraw(canvas);
                        drawAutofilledHighlight(canvas);
                        if (mOverlay != null && !mOverlay.isEmpty()) {
                            mOverlay.getOverlayView().draw(canvas);
                        }
                        if (isShowingLayoutBounds()) {
                            debugDrawFocus(canvas);
                        }
                    } else {
                        // 父view有绘制内容，调用总调度方法绘制 父view + 子view
                        draw(canvas);
                    }
                }
            } finally {
                // 结束绘制
                renderNode.endRecording();
                setDisplayListProperties(renderNode);
            }
        } else {
            // 不需要重建displayList 直接返回
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        }
        return renderNode;
    }
```



> 经过上述代码逻辑分析，我们不难得出updateDisplayListIfDirty其实就是一个递归。不断地往下递归记录所有的View的DisplayList
>
> 但是我们漏了一个东西——系统是如何记录DisplayList的呢？我们继续分析



### record displayList



> 精简了一下updateDisplayListIfDirty的逻辑。记录displayList的核心逻辑如下
>
> 很明显可以看出，所谓的录制操作其实就是使用了特定的canvas，在draw以前通过beginRecording获取一个特殊的canvas,
>
> 然后传入到draw方法内，执行完成后再调用endRecording.

```java
public RenderNode updateDisplayListIfDirty() {
 
    // ......
    final RecordingCanvas canvas = renderNode.beginRecording(width, height);
    try {
        if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
            dispatchDraw(canvas);
            drawAutofilledHighlight(canvas);
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().draw(canvas);
            }
            if (isShowingLayoutBounds()) {
                debugDrawFocus(canvas);
            }
        } else {
            draw(canvas);
        }
    }  finally {
        renderNode.endRecording();
        setDisplayListProperties(renderNode);
    }
    
    
}
```



- RecordingCanvas实例获取

> 接着分析下beginRecording是怎么创建canvas的
>
> 从代码中很容易可以得知这里的canvas只是简单地从缓存池中获取，没什么特殊的。（类似的还有android.os.Message）

```java
public final class RenderNode {
    // ......
    
    public @NonNull RecordingCanvas beginRecording(int width, int height) {
        if (mCurrentRecordingCanvas != null) {
            throw new IllegalStateException(
                    "Recording currently in progress - missing #endRecording() call?");
        }
        mCurrentRecordingCanvas = RecordingCanvas.obtain(this, width, height);
        return mCurrentRecordingCanvas;
    }
    
}
```



> 执行逻辑还是蛮简单的，先尝试从缓存池中获取实例，如果没有则创建一个新的，如果有则reset一下状态数据
>
> 除开几个JNI方法创建的逻辑目前来看没什么特殊的地方

```java
public final class RecordingCanvas extends BaseRecordingCanvas {
 
    // ......
    
    static RecordingCanvas obtain(@NonNull RenderNode node, int width, int height) {
        if (node == null) throw new IllegalArgumentException("node cannot be null");
        // 尝试从缓存池中获取实例
        // 如果没有则创建一个新的
        // 如果有则reset一下状态数据
        RecordingCanvas canvas = sPool.acquire();
        if (canvas == null) {
            canvas = new RecordingCanvas(node, width, height);
        } else {
            nResetDisplayListCanvas(canvas.mNativeCanvasWrapper, node.mNativeRenderNode,
                    width, height);
        }
        canvas.mNode = node;
        canvas.mWidth = width;
        canvas.mHeight = height;
        return canvas;
    }
    
     private RecordingCanvas(@NonNull RenderNode node, int width, int height) {
        super(nCreateDisplayListCanvas(node.mNativeRenderNode, width, height));
        mDensity = 0; // disable bitmap density scaling
    }
    
}
```



- RecordingCanvas JNI分析

``` c++
// android_graphics_DisplayListCanvas.cpp

// ----------------------------------------------------------------------------
// JNI Glue
// ----------------------------------------------------------------------------

const char* const kClassPathName = "android/graphics/RecordingCanvas";

static JNINativeMethod gMethods[] = {

    // ------------ @FastNative ------------------

    { "nCallDrawGLFunction", "(JJLjava/lang/Runnable;)V",
            (void*) android_view_DisplayListCanvas_callDrawGLFunction },

    // ------------ @CriticalNative --------------
    { "nCreateDisplayListCanvas", "(JII)J",     (void*) android_view_DisplayListCanvas_createDisplayListCanvas },
    { "nResetDisplayListCanvas",  "(JJII)V",    (void*) android_view_DisplayListCanvas_resetDisplayListCanvas },
    { "nGetMaximumTextureWidth",  "()I",        (void*) android_view_DisplayListCanvas_getMaxTextureSize },
    { "nGetMaximumTextureHeight", "()I",        (void*) android_view_DisplayListCanvas_getMaxTextureSize },
    { "nInsertReorderBarrier",    "(JZ)V",      (void*) android_view_DisplayListCanvas_insertReorderBarrier },
    { "nFinishRecording",         "(J)J",       (void*) android_view_DisplayListCanvas_finishRecording },
    { "nDrawRenderNode",          "(JJ)V",      (void*) android_view_DisplayListCanvas_drawRenderNode },
    { "nDrawTextureLayer",        "(JJ)V",      (void*) android_view_DisplayListCanvas_drawTextureLayer },
    { "nDrawCircle",              "(JJJJJ)V",   (void*) android_view_DisplayListCanvas_drawCircleProps },
    { "nDrawRoundRect",           "(JJJJJJJJ)V",(void*) android_view_DisplayListCanvas_drawRoundRectProps },
    { "nDrawWebViewFunctor",      "(JI)V",      (void*) android_view_DisplayListCanvas_drawWebViewFunctor },
};

int register_android_view_DisplayListCanvas(JNIEnv* env) {
    jclass runnableClass = FindClassOrDie(env, "java/lang/Runnable");
    gRunnableMethodId = GetMethodIDOrDie(env, runnableClass, "run", "()V");

    return RegisterMethodsOrDie(env, kClassPathName, gMethods, NELEM(gMethods));
}
```



> nCreateDisplayListCanvas方法没啥特殊的
>
> 创建了一个SkiaRecordingCanvas reset了一下状态信息然后将SkiaRecordingCanvas的句柄转化了一下就返回了。

``` cpp
static jlong android_view_DisplayListCanvas_createDisplayListCanvas(CRITICAL_JNI_PARAMS_COMMA jlong renderNodePtr,
        jint width, jint height) {
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    return reinterpret_cast<jlong>(Canvas::create_recording_canvas(width, height, renderNode));
}

// Canvas.cpp
Canvas* Canvas::create_recording_canvas(int width, int height, uirenderer::RenderNode* renderNode) {
    return new uirenderer::skiapipeline::SkiaRecordingCanvas(renderNode, width, height);
}

// SkiaRecordingCanvas.h
explicit SkiaRecordingCanvas(uirenderer::RenderNode* renderNode, int width, int height) {
        initDisplayList(renderNode, width, height);
}

// SkiaRecordingCanvas.cpp
void SkiaRecordingCanvas::initDisplayList(uirenderer::RenderNode* renderNode, int width,
                                          int height) {
    mCurrentBarrier = nullptr;
    SkASSERT(mDisplayList.get() == nullptr);

    if (renderNode) {
        mDisplayList = renderNode->detachAvailableList();
    }
    if (!mDisplayList) {
        mDisplayList.reset(new SkiaDisplayList());
    }

    mDisplayList->attachRecorder(&mRecorder, SkIRect::MakeWH(width, height));
    SkiaCanvas::reset(&mRecorder);
}
```



> nResetDisplayListCanvas

```c++
static void android_view_DisplayListCanvas_resetDisplayListCanvas(CRITICAL_JNI_PARAMS_COMMA jlong canvasPtr,
        jlong renderNodePtr, jint width, jint height) {
    Canvas* canvas = reinterpret_cast<Canvas*>(canvasPtr);
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    canvas->resetRecording(width, height, renderNode);
}

// Skiarecordingcanvas.h 
virtual void resetRecording(int width, int height,
                                uirenderer::RenderNode* renderNode) override {
        initDisplayList(renderNode, width, height);
}
```



- DisplayList记录

> 分析完上述的RecordingCanvas创建以及初始化操作，我们总算是可以继续分析最核心的displayList生成逻辑了。
>
> 不难看出生成displayList的核心逻辑其实在draw/dispatchDraw内。
>
> 聪明的你肯定是找到draw/dispatchDraw方法内部其实主要就是调用canvas.drawXX方法用于绘制。
>
> 难不成displayList是在canvas.drawXX生成的？

``` java
public RenderNode updateDisplayListIfDirty() {
 
    // ......
    final RecordingCanvas canvas = renderNode.beginRecording(width, height);
    try {
        if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
            dispatchDraw(canvas);
            drawAutofilledHighlight(canvas);
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().draw(canvas);
            }
            if (isShowingLayoutBounds()) {
                debugDrawFocus(canvas);
            }
        } else {
            draw(canvas);
        }
    }  finally {
        renderNode.endRecording();
        setDisplayListProperties(renderNode);
    }
    
    
```



> 没错，确实是在这个过程生成的。
>
> 由于canvas的方法主要是native方法，我们就直接从JNI入手了，不去分析java实现了
>
> 可以发现所有drawXXX的方法在BaseRecordingCanvas都是注册了JNI的实现的

```c++
static const JNINativeMethod gMethods[] = {
    {"nGetNativeFinalizer", "()J", (void*) CanvasJNI::getNativeFinalizer},
    {"nFreeCaches", "()V", (void*) CanvasJNI::freeCaches},
    {"nFreeTextLayoutCaches", "()V", (void*) CanvasJNI::freeTextLayoutCaches},
    {"nSetCompatibilityVersion", "(I)V", (void*) CanvasJNI::setCompatibilityVersion},

    // ------------ @FastNative ----------------
    {"nInitRaster", "(J)J", (void*) CanvasJNI::initRaster},
    {"nSetBitmap", "(JJ)V", (void*) CanvasJNI::setBitmap},
    {"nGetClipBounds","(JLandroid/graphics/Rect;)Z", (void*) CanvasJNI::getClipBounds},

    // ------------ @CriticalNative ----------------
    {"nIsOpaque","(J)Z", (void*) CanvasJNI::isOpaque},
    {"nGetWidth","(J)I", (void*) CanvasJNI::getWidth},
    {"nGetHeight","(J)I", (void*) CanvasJNI::getHeight},
    {"nSave","(JI)I", (void*) CanvasJNI::save},
    {"nSaveLayer","(JFFFFJI)I", (void*) CanvasJNI::saveLayer},
    {"nSaveLayerAlpha","(JFFFFII)I", (void*) CanvasJNI::saveLayerAlpha},
    {"nSaveUnclippedLayer","(JIIII)I", (void*) CanvasJNI::saveUnclippedLayer},
    {"nRestoreUnclippedLayer","(JIJ)V", (void*) CanvasJNI::restoreUnclippedLayer},
    {"nGetSaveCount","(J)I", (void*) CanvasJNI::getSaveCount},
    {"nRestore","(J)Z", (void*) CanvasJNI::restore},
    {"nRestoreToCount","(JI)V", (void*) CanvasJNI::restoreToCount},
    {"nGetMatrix", "(JJ)V", (void*)CanvasJNI::getMatrix},
    {"nSetMatrix","(JJ)V", (void*) CanvasJNI::setMatrix},
    {"nConcat","(JJ)V", (void*) CanvasJNI::concat},
    {"nRotate","(JF)V", (void*) CanvasJNI::rotate},
    {"nScale","(JFF)V", (void*) CanvasJNI::scale},
    {"nSkew","(JFF)V", (void*) CanvasJNI::skew},
    {"nTranslate","(JFF)V", (void*) CanvasJNI::translate},
    {"nQuickReject","(JJ)Z", (void*) CanvasJNI::quickRejectPath},
    {"nQuickReject","(JFFFF)Z", (void*)CanvasJNI::quickRejectRect},
    {"nClipRect","(JFFFFI)Z", (void*) CanvasJNI::clipRect},
    {"nClipPath","(JJI)Z", (void*) CanvasJNI::clipPath},
    {"nSetDrawFilter", "(JJ)V", (void*) CanvasJNI::setPaintFilter},
};

// If called from Canvas these are regular JNI
// If called from DisplayListCanvas they are @FastNative
static const JNINativeMethod gDrawMethods[] = {
    {"nDrawColor","(JII)V", (void*) CanvasJNI::drawColor},
    {"nDrawColor","(JJJI)V", (void*) CanvasJNI::drawColorLong},
    {"nDrawPaint","(JJ)V", (void*) CanvasJNI::drawPaint},
    {"nDrawPoint", "(JFFJ)V", (void*) CanvasJNI::drawPoint},
    {"nDrawPoints", "(J[FIIJ)V", (void*) CanvasJNI::drawPoints},
    {"nDrawLine", "(JFFFFJ)V", (void*) CanvasJNI::drawLine},
    {"nDrawLines", "(J[FIIJ)V", (void*) CanvasJNI::drawLines},
    {"nDrawRect","(JFFFFJ)V", (void*) CanvasJNI::drawRect},
    {"nDrawRegion", "(JJJ)V", (void*) CanvasJNI::drawRegion },
    {"nDrawRoundRect","(JFFFFFFJ)V", (void*) CanvasJNI::drawRoundRect},
    {"nDrawDoubleRoundRect", "(JFFFFFFFFFFFFJ)V", (void*) CanvasJNI::drawDoubleRoundRectXY},
    {"nDrawDoubleRoundRect", "(JFFFF[FFFFF[FJ)V", (void*) CanvasJNI::drawDoubleRoundRectRadii},
    {"nDrawCircle","(JFFFJ)V", (void*) CanvasJNI::drawCircle},
    {"nDrawOval","(JFFFFJ)V", (void*) CanvasJNI::drawOval},
    {"nDrawArc","(JFFFFFFZJ)V", (void*) CanvasJNI::drawArc},
    {"nDrawPath","(JJJ)V", (void*) CanvasJNI::drawPath},
    {"nDrawVertices", "(JII[FI[FI[II[SIIJ)V", (void*)CanvasJNI::drawVertices},
    {"nDrawNinePatch", "(JJJFFFFJII)V", (void*)CanvasJNI::drawNinePatch},
    {"nDrawBitmapMatrix", "(JJJJ)V", (void*)CanvasJNI::drawBitmapMatrix},
    {"nDrawBitmapMesh", "(JJII[FI[IIJ)V", (void*)CanvasJNI::drawBitmapMesh},
    {"nDrawBitmap","(JJFFJIII)V", (void*) CanvasJNI::drawBitmap},
    {"nDrawBitmap","(JJFFFFFFFFJII)V", (void*) CanvasJNI::drawBitmapRect},
    {"nDrawBitmap", "(J[IIIFFIIZJ)V", (void*)CanvasJNI::drawBitmapArray},
    {"nDrawText","(J[CIIFFIJ)V", (void*) CanvasJNI::drawTextChars},
    {"nDrawText","(JLjava/lang/String;IIFFIJ)V", (void*) CanvasJNI::drawTextString},
    {"nDrawTextRun","(J[CIIIIFFZJJ)V", (void*) CanvasJNI::drawTextRunChars},
    {"nDrawTextRun","(JLjava/lang/String;IIIIFFZJ)V", (void*) CanvasJNI::drawTextRunString},
    {"nDrawTextOnPath","(J[CIIJFFIJ)V", (void*) CanvasJNI::drawTextOnPathChars},
    {"nDrawTextOnPath","(JLjava/lang/String;JFFIJ)V", (void*) CanvasJNI::drawTextOnPathString},
};

int register_android_graphics_Canvas(JNIEnv* env) {
    int ret = 0;
    ret |= RegisterMethodsOrDie(env, "android/graphics/Canvas", gMethods, NELEM(gMethods));
    ret |= RegisterMethodsOrDie(env, "android/graphics/BaseCanvas", gDrawMethods, NELEM(gDrawMethods));
    ret |= RegisterMethodsOrDie(env, "android/graphics/BaseRecordingCanvas", gDrawMethods, NELEM(gDrawMethods));
    return ret;

}
```



> 接着我们随便找一个方法drawArc
>
> 方法调用链：
>
> SkiaRecordingCanvas::drawArc -> 
>
> ​	RecordingCanvas::drawArc ->
>
> ​		RecordingCanvas::onDrawArc ->
>
> ​			DisplayListData::drawArc ->
>
> ​				DisplayListData::push<DrawArc> ->
>
> 通过下方方法我们不难分析出：
>
> SkiaRecordingCanvas::drawArc核心逻辑其实就是创建了一个DrawArc存入了DisplayListData中。（并没有进行实际的合成操作）

``` c++
// android_graphics_Canvas.cpp
static void drawArc(JNIEnv* env, jobject, jlong canvasHandle, jfloat left, jfloat top,
                    jfloat right, jfloat bottom, jfloat startAngle, jfloat sweepAngle,
                    jboolean useCenter, jlong paintHandle) {
    const Paint* paint = reinterpret_cast<Paint*>(paintHandle);
    // 注意这里的canvas对象是 SkiaRecordingCanvas 
    get_canvas(canvasHandle)->drawArc(left, top, right, bottom, startAngle, sweepAngle,
                                       useCenter, *paint);
}

//SkiaCanvas.cpp
void SkiaCanvas::drawArc(float left, float top, float right, float bottom, float startAngle,
                         float sweepAngle, bool useCenter, const Paint& paint) {
    if (CC_UNLIKELY(paint.nothingToDraw())) return;
    SkRect arc = SkRect::MakeLTRB(left, top, right, bottom);
    apply_looper(&paint, [&](const SkPaint& p) {
        if (fabs(sweepAngle) >= 360.0f) {
            // 注意这里的mCanvas对象是RecordingCanvas
            mCanvas->drawOval(arc, p);
        } else {
            mCanvas->drawArc(arc, startAngle, sweepAngle, useCenter, p);
        }
    });
}

void SkCanvas::drawArc(const SkRect& oval, SkScalar startAngle,
                       SkScalar sweepAngle, bool useCenter,
                       const SkPaint& paint) {
    TRACE_EVENT0("skia", TRACE_FUNC);
    if (oval.isEmpty() || !sweepAngle) {
        return;
    }
    this->onDrawArc(oval, startAngle, sweepAngle, useCenter, paint);
}

// RecordingCanvas.cpp
void RecordingCanvas::onDrawArc(const SkRect& oval, SkScalar startAngle, SkScalar sweepAngle,
                                bool useCenter, const SkPaint& paint) {
    // 注意dDL实例为DisplayListData
    fDL->drawArc(oval, startAngle, sweepAngle, useCenter, paint);
}

void DisplayListData::drawArc(const SkRect& oval, SkScalar startAngle, SkScalar sweepAngle,
                              bool useCenter, const SkPaint& paint) {
    this->push<DrawArc>(0, oval, startAngle, sweepAngle, useCenter, paint);
}


```



> 补充一下push方法的实现
>
> push方法是一个模板方法。
>
> 主要就干了一件事情——创建T实例(T类型由调用方传入)， 并将T实例保存到fBytes数组中去。

``` c++
/**
** pod  额外的数据长度，调用方在push方法结束后写入
** args 用于创建T实例的额外数据
**/
template <typename T, typename... Args>
void* DisplayListData::push(size_t pod, Args&&... args) {
    // 指针按照4 or 8字节对齐(依据位宽32/64)
    size_t skip = SkAlignPtr(sizeof(T) + pod);
    SkASSERT(skip < (1 << 24));
    // 超出容量大小，进行扩容
    if (fUsed + skip > fReserved) {
        static_assert(SkIsPow2(SKLITEDL_PAGE), "This math needs updating for non-pow2.");
        // Next greater multiple of SKLITEDL_PAGE.
        fReserved = (fUsed + skip + SKLITEDL_PAGE) & ~(SKLITEDL_PAGE - 1);
        fBytes.realloc(fReserved);
    }
    SkASSERT(fUsed + skip <= fReserved);
    // 计算入队的位置
    auto op = (T*)(fBytes.get() + fUsed);
    fUsed += skip;
    // 从入队位置new一个泛型类型实例
    new (op) T{std::forward<Args>(args)...};
    // 设置type & ofsset
    op->type = (uint32_t)T::kType;
    op->skip = skip;
    // 
    return op + 1;
}
```



> 其中T可能的类型如下



![image-20250505214441093](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250505214441093.png)

> 每一个View的每一个drawXXX都对于一个结构体

![image-20250505215155468](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250505215155468.png)







## syncAndDrawFrame



> syncAndDrawFrame实现在native.

``` java
// HardwareRenderer.java
public int syncAndDrawFrame(@NonNull FrameInfo frameInfo) {
        return nSyncAndDrawFrame(mNativeProxy, frameInfo.frameInfo, frameInfo.frameInfo.length);
}
```



> JNI注册

``` c++

const char* const kClassPathName = "android/graphics/HardwareRenderer";

static const JNINativeMethod gMethods[] = {
    // .......
    { "nSyncAndDrawFrame", "(J[JI)I", (void*) android_view_ThreadedRenderer_syncAndDrawFrame },
    // .......
};

int register_android_view_ThreadedRenderer(JNIEnv* env) {
	// .......
    return RegisterMethodsOrDie(env, kClassPathName, gMethods, NELEM(gMethods));
}

```



> JNI方法，调用了proxy->syncAndDrawFrame进行帧绘制
>
> 然而syncAndDrawFrame并没有做啥实际的绘制操作，而是将**绘制任务入队并等待**。

```c++
// android_view_ThreadedRenderer.cpp
static int android_view_ThreadedRenderer_syncAndDrawFrame(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jlongArray frameInfo, jint frameInfoSize) {
    LOG_ALWAYS_FATAL_IF(frameInfoSize != UI_THREAD_FRAME_INFO_SIZE,
            "Mismatched size expectations, given %d expected %d",
            frameInfoSize, UI_THREAD_FRAME_INFO_SIZE);
    // 获取RenderProxy实例
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    // 将frameInfo复制到proxy->frameInfo中（不是重点）
    env->GetLongArrayRegion(frameInfo, 0, frameInfoSize, proxy->frameInfo());
    // 执行帧绘制操作
    return proxy->syncAndDrawFrame();
}

// RenderProxy.cpp
int RenderProxy::syncAndDrawFrame() {
    return mDrawFrameTask.drawFrame();
}

// DrawFrameTask.cpp
int DrawFrameTask::drawFrame() {
    LOG_ALWAYS_FATAL_IF(!mContext, "Cannot drawFrame with no CanvasContext!");

    mSyncResult = SyncResult::OK;
    mSyncQueued = systemTime(SYSTEM_TIME_MONOTONIC);
    postAndWait();

    return mSyncResult;
}

void DrawFrameTask::postAndWait() {
    AutoMutex _lock(mLock);
    // 向工作队列(RenderThread)中入队一个任务
    mRenderThread->queue().post([this]() { run(); });
    // 等待执行结束通知
    mSignal.wait(mLock);
}

```



> 紧接着的run方法的执行线程由main 切换到了RenderThread

``` c++
void DrawFrameTask::run() {
    const int64_t vsyncId = mFrameInfo[static_cast<int>(FrameInfoIndex::FrameTimelineVsyncId)];
    ATRACE_FORMAT("DrawFrames %" PRId64, vsyncId);

    mContext->setSyncDelayDuration(systemTime(SYSTEM_TIME_MONOTONIC) - mSyncQueued);
    mContext->setTargetSdrHdrRatio(mRenderSdrHdrRatio);

    auto hardwareBufferParams = mHardwareBufferParams;
    mContext->setHardwareBufferRenderParams(hardwareBufferParams);
    IRenderPipeline* pipeline = mContext->getRenderPipeline();
    bool canUnblockUiThread;
    bool canDrawThisFrame;
    bool solelyTextureViewUpdates;
    {
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        info.forceDrawFrame = mForceDrawFrame;
        mForceDrawFrame = false;
        // 1. 同步状态信息
        canUnblockUiThread = syncFrameState(info);
        canDrawThisFrame = !info.out.skippedFrameReason.has_value();
        solelyTextureViewUpdates = info.out.solelyTextureViewUpdates;

        if (mFrameCommitCallback) {
            mContext->addFrameCommitListener(std::move(mFrameCommitCallback));
            mFrameCommitCallback = nullptr;
        }
    }

    // Grab a copy of everything we need
    CanvasContext* context = mContext;
    std::function<std::function<void(bool)>(int32_t, int64_t)> frameCallback =
            std::move(mFrameCallback);
    std::function<void()> frameCompleteCallback = std::move(mFrameCompleteCallback);
    mFrameCallback = nullptr;
    mFrameCompleteCallback = nullptr;

    // From this point on anything in "this" is *UNSAFE TO ACCESS*
    if (canUnblockUiThread) {
        unblockUiThread();
    }

    // Even if we aren't drawing this vsync pulse the next frame number will still be accurate
    if (CC_UNLIKELY(frameCallback)) {
        context->enqueueFrameWork([frameCallback, context, syncResult = mSyncResult,
                                   frameNr = context->getFrameNumber()]() {
            auto frameCommitCallback = frameCallback(syncResult, frameNr);
            if (frameCommitCallback) {
                context->addFrameCommitListener(std::move(frameCommitCallback));
            }
        });
    }

    // 2. 绘制
    if (CC_LIKELY(canDrawThisFrame)) {
        context->draw(solelyTextureViewUpdates);
    } else {
        // Do a flush in case syncFrameState performed any texture uploads. Since we skipped
        // the draw() call, those uploads (or deletes) will end up sitting in the queue.
        // Do them now
        if (GrDirectContext* grContext = mRenderThread->getGrContext()) {
            grContext->flushAndSubmit();
        }
        // wait on fences so tasks don't overlap next frame
        context->waitOnFences();
    }

    if (CC_UNLIKELY(frameCompleteCallback)) {
        std::invoke(frameCompleteCallback);
    }
	
    // 3. unblock主线程
    if (!canUnblockUiThread) {
        unblockUiThread();
    }

    if (pipeline->hasHardwareBuffer()) {
        auto fence = pipeline->flush();
        hardwareBufferParams.invokeRenderCallback(std::move(fence), 0);
    }
}
```



### syncFrameState



```c++
// DrawFrameTask.cpp
bool DrawFrameTask::syncFrameState(TreeInfo& info) {
    ATRACE_CALL();
    // 1. 记录当前帧的信息
    int64_t vsync = mFrameInfo[static_cast<int>(FrameInfoIndex::Vsync)];
    int64_t intendedVsync = mFrameInfo[static_cast<int>(FrameInfoIndex::IntendedVsync)];
    int64_t vsyncId = mFrameInfo[static_cast<int>(FrameInfoIndex::FrameTimelineVsyncId)];
    int64_t frameDeadline = mFrameInfo[static_cast<int>(FrameInfoIndex::FrameDeadline)];
    int64_t frameInterval = mFrameInfo[static_cast<int>(FrameInfoIndex::FrameInterval)];
    mRenderThread->timeLord().vsyncReceived(vsync, intendedVsync, vsyncId, frameDeadline,
            frameInterval);
    bool canDraw = mContext->makeCurrent();
    mContext->unpinImages();
	
    // 2. 应用SurfaceTexture的信息
    for (size_t i = 0; i < mLayers.size(); i++) {
        if (mLayers[i]) {
            mLayers[i]->apply();
        }
    }
    mLayers.clear();
    mContext->setContentDrawBounds(mContentDrawBounds);
    // 3. prepareTree
    mContext->prepareTree(info, mFrameInfo, mSyncQueued, mTargetNode);

    // This is after the prepareTree so that any pending operations
    // (RenderNode tree state, prefetched layers, etc...) will be flushed.
    bool hasTarget = mContext->hasOutputTarget();
    if (CC_UNLIKELY(!hasTarget || !canDraw)) {
        if (!hasTarget) {
            mSyncResult |= SyncResult::LostSurfaceRewardIfFound;
            info.out.skippedFrameReason = SkippedFrameReason::NoOutputTarget;
        } else {
            // If we have a surface but can't draw we must be stopped
            mSyncResult |= SyncResult::ContextIsStopped;
            info.out.skippedFrameReason = SkippedFrameReason::ContextIsStopped;
        }
    }

    if (info.out.hasAnimations) {
        if (info.out.requiresUiRedraw) {
            mSyncResult |= SyncResult::UIRedrawRequired;
        }
    }
    if (info.out.skippedFrameReason) {
        mSyncResult |= SyncResult::FrameDropped;
    }
    // If prepareTextures is false, we ran out of texture cache space
    return info.prepareTextures;
}
```



> prepareTree
>
> 1.调用所有renderNode的prepareTree

```c++
// CanvasContext.cpp
void CanvasContext::prepareTree(TreeInfo& info, int64_t* uiFrameInfo, int64_t syncQueued,
                                RenderNode* target) {
    mRenderThread.removeFrameCallback(this);

    // If the previous frame was dropped we don't need to hold onto it, so
    // just keep using the previous frame's structure instead
    if (const auto reason = wasSkipped(mCurrentFrameInfo)) {
        // Use the oldest skipped frame in case we skip more than a single frame
        if (!mSkippedFrameInfo) {
            switch (*reason) {
                case SkippedFrameReason::AlreadyDrawn:
                case SkippedFrameReason::NoBuffer:
                case SkippedFrameReason::NoOutputTarget:
                    mSkippedFrameInfo.emplace();
                    mSkippedFrameInfo->vsyncId =
                            mCurrentFrameInfo->get(FrameInfoIndex::FrameTimelineVsyncId);
                    mSkippedFrameInfo->startTime =
                            mCurrentFrameInfo->get(FrameInfoIndex::FrameStartTime);
                    break;
                case SkippedFrameReason::DrawingOff:
                case SkippedFrameReason::ContextIsStopped:
                case SkippedFrameReason::NothingToDraw:
                    // Do not report those as skipped frames as there was no frame expected to be
                    // drawn
                    break;
            }
        }
    } else {
        mCurrentFrameInfo = mJankTracker.startFrame();
        mSkippedFrameInfo.reset();
    }

    mCurrentFrameInfo->importUiThreadInfo(uiFrameInfo);
    mCurrentFrameInfo->set(FrameInfoIndex::SyncQueued) = syncQueued;
    mCurrentFrameInfo->markSyncStart();

    info.damageAccumulator = &mDamageAccumulator;
    info.layerUpdateQueue = &mLayerUpdateQueue;
    info.damageGenerationId = mDamageId++;
    info.out.skippedFrameReason = std::nullopt;

    mAnimationContext->startFrame(info.mode);
    // 1. 遍历所有的renderNodes，调用其prepareTree方法
    for (const sp<RenderNode>& node : mRenderNodes) {
        // Only the primary target node will be drawn full - all other nodes would get drawn in
        // real time mode. In case of a window, the primary node is the window content and the other
        // node(s) are non client / filler nodes.
        info.mode = (node.get() == target ? TreeInfo::MODE_FULL : TreeInfo::MODE_RT_ONLY);
        node->prepareTree(info);
        GL_CHECKPOINT(MODERATE);
    }
    mAnimationContext->runRemainingAnimations(info);
    GL_CHECKPOINT(MODERATE);

    freePrefetchedLayers();
    GL_CHECKPOINT(MODERATE);

    mIsDirty = true;

    if (CC_UNLIKELY(!hasOutputTarget())) {
        info.out.skippedFrameReason = SkippedFrameReason::NoOutputTarget;
        mCurrentFrameInfo->setSkippedFrameReason(*info.out.skippedFrameReason);
        return;
    }

    if (CC_LIKELY(mSwapHistory.size() && !info.forceDrawFrame)) {
        nsecs_t latestVsync = mRenderThread.timeLord().latestVsync();
        SwapHistory& lastSwap = mSwapHistory.back();
        nsecs_t vsyncDelta = std::abs(lastSwap.vsyncTime - latestVsync);
        // The slight fudge-factor is to deal with cases where
        // the vsync was estimated due to being slow handling the signal.
        // See the logic in TimeLord#computeFrameTimeNanos or in
        // Choreographer.java for details on when this happens
        if (vsyncDelta < 2_ms) {
            // Already drew for this vsync pulse, UI draw request missed
            // the deadline for RT animations
            info.out.skippedFrameReason = SkippedFrameReason::AlreadyDrawn;
        }
    } else {
        info.out.skippedFrameReason = std::nullopt;
    }

    // TODO: Do we need to abort out if the backdrop is added but not ready? Should that even
    // be an allowable combination?
    if (mRenderNodes.size() > 2 && !mRenderNodes[1]->isRenderable()) {
        info.out.skippedFrameReason = SkippedFrameReason::NothingToDraw;
    }

    if (!info.out.skippedFrameReason) {
        int err = mNativeSurface->reserveNext();
        if (err != OK) {
            info.out.skippedFrameReason = SkippedFrameReason::NoBuffer;
            mCurrentFrameInfo->setSkippedFrameReason(*info.out.skippedFrameReason);
            ALOGW("reserveNext failed, error = %d (%s)", err, strerror(-err));
            if (err != TIMED_OUT) {
                // A timed out surface can still recover, but assume others are permanently dead.
                setSurface(nullptr);
                return;
            }
        }
    } else {
        mCurrentFrameInfo->setSkippedFrameReason(*info.out.skippedFrameReason);
    }

    bool postedFrameCallback = false;
    if (info.out.hasAnimations || info.out.skippedFrameReason) {
        if (CC_UNLIKELY(!Properties::enableRTAnimations)) {
            info.out.requiresUiRedraw = true;
        }
        if (!info.out.requiresUiRedraw) {
            // If animationsNeedsRedraw is set don't bother posting for an RT anim
            // as we will just end up fighting the UI thread.
            mRenderThread.postFrameCallback(this);
            postedFrameCallback = true;
        }
    }

    if (!postedFrameCallback &&
        info.out.animatedImageDelay != TreeInfo::Out::kNoAnimatedImageDelay) {
        // Subtract the time of one frame so it can be displayed on time.
        const nsecs_t kFrameTime = mRenderThread.timeLord().frameIntervalNanos();
        if (info.out.animatedImageDelay <= kFrameTime) {
            mRenderThread.postFrameCallback(this);
        } else {
            const auto delay = info.out.animatedImageDelay - kFrameTime;
            int genId = mGenerationID;
            mRenderThread.queue().postDelayed(delay, [this, genId]() {
                if (mGenerationID == genId) {
                    mRenderThread.postFrameCallback(this);
                }
            });
        }
    }
}
```



- RenderNode::prepareTree

RT（RenderThread）驱动

定义：由 渲染线程 主导帧生成，UI 线程仅提交视图的最终属性（如位置、透明度），无需频繁同步中间状态。
适用场景：视图属性变化简单、无需遍历整个视图树时（如动画、滚动）。



MODE_FULL
定义：由 UI 线程 主导帧生成，必须同步视图树的完整状态到渲染线程。
适用场景：视图结构或复杂属性变更（如动态添加/移除视图、修改 ViewGroup 布局）。



> 1.pushStagingPropertiesChanges
>
> 2.prepareForFunctorPresence
>
> 3.prepareLayer
>
> 4.pushStagingDisplayListChanges
>
> 5.prepareListAndChildren

``` c++
void RenderNode::prepareTree(TreeInfo& info) {
    ATRACE_CALL();
    LOG_ALWAYS_FATAL_IF(!info.damageAccumulator, "DamageAccumulator missing");
    MarkAndSweepRemoved observer(&info);

    const int before = info.disableForceDark;
    prepareTreeImpl(observer, info, false);
    LOG_ALWAYS_FATAL_IF(before != info.disableForceDark, "Mis-matched force dark");
}

void RenderNode::prepareTreeImpl(TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer) {
    if (mDamageGenerationId == info.damageGenerationId && mDamageGenerationId != 0) {
        // We hit the same node a second time in the same tree. We don't know the minimal
        // damage rect anymore, so just push the biggest we can onto our parent's transform
        // We push directly onto parent in case we are clipped to bounds but have moved position.
        info.damageAccumulator->dirty(DIRTY_MIN, DIRTY_MIN, DIRTY_MAX, DIRTY_MAX);
    }
    info.damageAccumulator->pushTransform(this);
	// 1. 
    if (info.mode == TreeInfo::MODE_FULL) {
        pushStagingPropertiesChanges(info);
    }

    if (!mProperties.getAllowForceDark()) {
        info.disableForceDark++;
    }
    if (!mProperties.layerProperties().getStretchEffect().isEmpty()) {
        info.stretchEffectCount++;
    }

    uint32_t animatorDirtyMask = 0;
    if (CC_LIKELY(info.runAnimations)) {
        animatorDirtyMask = mAnimatorManager.animate(info);
    }

    bool willHaveFunctor = false;
    if (info.mode == TreeInfo::MODE_FULL && mStagingDisplayList) {
        willHaveFunctor = mStagingDisplayList.hasFunctor();
    } else if (mDisplayList) {
        willHaveFunctor = mDisplayList.hasFunctor();
    }
    bool childFunctorsNeedLayer =
            mProperties.prepareForFunctorPresence(willHaveFunctor, functorsNeedLayer);

    if (CC_UNLIKELY(mPositionListener.get())) {
        mPositionListener->onPositionUpdated(*this, info);
    }

    prepareLayer(info, animatorDirtyMask);
    if (info.mode == TreeInfo::MODE_FULL) {
        pushStagingDisplayListChanges(observer, info);
    }

    // always damageSelf when filtering backdrop content, or else the BackdropFilterDrawable will
    // get a wrong snapshot of previous content.
    if (mProperties.layerProperties().getBackdropImageFilter()) {
        damageSelf(info);
    }

    if (mDisplayList) {
        info.out.hasFunctors |= mDisplayList.hasFunctor();
        mHasHolePunches = mDisplayList.hasHolePunches();
        bool isDirty = mDisplayList.prepareListAndChildren(
                observer, info, childFunctorsNeedLayer,
                [this](RenderNode* child, TreeObserver& observer, TreeInfo& info,
                       bool functorsNeedLayer) {
                    child->prepareTreeImpl(observer, info, functorsNeedLayer);
                    mHasHolePunches |= child->hasHolePunches();
                });
        if (isDirty) {
            damageSelf(info);
        }
    } else {
        mHasHolePunches = false;
    }
    pushLayerUpdate(info);

    if (!mProperties.getAllowForceDark()) {
        info.disableForceDark--;
    }
    if (!mProperties.layerProperties().getStretchEffect().isEmpty()) {
        info.stretchEffectCount--;
    }
    info.damageAccumulator->popTransform();
}
```



#### pushStagingPropertiesChanges

> 用于同步RenderNode的property

1.同步PositionListener

2.同步dirty properties

```c++
void RenderNode::pushStagingPropertiesChanges(TreeInfo& info) {
    // 1. 赋值PositionListener
    if (mPositionListenerDirty) {
        mPositionListener = std::move(mStagingPositionListener);
        mStagingPositionListener = nullptr;
        mPositionListenerDirty = false;
    }

    // Push the animators first so that setupStartValueIfNecessary() is called
    // before properties() is trampled by stagingProperties(), as they are
    // required by some animators.
    if (CC_LIKELY(info.runAnimations)) {
        mAnimatorManager.pushStaging();
    }
    
    if (mDirtyPropertyFields) { // renderNode Property变化的列表, 是一个二进制表示的掩码。
        mDirtyPropertyFields = 0;
        damageSelf(info);
        info.damageAccumulator->popTransform();
        // 同步property信息
        // 其实本质上就是改引用
        syncProperties();

        auto& layerProperties = mProperties.layerProperties();
        const StretchEffect& stagingStretch = layerProperties.getStretchEffect();
        if (stagingStretch.isEmpty()) {
            mStretchMask.clear();
        }

        if (layerProperties.getImageFilter() == nullptr) {
            mSnapshotResult.snapshot = nullptr;
            mTargetImageFilter = nullptr;
        }

        // We could try to be clever and only re-damage if the matrix changed.
        // However, we don't need to worry about that. The cost of over-damaging
        // here is only going to be a single additional map rect of this node
        // plus a rect join(). The parent's transform (and up) will only be
        // performed once.
        info.damageAccumulator->pushTransform(this);
        damageSelf(info);
    }
}
```



#### pushStagingDisplayListChanges



> 同步displayList

``` c++
void RenderNode::pushStagingDisplayListChanges(TreeObserver& observer, TreeInfo& info) {
    if (mNeedsDisplayListSync) {
        mNeedsDisplayListSync = false;
        // Damage with the old display list first then the new one to catch any
        // changes in isRenderable or, in the future, bounds
        damageSelf(info);
        syncDisplayList(observer, &info);
        damageSelf(info);
    }
}
```



```c++
void RenderNode::syncDisplayList(TreeObserver& observer, TreeInfo* info) {
    // Make sure we inc first so that we don't fluctuate between 0 and 1,
    // which would thrash the layer cache
    // 新增子节点引用， 防止内存被释放
    if (mStagingDisplayList) {
        mStagingDisplayList.updateChildren([](RenderNode* child) { child->incParentRefCount(); });
    }
    // 删除旧的显示列表时减少引用
    deleteDisplayList(observer, info);
    // 同步displayList(核心)
    mDisplayList = std::move(mStagingDisplayList);
    if (mDisplayList) {
        WebViewSyncData syncData{.applyForceDark = shouldEnableForceDark(info)};
        // 同步显示内容
        mDisplayList.syncContents(syncData);
        handleForceDark(info);
    }
}

```







#### DisplayList.prepareListAndChildren 



> 主要递归进行脏区计算

```c++
// RenderNode.cpp
// 1. rootRender调用child
void RenderNode::prepareTreeImpl(TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer) {
 
    
    if (mDisplayList) {
        // 收集必要的参数
        info.out.hasFunctors |= mDisplayList.hasFunctor();
        mHasHolePunches = mDisplayList.hasHolePunches();
        // 递归进行脏区计算
        bool isDirty = mDisplayList.prepareListAndChildren(
                observer, info, childFunctorsNeedLayer,
                [this](RenderNode* child, TreeObserver& observer, TreeInfo& info,
                       bool functorsNeedLayer) {
                    child->prepareTreeImpl(observer, info, functorsNeedLayer);
                    mHasHolePunches |= child->hasHolePunches();
                });
        if (isDirty) {
            damageSelf(info);
        }
    } else {
        mHasHolePunches = false;
    }
    
    
}


// DisplayList.h
// 2. renderNode遍历chil节点
bool prepareListAndChildren(
        TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer,
        std::function<void(RenderNode*, TreeObserver&, TreeInfo&, bool)> childFn) {
    return mImpl && mImpl->prepareListAndChildren(
            observer, info, functorsNeedLayer, std::move(childFn));
}

// SkiaDisplayList.cpp
// 3. 遍历child节点
bool SkiaDisplayList::prepareListAndChildren(
        TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer,
        std::function<void(RenderNode*, TreeObserver&, TreeInfo&, bool)> childFn) {
    // If the prepare tree is triggered by the UI thread and no previous call to
    // pinImages has failed then we must pin all mutable images in the GPU cache
    // until the next UI thread draw.
    // 处理部分android特有的逻辑
#ifdef __ANDROID__ // Layoutlib does not support CanvasContext
    if (info.prepareTextures && !info.canvasContext.pinImages(mMutableImages)) {
        // In the event that pinning failed we prevent future pinImage calls for the
        // remainder of this tree traversal and also unpin any currently pinned images
        // to free up GPU resources.
        info.prepareTextures = false;
        info.canvasContext.unpinImages();
    }

    auto grContext = info.canvasContext.getGrContext();
    for (auto mesh : mMeshes) {
        mesh->updateSkMesh(grContext);
    }

#endif

    bool hasBackwardProjectedNodesHere = false;
    bool hasBackwardProjectedNodesSubtree = false;
	
    // 逐一遍历所有的child节点
    for (auto& child : mChildNodes) {
        RenderNode* childNode = child.getRenderNode();
        Matrix4 mat4(child.getRecordedMatrix());
        info.damageAccumulator->pushTransform(&mat4);
        info.hasBackwardProjectedNodes = false;
        // 调用传入的function
        // 可以回过头看#1.rootRender调用child
        // childFuc内部调用了child->prepareTreeImpl
        childFn(childNode, observer, info, functorsNeedLayer);
        hasBackwardProjectedNodesHere |= child.getNodeProperties().getProjectBackwards();
        hasBackwardProjectedNodesSubtree |= info.hasBackwardProjectedNodes;
        info.damageAccumulator->popTransform();
    }

    // The purpose of next block of code is to reset projected display list if there are no
    // backward projected nodes. This speeds up drawing, by avoiding an extra walk of the tree
    if (mProjectionReceiver) {
        mProjectionReceiver->setProjectedDisplayList(hasBackwardProjectedNodesSubtree ? this
                                                                                      : nullptr);
        info.hasBackwardProjectedNodes = hasBackwardProjectedNodesHere;
    } else {
        info.hasBackwardProjectedNodes =
                hasBackwardProjectedNodesSubtree || hasBackwardProjectedNodesHere;
    }

    bool isDirty = false;
    for (auto& animatedImage : mAnimatedImages) {
        nsecs_t timeTilNextFrame = TreeInfo::Out::kNoAnimatedImageDelay;
        // If any animated image in the display list needs updated, then damage the node.
        if (animatedImage->isDirty(&timeTilNextFrame)) {
            isDirty = true;
        }

        if (animatedImage->isRunning() &&
            timeTilNextFrame != TreeInfo::Out::kNoAnimatedImageDelay) {
            auto& delay = info.out.animatedImageDelay;
            if (delay == TreeInfo::Out::kNoAnimatedImageDelay || timeTilNextFrame < delay) {
                delay = timeTilNextFrame;
            }
        }
    }

    for (auto& [vectorDrawable, cachedMatrix] : mVectorDrawables) {
        // If any vector drawable in the display list needs update, damage the node.
        if (vectorDrawable->isDirty()) {
            Matrix4 totalMatrix;
            info.damageAccumulator->computeCurrentTransform(&totalMatrix);
            Matrix4 canvasMatrix(cachedMatrix);
            totalMatrix.multiply(canvasMatrix);
            const SkRect& bounds = vectorDrawable->properties().getBounds();
            if (intersects(info.screenSize, totalMatrix, bounds)) {
                isDirty = true;
                vectorDrawable->setPropertyChangeWillBeConsumed(true);
            }
        }
    }
    return isDirty;
}
```











### draw



```c++
void CanvasContext::draw(bool solelyTextureViewUpdates) {
    // check GrContext是否状态异常
    // ......
    
    SkRect dirty;
    mDamageAccumulator.finish(&dirty);

    // reset syncDelayDuration each time we draw
    nsecs_t syncDelayDuration = mSyncDelayDuration;
    nsecs_t idleDuration = mIdleDuration;
    mSyncDelayDuration = 0;
    mIdleDuration = 0;

    const auto skippedFrameReason = [&]() -> std::optional<SkippedFrameReason> {
        if (!Properties::isDrawingEnabled()) {
            return SkippedFrameReason::DrawingOff;
        }

        if (dirty.isEmpty() && Properties::skipEmptyFrames && !surfaceRequiresRedraw()) {
            return SkippedFrameReason::NothingToDraw;
        }

        return std::nullopt;
    }();
    // 跳过绘制
    if (skippedFrameReason) {
        mCurrentFrameInfo->setSkippedFrameReason(*skippedFrameReason);

        if (auto grContext = getGrContext()) {
            // Submit to ensure that any texture uploads complete and Skia can
            // free its staging buffers.
            grContext->flushAndSubmit();
        }

        // Notify the callbacks, even if there's nothing to draw so they aren't waiting
        // indefinitely
        waitOnFences();
        for (auto& func : mFrameCommitCallbacks) {
            std::invoke(func, false /* didProduceBuffer */);
        }
        mFrameCommitCallbacks.clear();
        return;
    }

    ScopedActiveContext activeContext(this);
    mCurrentFrameInfo->set(FrameInfoIndex::FrameInterval) =
            mRenderThread.timeLord().frameIntervalNanos();

    mCurrentFrameInfo->markIssueDrawCommandsStart();

    Frame frame = getFrame();

    SkRect windowDirty = computeDirtyRect(frame, &dirty);

    ATRACE_FORMAT("Drawing " RECT_STRING, SK_RECT_ARGS(dirty));

    IRenderPipeline::DrawResult drawResult;
    {
        // FrameInfoVisualizer accesses the frame events, which cannot be mutated mid-draw
        // or it can lead to memory corruption.
        drawResult = mRenderPipeline->draw(
                frame, windowDirty, dirty, mLightGeometry, &mLayerUpdateQueue, mContentDrawBounds,
                mOpaque, mLightInfo, mRenderNodes, &(profiler()), mBufferParams, profilerLock());
    }

    uint64_t frameCompleteNr = getFrameNumber();

    waitOnFences();

    if (mNativeSurface) {
        // TODO(b/165985262): measure performance impact
        const auto vsyncId = mCurrentFrameInfo->get(FrameInfoIndex::FrameTimelineVsyncId);
        if (vsyncId != UiFrameInfoBuilder::INVALID_VSYNC_ID) {
            const auto inputEventId =
                    static_cast<int32_t>(mCurrentFrameInfo->get(FrameInfoIndex::InputEventId));
            const ANativeWindowFrameTimelineInfo ftl = {
                    .frameNumber = frameCompleteNr,
                    .frameTimelineVsyncId = vsyncId,
                    .inputEventId = inputEventId,
                    .startTimeNanos = mCurrentFrameInfo->get(FrameInfoIndex::FrameStartTime),
                    .useForRefreshRateSelection = solelyTextureViewUpdates,
                    .skippedFrameVsyncId = mSkippedFrameInfo ? mSkippedFrameInfo->vsyncId
                                                             : UiFrameInfoBuilder::INVALID_VSYNC_ID,
                    .skippedFrameStartTimeNanos =
                            mSkippedFrameInfo ? mSkippedFrameInfo->startTime : 0,
            };
            native_window_set_frame_timeline_info(mNativeSurface->getNativeWindow(), ftl);
        }
    }

    bool requireSwap = false;
    bool didDraw = false;

    int error = OK;
    bool didSwap = mRenderPipeline->swapBuffers(frame, drawResult, windowDirty, mCurrentFrameInfo,
                                                &requireSwap);

    mCurrentFrameInfo->set(FrameInfoIndex::CommandSubmissionCompleted) = std::max(
            drawResult.commandSubmissionTime, mCurrentFrameInfo->get(FrameInfoIndex::SwapBuffers));

    mIsDirty = false;

    if (requireSwap) {
        didDraw = true;
        // Handle any swapchain errors
        error = mNativeSurface->getAndClearError();
        if (error == TIMED_OUT) {
            // Try again
            mRenderThread.postFrameCallback(this);
            // But since this frame didn't happen, we need to mark full damage in the swap
            // history
            didDraw = false;

        } else if (error != OK || !didSwap) {
            // Unknown error, abandon the surface
            setSurface(nullptr);
            didDraw = false;
        }

        SwapHistory& swap = mSwapHistory.next();
        if (didDraw) {
            swap.damage = windowDirty;
        } else {
            float max = static_cast<float>(INT_MAX);
            swap.damage = SkRect::MakeWH(max, max);
        }
        swap.swapCompletedTime = systemTime(SYSTEM_TIME_MONOTONIC);
        swap.vsyncTime = mRenderThread.timeLord().latestVsync();
        if (didDraw) {
            nsecs_t dequeueStart =
                    ANativeWindow_getLastDequeueStartTime(mNativeSurface->getNativeWindow());
            if (dequeueStart < mCurrentFrameInfo->get(FrameInfoIndex::SyncStart)) {
                // Ignoring dequeue duration as it happened prior to frame render start
                // and thus is not part of the frame.
                swap.dequeueDuration = 0;
            } else {
                swap.dequeueDuration =
                        ANativeWindow_getLastDequeueDuration(mNativeSurface->getNativeWindow());
            }
            swap.queueDuration =
                    ANativeWindow_getLastQueueDuration(mNativeSurface->getNativeWindow());
        } else {
            swap.dequeueDuration = 0;
            swap.queueDuration = 0;
        }
        mCurrentFrameInfo->set(FrameInfoIndex::DequeueBufferDuration) = swap.dequeueDuration;
        mCurrentFrameInfo->set(FrameInfoIndex::QueueBufferDuration) = swap.queueDuration;
        mHaveNewSurface = false;
        mFrameNumber = 0;
    } else {
        mCurrentFrameInfo->set(FrameInfoIndex::DequeueBufferDuration) = 0;
        mCurrentFrameInfo->set(FrameInfoIndex::QueueBufferDuration) = 0;
    }

    mCurrentFrameInfo->markSwapBuffersCompleted();

#if LOG_FRAMETIME_MMA
    float thisFrame = mCurrentFrameInfo->duration(FrameInfoIndex::IssueDrawCommandsStart,
                                                  FrameInfoIndex::FrameCompleted) /
                      NANOS_PER_MILLIS_F;
    if (sFrameCount) {
        sBenchMma = ((9 * sBenchMma) + thisFrame) / 10;
    } else {
        sBenchMma = thisFrame;
    }
    if (++sFrameCount == 10) {
        sFrameCount = 1;
        ALOGD("Average frame time: %.4f", sBenchMma);
    }
#endif

    if (didSwap) {
        for (auto& func : mFrameCommitCallbacks) {
            std::invoke(func, true /* didProduceBuffer */);
        }
        mFrameCommitCallbacks.clear();
    }

    if (requireSwap) {
        if (mExpectSurfaceStats) {
            reportMetricsWithPresentTime();
            {  // acquire lock
                std::lock_guard lock(mLast4FrameMetricsInfosMutex);
                FrameMetricsInfo& next = mLast4FrameMetricsInfos.next();
                next.frameInfo = mCurrentFrameInfo;
                next.frameNumber = frameCompleteNr;
                next.surfaceId = mSurfaceControlGenerationId;
            }  // release lock
        } else {
            mCurrentFrameInfo->markFrameCompleted();
            mCurrentFrameInfo->set(FrameInfoIndex::GpuCompleted)
                    = mCurrentFrameInfo->get(FrameInfoIndex::FrameCompleted);
            std::scoped_lock lock(mFrameInfoMutex);
            mJankTracker.finishFrame(*mCurrentFrameInfo, mFrameMetricsReporter, frameCompleteNr,
                                     mSurfaceControlGenerationId);
        }
    }

    int64_t intendedVsync = mCurrentFrameInfo->get(FrameInfoIndex::IntendedVsync);
    int64_t frameDeadline = mCurrentFrameInfo->get(FrameInfoIndex::FrameDeadline);
    int64_t dequeueBufferDuration = mCurrentFrameInfo->get(FrameInfoIndex::DequeueBufferDuration);

    mHintSessionWrapper->updateTargetWorkDuration(frameDeadline - intendedVsync);

    if (didDraw) {
        int64_t frameStartTime = mCurrentFrameInfo->get(FrameInfoIndex::FrameStartTime);
        int64_t frameDuration = systemTime(SYSTEM_TIME_MONOTONIC) - frameStartTime;
        int64_t actualDuration = frameDuration -
                                 (std::min(syncDelayDuration, mLastDequeueBufferDuration)) -
                                 dequeueBufferDuration - idleDuration;
        mHintSessionWrapper->reportActualWorkDuration(actualDuration);
    }

    mLastDequeueBufferDuration = dequeueBufferDuration;

    mRenderThread.cacheManager().onFrameCompleted();
    return;
}
```





#### issueDrawCommand

``` c++
void CanvasContext::draw(bool solelyTextureViewUpdates) {
    
    // ......
 
    // 1. issueDrawCommand
    IRenderPipeline::DrawResult drawResult;
    {
        // FrameInfoVisualizer accesses the frame events, which cannot be mutated mid-draw
        // or it can lead to memory corruption.
        drawResult = mRenderPipeline->draw(
                frame, windowDirty, dirty, mLightGeometry, &mLayerUpdateQueue, mContentDrawBounds,
                mOpaque, mLightInfo, mRenderNodes, &(profiler()), mBufferParams, profilerLock());
    }
    
    // 2. swapBuffers
    bool didSwap = mRenderPipeline->swapBuffers(frame, drawResult, windowDirty, mCurrentFrameInfo,
                                                &requireSwap);
    
    
}
```



>  issueDrawCommand的核心在于**mRenderPipeline->draw**
>
> 而这个draw方法内部其实是将displayList的drawOps同步到渲染管线

```c++
// SkiaOpenGLPipeline.cpp
IRenderPipeline::DrawResult SkiaOpenGLPipeline::draw(
        const Frame& frame, const SkRect& screenDirty, const SkRect& dirty,
        const LightGeometry& lightGeometry, LayerUpdateQueue* layerUpdateQueue,
        const Rect& contentDrawBounds, bool opaque, const LightInfo& lightInfo,
        const std::vector<sp<RenderNode>>& renderNodes, FrameInfoVisualizer* profiler,
        const HardwareBufferRenderParams& bufferParams, std::mutex& profilerLock) {
    if (!isCapturingSkp() && !mHardwareBuffer) {
        mEglManager.damageFrame(frame, dirty);
    }

    SkColorType colorType = getSurfaceColorType();
    // setup surface for fbo0
    GrGLFramebufferInfo fboInfo;
    fboInfo.fFBOID = 0;
    if (colorType == kRGBA_F16_SkColorType) {
        fboInfo.fFormat = GL_RGBA16F;
    } else if (colorType == kN32_SkColorType) {
        // Note: The default preference of pixel format is RGBA_8888, when other
        // pixel format is available, we should branch out and do more check.
        fboInfo.fFormat = GL_RGBA8;
    } else if (colorType == kRGBA_1010102_SkColorType) {
        fboInfo.fFormat = GL_RGB10_A2;
    } else if (colorType == kAlpha_8_SkColorType) {
        fboInfo.fFormat = GL_R8;
    } else {
        LOG_ALWAYS_FATAL("Unsupported color type.");
    }
	// 创建renderTarget用于后续创建surface
    auto backendRT = GrBackendRenderTargets::MakeGL(frame.width(), frame.height(), 0,
                                                    STENCIL_BUFFER_SIZE, fboInfo);

    SkSurfaceProps props(mColorMode == ColorMode::Default ? 0 : SkSurfaceProps::kAlwaysDither_Flag,
                         kUnknown_SkPixelGeometry);

    SkASSERT(mRenderThread.getGrContext() != nullptr);
    sk_sp<SkSurface> surface;
    SkMatrix preTransform;
    // 创建surface
    if (mHardwareBuffer) {
        surface = getBufferSkSurface(bufferParams);
        preTransform = bufferParams.getTransform();
    } else {
        surface = SkSurfaces::WrapBackendRenderTarget(mRenderThread.getGrContext(), backendRT,
                                                      getSurfaceOrigin(), colorType,
                                                      mSurfaceColorSpace, &props);
        preTransform = SkMatrix::I();
    }

    SkPoint lightCenter = preTransform.mapXY(lightGeometry.center.x, lightGeometry.center.y);
    LightGeometry localGeometry = lightGeometry;
    localGeometry.center.x = lightCenter.fX;
    localGeometry.center.y = lightCenter.fY;
    LightingInfo::updateLighting(localGeometry, lightInfo);
    // 渲染一帧
    renderFrame(*layerUpdateQueue, dirty, renderNodes, opaque, contentDrawBounds, surface,
                preTransform);

    // Draw visual debugging features
    if (CC_UNLIKELY(Properties::showDirtyRegions ||
                    ProfileType::None != Properties::getProfileType())) {
        std::scoped_lock lock(profilerLock);
        SkCanvas* profileCanvas = surface->getCanvas();
        SkiaProfileRenderer profileRenderer(profileCanvas, frame.width(), frame.height());
        profiler->draw(profileRenderer);
    }

    {
        ATRACE_NAME("flush commands");
        skgpu::ganesh::FlushAndSubmit(surface);
    }
    layerUpdateQueue->clear();

    // Log memory statistics
    if (CC_UNLIKELY(Properties::debugLevel != kDebugDisabled)) {
        dumpResourceCacheUsage();
    }

    return {true, IRenderPipeline::DrawResult::kUnknownTime, android::base::unique_fd{}};
}
```



- renderFrame

```c++
// SkiaPipeline.cpp

void SkiaPipeline::renderFrame(const LayerUpdateQueue& layers, const SkRect& clip,
                               const std::vector<sp<RenderNode>>& nodes, bool opaque,
                               const Rect& contentDrawBounds, sk_sp<SkSurface> surface,
                               const SkMatrix& preTransform) {
    bool previousSkpEnabled = Properties::skpCaptureEnabled;
    if (mPictureCapturedCallback) {
        Properties::skpCaptureEnabled = true;
    }

    // Initialize the canvas for the current frame, that might be a recording canvas if SKP
    // capture is enabled.
    SkCanvas* canvas = tryCapture(surface.get(), nodes[0].get(), layers);

    // draw all layers up front
    // 将离屏渲染的图层（如 View.setLayerType(LAYER_TYPE_HARDWARE)内容绘制到各自的纹理。
    renderLayersImpl(layers, opaque);

    // 渲染View数据，将DisplayList的数据渲染到SkCanvas中
    renderFrameImpl(clip, nodes, opaque, contentDrawBounds, canvas, preTransform);

    endCapture(surface.get());

    if (CC_UNLIKELY(Properties::debugOverdraw)) {
        renderOverdraw(clip, nodes, contentDrawBounds, surface, preTransform);
    }

    Properties::skpCaptureEnabled = previousSkpEnabled;
}
```





#### swapBuffers



```c++
// SkiaOpenGLPipeline.cpp

bool SkiaOpenGLPipeline::swapBuffers(const Frame& frame, IRenderPipeline::DrawResult& drawResult,
                                     const SkRect& screenDirty, FrameInfo* currentFrameInfo,
                                     bool* requireSwap) {
    GL_CHECKPOINT(LOW);

    // Even if we decided to cancel the frame, from the perspective of jank
    // metrics the frame was swapped at this point
    currentFrameInfo->markSwapBuffers();

    if (mHardwareBuffer) {
        return false;
    }

    *requireSwap = drawResult.success || mEglManager.damageRequiresSwap();

    if (*requireSwap && (CC_UNLIKELY(!mEglManager.swapBuffers(frame, screenDirty)))) {
        return false;
    }

    return *requireSwap;
}
```



> Egl

```c++
// EglManager.cpp
bool EglManager::swapBuffers(const Frame& frame, const SkRect& screenDirty) {
    // 仅在调试模式（如 adb shell setprop debug.egl.sync true）启用，用于 排查渲染时序问题。
    if (CC_UNLIKELY(Properties::waitForGpuCompletion)) {
        ATRACE_NAME("Finishing GPU work");
        // 向 GPU 提交一个同步栅栏（Fence），阻塞 CPU 直到 GPU 完成当前队列中的所有任务，确保渲染完整性（牺牲性能换取确定性）。
        fence();
    }

    EGLint rects[4];
    // 将 Skia 坐标系下的脏区域转换为 EGL 所需的格式
    frame.map(screenDirty, rects);
    // 将后台缓冲区（Back Buffer）中已渲染的内容提交到前台显示（Display），并指定本次提交需要更新的屏幕区域（脏区域）
    eglSwapBuffersWithDamageKHR(mEglDisplay, frame.mSurface, rects, screenDirty.isEmpty() ? 0 : 1);

    // 处理执行结果
    EGLint err = eglGetError();
    if (CC_LIKELY(err == EGL_SUCCESS)) {
        return true;
    }
    if (err == EGL_BAD_SURFACE || err == EGL_BAD_NATIVE_WINDOW) {
        // For some reason our surface was destroyed out from under us
        // This really shouldn't happen, but if it does we can recover easily
        // by just not trying to use the surface anymore
        ALOGW("swapBuffers encountered EGL error %d on %p, halting rendering...", err,
              frame.mSurface);
        return false;
    }
    LOG_ALWAYS_FATAL("Encountered EGL error %d %s during rendering", err, egl_error_str(err));
    // Impossible to hit this, but the compiler doesn't know that
    return false;
}
```







#### 



