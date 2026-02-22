---
title: AOSP UI渲染系统
tags:
cover:
---
# AOSP UI渲染系统

![image rendering components](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/ape-fwk-graphics.png)

# 关于本文

只是大致讲解一下AOSP UI显示系统的概览，不会做细致分析，初来分析UI系统，做roadmap


# View绘制

**核心角色**：`ViewRootImpl`（根节点）、`DecorView`（根ViewGroup）、`View/ViewGroup`（具体内容）、`Choreographer`（帧同步）。
这一步骤主要做的就是View的渲染操作，包含View的measure/layout/draw以及RenderThread的绘制操作，最后将数据同步到BufferQueue中。


**流程与作用**：
- **触发时机**：Activity的`onResume()`后，`ViewRootImpl`通过`WindowManager.addView()`将`DecorView`（包含`setContentView()`加载的布局）添加到窗口，触发首次绘制。后续通过`requestLayout()`（尺寸/位置变化）或`invalidate()`（外观变化）触发重绘。
    
- **帧同步**：`Choreographer`接收**VSync信号**（16.6ms/帧，60Hz），回调`doFrame()`，触发`ViewRootImpl.scheduleTraversals()`，启动**测量（Measure）→ 布局（Layout）→ 绘制（Draw）**三大步骤。
    
- **测量（Measure）**：从`DecorView`开始，递归计算每个View的尺寸（`measuredWidth`/`measuredHeight`）。父View通过`MeasureSpec`（约束模式：`EXACTLY`/`AT_MOST`/`UNSPECIFIED`）传递给子View，子View重写`onMeasure()`处理约束，调用`setMeasuredDimension()`保存结果。
    
- **布局（Layout）**：确定每个View的位置（`left`/`top`/`right`/`bottom`）。父View重写`onLayout()`，为每个子View调用`child.layout()`，保存位置信息。
    
- **绘制（Draw）**：将View内容渲染到`Surface`的缓冲区。
    
    - 软件绘制：直接通过`Canvas`栅格化（如`drawRect()`、`drawText()`），调用`unlockCanvasAndPost()`提交缓冲区。
        
    - 硬件加速（默认）：将View录制成`RenderNode`（显示列表），由`RenderThread`（独立于UI线程）合成，通过`eglSwapBuffers()`提交到`SurfaceFlinger`。

# WMS

**核心角色**：`WindowManagerService`（系统服务）、`Window`（窗口抽象）、`Layer`（SurfaceFlinger层）。

**流程与作用**：

- **窗口注册**：应用通过`WindowManager.addView()`将`DecorView`添加到`WMS`，`WMS`为每个窗口创建`Layer`（对应`Surface`），分发处理Input事件，并记录窗口属性（层级/Z序、位置、大小、透明度、可见性）。
    
- **属性同步**：当窗口发生变化（如旋转、 resize），`WMS`更新`Layer`属性，并通知`SurfaceFlinger`调整合成策略。
    
- **层级管理**：窗口分为**应用窗口**（如Activity）、**子窗口**（如Dialog）、**系统窗口**（如状态栏），层级依次递增（1000-1999/2000-2999），确保正确的遮挡关系。


# SurfaceFlinger


**核心角色**：`SurfaceFlinger`（系统合成服务）、`Layer`（图层）、`Hardware Composer（HWC）`（硬件合成）、`GPU`（软件合成）。

**流程与作用**：

- **缓冲区获取**：`SurfaceFlinger`循环监听`VSync`信号，从每个`Layer`的`BufferQueue`中`acquireBuffer()`获取最新缓冲区（若有未提交的缓冲区，沿用上一帧）。
    
- **合成策略**：根据`WMS`提供的`Layer`属性（Z序、位置、透明度），选择**硬件合成（HWC）**或**软件合成（GPU）**：
    
    - 硬件合成（优先）：`HWC`直接将多个`Layer`缓冲区合成到屏幕（如Overlay层），减少GPU开销。
        
    - 软件合成：若`Layer`不支持硬件合成（如复杂变换），`GPU`将`Layer`渲染到临时缓冲区，再合成输出。
        
    
- **输出到屏幕**：合成后的最终画面通过**显示驱动**（如DRM/KMS）发送到屏幕，完成一帧渲染。





## SurfaceFlinger架构

SurfaceFlinger -> mCompositionEngine(CompositionEngine) -> RenderEngine -> HWComposer



## Vsync


### CompositionEngine创建

```c++
SurfaceFlinger::SurfaceFlinger(Factory& factory, SkipInitializationTag)
      : mFactory(factory),
        mPid(getpid()),
        mTimeStats(std::make_shared<impl::TimeStats>()),
        mFrameTracer(mFactory.createFrameTracer()),
        mFrameTimeline(mFactory.createFrameTimeline(mTimeStats, mPid)),
        // 关注点
        mCompositionEngine(mFactory.createCompositionEngine()),
        mHwcServiceName(base::GetProperty("debug.sf.hwc_service_name"s, "default"s)),
        mTunnelModeEnabledReporter(sp<TunnelModeEnabledReporter>::make()),
        mEmulatedDisplayDensity(getDensityFromProperty("qemu.sf.lcd_density", false)),
        mInternalDisplayDensity(
                getDensityFromProperty("ro.sf.lcd_density", !mEmulatedDisplayDensity)),
        mPowerAdvisor(std::make_unique<Hwc2::impl::PowerAdvisor>(*this)),
        mWindowInfosListenerInvoker(sp<WindowInfosListenerInvoker>::make()) {
    ALOGI("Using HWComposer service: %s", mHwcServiceName.c_str());
}

// /frameworks/native/services/surfaceflinger/SurfaceFlingerDefaultFactory.cpp
std::unique_ptr<compositionengine::CompositionEngine> DefaultFactory::createCompositionEngine() {
    return compositionengine::impl::createCompositionEngine();
}

// /frameworks/native/services/surfaceflinger/CompositionEngine/src/CompositionEngine.cpp
std::unique_ptr<compositionengine::CompositionEngine> createCompositionEngine() {
    return std::make_unique<CompositionEngine>();
}
```


### RenderEngine初始化


1.创建RenderNode
2.使用surfaceflinger::Factory创建HWComposer
3.
```java
// /frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp

void SurfaceFlinger::init() FTL_FAKE_GUARD(kMainThreadContext) {
    // .....

    // 关注点 1
    // 构建 RenderEngine，用于 Client 合成
    auto builder = renderengine::RenderEngineCreationArgs::Builder()
                           .    1`(static_cast<int32_t>(defaultCompositionPixelFormat))
                           .setImageCacheSize(maxFrameBufferAcquiredBuffers)
                           .setUseColorManagerment(useColorManagement)
                           .setEnableProtectedContext(enable_protected_contents(false))
                           .setPrecacheToneMapperShaderOnly(false)
                           .setSupportsBackgroundBlur(mSupportsBlur)
                           .setContextPriority(
                                   useContextPriority
                                           ? renderengine::RenderEngine::ContextPriority::REALTIME
                                           : renderengine::RenderEngine::ContextPriority::MEDIUM);
    
    if (auto type = chooseRenderEngineTypeViaSysProp()) {
        builder.setRenderEngineType(type.value());
    }

    mRenderEngine = renderengine::RenderEngine::create(builder.build());
    mCompositionEngine->setRenderEngine(mRenderEngine.get());

    // ......
    
    // 关注点2
    // 创建HWComposer对象并传入一个name属性，再通过mCompositionEngine->setHwComposer设置对象属性。
    mCompositionEngine->setHwComposer(getFactory().createHWComposer(mHwcServiceName));
    // 这里的this就是SurfaceFlinger对象本身，因为它实现了HWC2::ComposerCallback回调接口
    mCompositionEngine->getHwComposer().setCallback(*this);
    // ......


    // Commit primary display.
    sp<const DisplayDevice> display;
    if (const auto indexOpt = mCurrentState.getDisplayIndex(getPrimaryDisplayIdLocked())) {
        const auto& displays = mCurrentState.displays;

        const auto& token = displays.keyAt(*indexOpt);
        const auto& state = displays.valueAt(*indexOpt);
         // 关注点3
        processDisplayAdded(token, state);

        mDrawingState.displays.add(token, state);

        //
        display = getDefaultDisplayDeviceLocked();
    }

    //......
    // 关注点4
    initScheduler(display);
    // ......
}
```

### HWComposer

```c++
// 关注点2
// 创建HWComposer对象并传入一个name属性，再通过mCompositionEngine->setHwComposer设置对象属性。
mCompositionEngine->setHwComposer(getFactory().createHWComposer(mHwcServiceName));
// 这里的this就是SurfaceFlinger对象本身，因为它实现了HWC2::ComposerCallback回调接口
mCompositionEngine->getHwComposer().setCallback(*this);
```



### HWComposer 回调注册



vsync实现原理：

https://juejin.cn/post/7397410962848071717?searchId=20260207004351160DAE881DB25C30CACE#:~:text=%E5%9B%BE%E6%89%80%E7%A4%BA%EF%BC%9A-,3.3%20HWComposer%20%E5%88%9D%E5%A7%8B%E5%8C%96,-%E6%8E%A5%E7%9D%80%E5%9B%9E%E5%88%B0%20init

HWC渲染合成：
https://juejin.cn/post/7412848251844624424?searchId=202602070145220A82EE03A3A72D2CB80C




# Surface与BufferQueue：跨进程数据传递



**核心角色**：`Surface`（应用端绘图接口）、`BufferQueue`（生产者-消费者模型）、`GraphicBuffer`（图形缓冲区）。

**流程与作用**：

- **Surface创建**：`ViewRootImpl`通过`WindowManager`向`WMS`申请`Surface`，`WMS`通过`SurfaceFlinger`创建`SurfaceControl`（控制句柄），再通过`Surface.copyFrom()`关联到应用端`Surface`。
    
- **BufferQueue机制**：`Surface`对应一个`BufferQueue`（最多31个缓冲区，双缓冲/三缓冲），采用**生产者-消费者模型**：
    
    - 生产者（应用）：通过`Surface.lockCanvas()`获取缓冲区，绘制后将`Canvas`内容提交（`unlockCanvasAndPost()`），即`queueBuffer()`。
        
    - 消费者（SurfaceFlinger）：从`BufferQueue`中`acquireBuffer()`获取最新缓冲区，用于合成。
        
    
- **数据共享**：通过**匿名共享内存（Ashmem）**实现跨进程数据传递，避免拷贝开销。



BLASTBufferQueue 即通过将 onFrameAvailable 进行拦截，将 SurfaceFlinger 的 Buffer 操作变为一次 Transaction 操作，以节省性能开销。



# refs



[官方文档 Android Graphics Architecture ](https://source.android.com/docs/core/graphics)

[掘金博客](https://juejin.cn/post/7345679865185075234#heading-0)
