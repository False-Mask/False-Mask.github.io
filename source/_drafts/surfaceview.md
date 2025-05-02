---
title: surfaceview
tags:
cover:
---



# SurfaceView原理分析





- format
- visble
- alpha
- creating
- size
- windowVisibility
- positionChange
- layout
- hint
- relativeZ
- surfaceLifecycleChanged
- hdrHeadroom



## SurfaceSyncer 



通过将状态存入队列，通过Chrograpger调度执行，合并后一起提交



Transition.apply是核心方法

