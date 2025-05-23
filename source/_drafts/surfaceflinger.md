---
title: SurfaceFlinger合成分析
tags:
cover:
---



# SurfaceFlinger合成分析





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
