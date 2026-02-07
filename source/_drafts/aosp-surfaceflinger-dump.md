---
title: aosp-surfaceflinger-dump
tags:
cover:
---

# dumpsys SurfaceFlinger分析



1. Layer是有父子结构的，这个父子结构有什么用





# 入口位置



frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp

```c++
void SurfaceFlinger::dumpAll(const DumpArgs& args, const std::string& compositionLayers,
                             std::string& result) const {
    // .......
    /*
     * Dump library configuration.
     */
    
    /*
     * Dump the visible layer list
     */
    
    /*
     * Dump SurfaceFlinger global state
     */
    
    /*
     * Dump HWComposer state
     */
    
    /*
     * Dump gralloc state
     */
    
    /*
     * Dump flag/property manager state
     */
}
```





# 过程分析





## Dump library configuration



> 看样子是一些静态的字段信息。



``` c++
void SurfaceFlinger::dumpAll(const DumpArgs& args, const std::string& compositionLayers,
                             std::string& result) const {
    // .......
    
    /*
     * Dump library configuration.
     */
    colorizer.bold(result); // 插入转义字符对后续字符进行加粗
    result.append("Build configuration:");
    colorizer.reset(result); // 恢复字符样式
    appendSfConfigString(result);
    result.append("\n");

    result.append("\nDisplay identification data:\n");
    dumpDisplayIdentificationData(result);

    result.append("\nWide-Color information:\n");
    dumpWideColorInfo(result);

    dumpHdrInfo(result);

    colorizer.bold(result);
    result.append("Sync configuration: ");
    colorizer.reset(result);
    result.append(SyncFeatures::getInstance().toString());
    result.append("\n\n");

    colorizer.bold(result);
    result.append("Scheduler:\n");
    colorizer.reset(result);
    dumpScheduler(result);
    dumpEvents(result);
    dumpVsync(result);
    result.append("\n");
    
    //.......
}
```



> Build configuration
>
> SurfaceFlinger信息

``` plaintText
Build configuration: [sf PRESENT_TIME_OFFSET=0 FORCE_HWC_FOR_RBG_TO_YUV=0 MAX_VIRT_DISPLAY_DIM=0 RUNNING_WITHOUT_SYNC_FRAMEWORK=0 NUM_FRAMEBUFFER_SURFACE_BUFFERS=3]

1. PRESENT_TIME_OFFSET -> 用于调整 SurfaceFlinger 提交帧到硬件的时机,补偿显示流水线中的延迟（如硬件处理、信号传输等）
2. FORCE_HWC_FOR_RBG_TO_YUV -> 是否强制使用 HWC（硬件合成器）处理 RGB 到 YUV 的颜色空间转换。
3. MAX_VIRT_DISPLAY_DIM ->  硬件支持的虚拟显示最大尺寸
4. RUNNING_WITHOUT_SYNC_FRAMEWORK -> 系统是否在没有同步框架（如 Android Sync Framework）的情况下运行
5. NUM_FRAMEBUFFER_SURFACE_BUFFERS -> 帧缓冲表面（FramebufferSurface）允许同时获取的缓冲区数量

```



> Display identification data
>
> 输出显示器信息





> Wide-Color information
>
> 输出位宽信息



> Sync configuration
>
> 输出画面同步机制



> Scheduler
>
> TODO ?





## Dump the visible layer list





``` c++
void SurfaceFlinger::dumpAll(const DumpArgs& args, const std::string& compositionLayers,
                             std::string& result) const {
    // .......
    
    /*
     * Dump library configuration.
     */
    colorizer.bold(result);
    StringAppendF(&result, "SurfaceFlinger New Frontend Enabled:%s\n",
                  mLayerLifecycleManagerEnabled ? "true" : "false");
    StringAppendF(&result, "Active Layers - layers with client handles (count = %zu)\n",
                  mNumLayers.load());
    colorizer.reset(result);

    result.append(compositionLayers);

    colorizer.bold(result);
    StringAppendF(&result, "Displays (%zu entries)\n", mDisplays.size());
    colorizer.reset(result);
    dumpDisplays(result);
    dumpCompositionDisplays(result);
    result.push_back('\n');

    mCompositionEngine->dump(result);
    
    //.......
}
```











# 记录





https://juejin.cn/post/6856293249624375304#heading-7

1.事件传递顺序：EventThread -> MessageQueue -> SF  

2.EventThread 对应 SurfaceFlinger::mSFEventThread \ MessageQueue对应 SurfaceFlinger::mEventQueue

3.EventThread和MessageQueue关系建立

``` c++
void SurfaceFlinger::init() {
    ...
    sp<VSyncSource> sfVsyncSrc = new DispSyncSource(&mPrimaryDispSync,
            sfVsyncPhaseOffsetNs, false);//创建合成延时Vsync源
    mSFEventThread = new EventThread(sfVsyncSrc);
    mEventQueue.setEventThread(mSFEventThread);//将合成延时vysnc源与SF的事件管理关联，因为SF主要负责合成
    ...
}
```

4.EventThread通过Connection向MessageQueue中的Looper发送信息，并回调同时MeesageQueue

Connection -> MessageQueue::cb_eventReceiver -> MessageQueue::eventReceiver -> mHandler->dispatchInvalidate/mHandler->dispatchRefresh

-> MessageQueue::Handler::handleMessage

5.SF的onMessageReceived方法中分别处理TRANSACTION，INVALIDATE以及REFRESH事件，

TRANSACTION事件主要是处理Layer和Display的属性变更

INVALIDATE事件中除了处理TRANSACTION事件的内容外还需要获取合成图层layer的最新帧数据，同时还要根据内容更新脏区域

REFRESH事件中主要是合并和渲染输出的处理。实际上我们可以看到，在INVALIDATE事件中包含了TRANSACTION和REFRESH事件的内容，它会完整的处理一次合并和渲染输出过程。



6.触发合成的时机

UI数据准备完成 -> 调用Layer::onFrameAvailable -> 回掉SF::signalLayerUpdate() -> 请求一次VSYNC信号

-> MessageQueue监听 -> SF

7.合成过程

- preComposiotion：

调用每一个layer的onPreComposioin方法判断其是否还有未处理的frame,如果有则进行额外的合成和渲染操作

- rebuildLayerStacks：

收集设备可见的layer集合，计算每个layer的可见区域和脏区域

- setUpHWComposer

创建workList，为所有可见的layer设置frameData, 调用hwc prepare确认合成方式

通过setFramedata将Layer的latchBuffer获取了layer最新的帧数据设置到layer中

通过HWC prepare确认合成方式为HWC还是OPENGL,并打上标记。

- doComposition

使用EGL/HWC合成对于的Layer到GraphicBuffer

- postFrameBuffer

将使用EGL合成的Layer以及需要HWC合成的Layer统一提交到HWC中进行合成
