---
title: tmp
tags:
cover:
---

# 零散记录





# MediaPlayer



MediaPlayer是C/S架构的

Client -> MediaPlayer

Server -> MediaPlayerServer





# 关于UI绘制的逻辑





Producer & Consumer



Producer

queueBuffer



consumer

dequeBuffer





# NavigationBar





1.NavigationBar的逻辑运行在SystemUI进程中

2.NavigationBar的初始化逻辑

- NavigationBar是一个View
- NavigationBar的渲染逻辑是通过WMS.addView创建了单独的Window对象，

TODO NavigationBar初始化逻辑

1. 调用onInit初始化View
2. onInit中初始化View相关变量& window对象并添加到View中。

TODO 类关系

SystemUIService.onCreate 

-> SystemUIApplication 

-> CentralSurfacesImpl 

-> NavigationBarControllerImpl 

-> NavigationBarComponent 

-> NavigationBar

->NavigationBarView

``` shell
onInit: dalvik.system.VMStack.getThreadStackTrace(Native Method)
                                                                                                    java.lang.Thread.getStackTrace(Thread.java:1841)
                                                                                                    com.android.systemui.navigationbar.NavigationBar.onInit(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:10)
                                                                                                    com.android.systemui.util.ViewController.init$10(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:6)
                                                                                                    com.android.systemui.navigationbar.NavigationBarControllerImpl.createNavigationBar(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:53)
                                                                                                    com.android.systemui.statusbar.phone.CentralSurfacesImpl.start(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:715)
                                                                                                    com.android.systemui.SystemUIApplication.startStartable(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:26)
                                                                                                    com.android.systemui.SystemUIApplication$$ExternalSyntheticLambda0.run(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:56)
                                                                                                    com.android.systemui.SystemUIApplication.timeInitialization(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:28)
                                                                                                    com.android.systemui.SystemUIApplication.startServicesIfNeeded(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:190)
                                                                                                    com.android.systemui.SystemUIApplication.startSystemUserServicesIfNeeded(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:64)
                                                                                                    com.android.systemui.SystemUIService.onCreate(go/retraceme 5226c91cf9b2549c1e80b26c6032c0962958e131bdfbfb6369c5b55ecfd52a07:10)
                                                                                                    android.app.ActivityThread.handleCreateService(ActivityThread.java:4912)
                                                                                                    android.app.ActivityThread.-$$Nest$mhandleCreateService(Unknown Source:0)
                                                                                                    android.app.ActivityThread$H.handleMessage(ActivityThread.java:2407)
                                                                                                    android.os.Handler.dispatchMessage(Handler.java:107)
                                                                                                    android.os.Looper.loopOnce(Looper.java:232)
                                                                                                    android.os.Looper.loop(Looper.java:317)
                                                                                                    android.app.ActivityThread.main(ActivityThread.java:8592)
                                                                                                    java.lang.reflect.Method.invoke(Native Method)
                                                                                                    com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:580)
                                                                                                    com.android.internal.os.ZygoteInit.main(ZygoteInit.java:878)
```



猜想：

用户看到的画面是如何合成的？

多个Window叠加的效果 -> 每个Window是由单独的App渲染 -> 最终通过WMS进行管理 -> SurfaceFlinger管理Layers -> HWC合成 -> HAL合成
