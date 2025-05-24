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

    	// 3. 根节点发生变化，重新生成rootNode(也不是重点)
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
    // 向工作队列中入队一个任务
    mRenderThread->queue().post([this]() { run(); });
    // 等待执行结束通知
    mSignal.wait(mLock);
}

```











# Canvas







# JNI











