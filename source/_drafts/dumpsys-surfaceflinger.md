---
title: dumpsys-surfaceflinger
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





