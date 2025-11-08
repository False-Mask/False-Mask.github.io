---
title: aosp-wms
tags:
cover:
---





# WMS逻辑解析





# 调用堆栈



- 初始化

初始化ViewRootImpl & WindowSession的关联关系 

Activity -> PhoneWindow -> WindowManagerImpl -> WindowManagerGlobal



- 相互通信

ViewRootImpl -> WindowSession -> WMS

WMS -> IWindow -> ViewRootImpl





# refs



[掘金专栏-WMS](https://juejin.cn/column/7339827208086929444)

[【Android 13源码分析】WindowContainer窗口层级-1-初识窗口层级树](https://juejin.cn/post/7339827208086896676)

[【Android 源码分析】Activity短暂的一生 -- 目录篇 （持续更新）](https://juejin.cn/post/7345105816242552871)
