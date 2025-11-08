---
title: aosp-window-tree
tags:
cover:
---



# Android Window Tree 结构







# 基础使用



``` shell
adb shell dumpsys window w 
```





# 层级分析



```
adb shell dumpsys activity containers
ACTIVITY MANAGER CONTAINERS (dumpsys activity containers)
└─ ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
   └─ Display 0 name="Built-in Screen" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1080,2400] bounds=[0,0][1080,2400]
      ├─ Leaf:36:36 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │  ├─ WindowToken{812fa3c type=2024 android.os.BinderProxy@505742f} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │  │  └─ ebb03c5 ScreenDecorOverlayBottom type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │  └─ WindowToken{aa9ee5b type=2024 android.os.BinderProxy@913396a} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │     └─ 88797d1 ScreenDecorOverlay type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      ├─ HideDisplayCutout:32:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │  ├─ OneHanded:34:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │  │  └─ FullscreenMagnification:34:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │  │     └─ Leaf:34:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │  ├─ FullscreenMagnification:33:33 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │  │  └─ Leaf:33:33 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │  └─ OneHanded:32:32 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      │     └─ Leaf:32:32 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
      └─ WindowedMagnification:0:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         ├─ HideDisplayCutout:26:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │  └─ OneHanded:26:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │     ├─ FullscreenMagnification:29:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │     │  └─ Leaf:29:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │     ├─ Leaf:28:28 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │     └─ FullscreenMagnification:26:27 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │        └─ Leaf:26:27 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         ├─ Leaf:24:25 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │  └─ WindowToken{a3502dd type=2019 android.os.BinderProxy@f07a787} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │     └─ 35fbe52 NavigationBar0 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         ├─ HideDisplayCutout:18:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │  └─ OneHanded:18:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │     └─ FullscreenMagnification:18:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │        └─ Leaf:18:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         ├─ OneHanded:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │  └─ FullscreenMagnification:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │     └─ Leaf:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │        └─ WindowToken{1ee12fe type=2040 android.os.BinderProxy@b358880} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │           └─ a9baa5f NotificationShade type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         ├─ HideDisplayCutout:16:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │  └─ OneHanded:16:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │     └─ FullscreenMagnification:16:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │        └─ Leaf:16:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         ├─ OneHanded:15:15 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │  └─ FullscreenMagnification:15:15 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │     └─ Leaf:15:15 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │        └─ WindowToken{1121bf1 type=2000 android.os.BinderProxy@47a8d7b} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         │           └─ 69f83d6 StatusBar type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         └─ HideDisplayCutout:0:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
            └─ OneHanded:0:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
               ├─ ImePlaceholder:13:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
               │  └─ ImeContainer type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
               │     └─ WindowToken{b33ce48 type=2011 android.os.Binder@84c20eb} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
               │        └─ 1843e2f InputMethod type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
               └─ FullscreenMagnification:0:12 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  ├─ Leaf:3:12 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  └─ WindowToken{39067e3 type=2038 android.os.BinderProxy@678b80a} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │     └─ 1d9966b ShellDropTarget type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  ├─ DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  ├─ Task=1 type=HOME mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │  └─ Task=6 type=HOME mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │     └─ ActivityRecord{1fd6bb6 u0 com.android.launcher3/.uioverrides.QuickstepLauncher t6} type=HOME mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │        └─ 8e31f76 com.android.launcher3/com.android.launcher3.uioverrides.QuickstepLauncher type=HOME mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  ├─ Task=180 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │  └─ ActivityRecord{65d533d u0 com.microsoft.emmx/com.microsoft.ruby.Main t180} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │     └─ 9f7f1c com.microsoft.emmx/com.microsoft.ruby.Main type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  ├─ Task=2 type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │  ├─ Task=4 type=undefined mode=MULTI-WINDOW override-mode=MULTI-WINDOW requested-bounds=[0,0][1080,1187] bounds=[0,0][1080,1187]
                  │  │  └─ Task=3 type=undefined mode=MULTI-WINDOW override-mode=MULTI-WINDOW requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  ├─ Task=177 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │  ├─ ActivityRecord{10c7d4c u0 com.android.permissioncontroller/.permission.ui.GrantPermissionsActivity t177} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │  │  └─ c70996a com.android.permissioncontroller/com.android.permissioncontroller.permission.ui.GrantPermissionsActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │  └─ ActivityRecord{3b63045 u0 com.google.android.googlequicksearchbox/.VoiceSearchActivity t177} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  ├─ Task=175 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │  ├─ ActivityRecord{15b8025 u0 com.android.frameworks.core.batterystatsviewer/.BatteryStatsViewerActivity t175} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │  │  └─ b971913 com.android.frameworks.core.batterystatsviewer/com.android.frameworks.core.batterystatsviewer.BatteryStatsViewerActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │  └─ ActivityRecord{6adb948 u0 com.android.frameworks.core.batterystatsviewer/.BatteryConsumerPickerActivity t175} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  │     └─ d7e7620 com.android.frameworks.core.batterystatsviewer/com.android.frameworks.core.batterystatsviewer.BatteryConsumerPickerActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │  └─ Task=178 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  │     └─ ActivityRecord{6970adf u0 com.google.android.youtube/.app.honeycomb.Shell$HomeActivity t178} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                  └─ Leaf:0:1 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                     └─ WallpaperWindowToken{a2c96b9 token=android.os.Binder@81bf080} type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
                        └─ 3cddfae com.android.systemui.wallpapers.ImageWallpaper type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
 
```



## RootWindowContainer

Winodw窗口属的根节点，是WindowContainer的直接子类

```java
class RootWindowContainer extends WindowContainer<DisplayContent>
        implements DisplayManager.DisplayListener {
```





## DisplayContent

根节点下的唯一子节点,  一个DisplayContent对象象征着一块屏幕

```java
/**
 * Utility class for keeping track of the WindowStates and other pertinent contents of a
 * particular Display.
 */
 class DisplayContent extends RootDisplayArea implements WindowManagerPolicy.DisplayContentInfo {
```


子节点配置

``` java
public abstract class DisplayAreaPolicy { 

    private void configureTrustedHierarchyBuilder(HierarchyBuilder rootHierarchy,
                    WindowManagerService wmService, DisplayContent content) {
                // WindowedMagnification should be on the top so that there is only one surface
                // to be magnified.
                rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                        FEATURE_WINDOWED_MAGNIFICATION)
                        .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                        .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                        // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                        .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                        .build());
                if (content.isDefaultDisplay) {
                    // Only default display can have cutout.
                    // See LocalDisplayAdapter.LocalDisplayDevice#getDisplayDeviceInfoLocked.
                    rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "HideDisplayCutout",
                            FEATURE_HIDE_DISPLAY_CUTOUT)
                            .all()
                            .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL, TYPE_STATUS_BAR,
                                    TYPE_NOTIFICATION_SHADE)
                            .build())
                            .addFeature(new Feature.Builder(wmService.mPolicy, "OneHanded",
                                    FEATURE_ONE_HANDED)
                                    .all()
                                    .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL,
                                            TYPE_SECURE_SYSTEM_OVERLAY)
                                    .build());
                }
                rootHierarchy
                        .addFeature(new Feature.Builder(wmService.mPolicy, "FullscreenMagnification",
                                FEATURE_FULLSCREEN_MAGNIFICATION)
                                .all()
                                .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY, TYPE_INPUT_METHOD,
                                        TYPE_INPUT_METHOD_DIALOG, TYPE_MAGNIFICATION_OVERLAY,
                                        TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                                .build())
                        .addFeature(new Feature.Builder(wmService.mPolicy, "ImePlaceholder",
                                FEATURE_IME_PLACEHOLDER)
                                .and(TYPE_INPUT_METHOD, TYPE_INPUT_METHOD_DIALOG)
                                .build());
            }
        }

}
```

- WindowedMagnification
- HideDisplayCutout
- 

## WindowedMagnification


## HideDisplayCutout


## DisplayArea.Tokens









## WindowToken





# Features



Feature是DisplayContent为了方便管理容器抽象出来的一层概念





## Features创建

调用堆栈

```plaintText
at com.android.server.wm.DisplayAreaPolicy$DefaultProvider.configureTrustedHierarchyBuilder(DisplayAreaPolicy.java:126)
                                                                                                                                at com.android.server.wm.DisplayAreaPolicy$DefaultProvider.instantiate(DisplayAreaPolicy.java:116)
                                                                                                                                at com.android.server.wm.DisplayContent.configureSurfaces(DisplayContent.java:1345)
                                                                                                                                at com.android.server.wm.DisplayContent.<init>(DisplayContent.java:1249)
                                                                                                                                at com.android.server.wm.RootWindowContainer.setWindowManager(RootWindowContainer.java:1272)
                                                                                                                                at com.android.server.wm.ActivityTaskManagerService.setWindowManager(ActivityTaskManagerService.java:1061)
                                                                                                                                at com.android.server.am.ActivityManagerService.setWindowManager(ActivityManagerService.java:2149)
                                                                                                                                at com.android.server.SystemServer.startOtherServices(SystemServer.java:1673)
                                                                                                                                at com.android.server.SystemServer.run(SystemServer.java:977)
                                                                                                                                at com.android.server.SystemServer.main(SystemServer.java:698)
                                                                                                                                at java.lang.reflect.Method.invoke(Native Method)
                                                                                                                                at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:580)
                                                                                                                                at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:856)
```

在此过程中会通过Feature.Builder创建5个Feature并通过addFeature添加到rootHierarchy中

```java
static final class DefaultProvider implements DisplayAreaPolicy.Provider {
    // ......
    
    private void configureTrustedHierarchyBuilder(HierarchyBuilder rootHierarchy,
                    WindowManagerService wmService, DisplayContent content) {
                // WindowedMagnification should be on the top so that there is only one surface
                // to be magnified.
                rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                        FEATURE_WINDOWED_MAGNIFICATION)
                        .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                        .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                        // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                        .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                        .build());
                if (content.isDefaultDisplay) {
                    // Only default display can have cutout.
                    // See LocalDisplayAdapter.LocalDisplayDevice#getDisplayDeviceInfoLocked.
                    rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "HideDisplayCutout",
                            FEATURE_HIDE_DISPLAY_CUTOUT)
                            .all()
                            .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL, TYPE_STATUS_BAR,
                                    TYPE_NOTIFICATION_SHADE)
                            .build())
                            .addFeature(new Feature.Builder(wmService.mPolicy, "OneHanded",
                                    FEATURE_ONE_HANDED)
                                    .all()
                                    .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL,
                                            TYPE_SECURE_SYSTEM_OVERLAY)
                                    .build());
                }
                rootHierarchy
                        .addFeature(new Feature.Builder(wmService.mPolicy, "FullscreenMagnification",
                                FEATURE_FULLSCREEN_MAGNIFICATION)
                                .all()
                                .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY, TYPE_INPUT_METHOD,
                                        TYPE_INPUT_METHOD_DIALOG, TYPE_MAGNIFICATION_OVERLAY,
                                        TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                                .build())
                        .addFeature(new Feature.Builder(wmService.mPolicy, "ImePlaceholder",
                                FEATURE_IME_PLACEHOLDER)
                                .and(TYPE_INPUT_METHOD, TYPE_INPUT_METHOD_DIALOG)
                                .build());
            }
        }

	// .....

}
```

如下是5个Feature的构建逻辑，在构建这些Feature以后会通过HierarchyBuilder.addFeature添加保存起来。下面对Feature的构建逻辑进行分析

```java
new  (wmService.mPolicy, "WindowedMagnification",
                        FEATURE_WINDOWED_MAGNIFICATION)
                        .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                        .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                        // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                        .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                        .build();

new Feature.Builder(wmService.mPolicy, "HideDisplayCutout",
                            FEATURE_HIDE_DISPLAY_CUTOUT)
                            .all()
                            .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL, TYPE_STATUS_BAR,
                                    TYPE_NOTIFICATION_SHADE)
                            .build();

new Feature.Builder(wmService.mPolicy, "OneHanded",
                                    FEATURE_ONE_HANDED)
                                    .all()
                                    .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL,
                                            TYPE_SECURE_SYSTEM_OVERLAY)
                                    .build();

new Feature.Builder(wmService.mPolicy, "FullscreenMagnification",
                                FEATURE_FULLSCREEN_MAGNIFICATION)
                                .all()
                                .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY, TYPE_INPUT_METHOD,
                                        TYPE_INPUT_METHOD_DIALOG, TYPE_MAGNIFICATION_OVERLAY,
                                        TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                                .build();

new Feature.Builder(wmService.mPolicy, "ImePlaceholder",
                                FEATURE_IME_PLACEHOLDER)
                                .and(TYPE_INPUT_METHOD, TYPE_INPUT_METHOD_DIALOG)
                                .build();
```



- 构造函数

```java
static class Builder {
	
    // ......
    
    Builder(WindowManagerPolicy policy, String name, int id) {
        mPolicy = policy;
        mName = name;
        // id取值
        // [FEATURE_SYSTEM_FIRST + 3, FEATURE_SYSTEM_FIRST + 7]
        mId = id;
        // 默认是37个
        mLayers = new boolean[mPolicy.getMaxWindowLayer() + 1];
    }
	
    // ......

}
```

- 常用构建方法

``` java
static class Builder {

    /**
     * Set that the feature applies to the given window types.
     */
    Builder and(int... types) {
        for (int i = 0; i < types.length; i++) {
            int type = types[i];
            set(type, true);
        }
        return this;
    }
    
    /**
     * Set that the feature applies to all window types.
     */
    // 将layers所有元素设置为true
    Builder all() {
        Arrays.fill(mLayers, true);
        return this;
    }
    
    /**
     * Set that the feature does not apply to the given window types.
     */
    // 将指定type层级的的layer设置为false
    Builder except(int... types) {
        for (int i = 0; i < types.length; i++) {
            int type = types[i];
            set(type, false);
        }
        return this;
    }
    
    /**
     * Set that the feature applies window types that are layerd at or below the layer of
     * the given window type.
     */
    // 将[0,max-1]的层级layer设置为true
    Builder upTo(int typeInclusive) {
        // max计算可见下方方法，其实很简单，就是做了一层映射，有一个很大的switch分支，对特定类型返回固定的值
        final int max = layerFromType(typeInclusive, false);
        for (int i = 0; i < max; i++) {
            mLayers[i] = true;
        }
        // 将typeInclusive对应的方法设置为true
        // 额外设置一些type
        set(typeInclusive, true);
        return this;
    }
    
    private void set(int type, boolean value) {
        mLayers[layerFromType(type, true)] = value;
        // 如果是TYPE_APPLICATION_OVERLAY，还会额外设置如下方法
        // 转化一下就是：
        // mLayers[11] = true
        // mLayers[9 ] = false
        // mLayers[10] = false
        // mLayers[9 ] = false
        if (type == TYPE_APPLICATION_OVERLAY) {
            mLayers[layerFromType(type, true)] = value;
            mLayers[layerFromType(TYPE_SYSTEM_ALERT, false)] = value;
            mLayers[layerFromType(TYPE_SYSTEM_OVERLAY, false)] = value;
            mLayers[layerFromType(TYPE_SYSTEM_ERROR, false)] = value;
        }
    }
    
    Feature build() {
        // 默认情况下都是true
        if (mExcludeRoundedCorner) {
            // 最后一层需要留给rounded corner layer因此实际可分配的layer为[0,36]
            // Always put the rounded corner layer to the top most layer.
            mLayers[mPolicy.getMaxWindowLayer()] = false;
        }
        return new Feature(mName, mId, mLayers.clone(), mNewDisplayAreaSupplier);
    }
    
}
```

- 最终Feature层级关系

```java
// [0,31]
WindowedMagnification
// [0,14] 16 [18,23] [26,35]
HideDisplayCutout
// [0,23],[26,32],[34,35]
OneHanded
// [0,12],[15,23],[26,27],[29,31],[33,35]    
FullscreenMagnification
// [13,14]
ImePlaceholder
```







## DisplayAreaPolicy初始化

instantiate主要是用来创建DisplayArea tree，执行完成后将构建从RootDisplayArea作为根节点的树形结构

1.创建TaskDisplayArea添加到tdaList

2.创建RootHierarchy 设置ImeContainer和TaskDisplayAreas

3.添加配置Features

4.创建DisplayAreaPolicy

```java
static final class DefaultProvider implements DisplayAreaPolicy.Provider {
    // ......
    
    public DisplayAreaPolicy instantiate(WindowManagerService wmService,
            DisplayContent content, RootDisplayArea root,
            DisplayArea.Tokens imeContainer) {
        
        // 1.创建TaskDisplayArea
        final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
                "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);
        final List<TaskDisplayArea> tdaList = new ArrayList<>();
        tdaList.add(defaultTaskDisplayArea);

        // Define the features that will be supported under the root of the whole logical
        // display. The policy will build the DisplayArea hierarchy based on this.
        final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);
        // Set the essential containers (even if the display doesn't support IME).
        // 2.创建RootHierarchy 设置ImeContainer和TaskDisplayAreas
        rootHierarchy.setImeContainer(imeContainer).setTaskDisplayAreas(tdaList);
        // 3. 添加配置Features
        if (content.isTrusted()) {
            // Only trusted display can have system decorations.
            // 配置
            // WindowedMagnification, HideDisplayCutout, OneHanded, FullscreenMagnification, ImePlaceholder
            configureTrustedHierarchyBuilder(rootHierarchy, wmService, content);
        }

        // Instantiate the policy with the hierarchy defined above. This will create and attach
        // all the necessary DisplayAreas to the root.
        return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);
    }
    
    // ......
        
}
```

＃2设置ImeContainer & TaskDisplayArea需要注意下。后面会用到

```java
// DefaultProvider.java
@Override
public DisplayAreaPolicy instantiate(WindowManagerService wmService,
        DisplayContent content, RootDisplayArea root,
        DisplayArea.Tokens imeContainer) {
    
    // 1. 创建TaskDisplayArea
    final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
            "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);
    final List<TaskDisplayArea> tdaList = new ArrayList<>();
    tdaList.add(defaultTaskDisplayArea);

    final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);
    // 2. 设置ImeContainer，设置TaskDisplayAreas
    rootHierarchy.setImeContainer(imeContainer).setTaskDisplayAreas(tdaList);
    // ......

    // Instantiate the policy with the hierarchy defined above. This will create and attach
    // all the necessary DisplayAreas to the root.
    return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);
}
```





### DisplayAreaPolicyBuilder.build

```java
class DisplayAreaPolicyBuilder {
	
    // ......
    Result build(WindowManagerService wmService) {
        // 1.输入参数验证
        validate();

        // Attach DA group roots to screen hierarchy before adding windows to group hierarchies.
        // 2. 将rootHierachy中的元素进行遍历attach到DisplayAreaGroup中去
        mRootHierarchyBuilder.build(mDisplayAreaGroupHierarchyBuilders);
        List<RootDisplayArea> displayAreaGroupRoots = new ArrayList<>(
                mDisplayAreaGroupHierarchyBuilders.size());
        // 3. 遍历displayAreaGroup元素，逐一进行构建。
        for (int i = 0; i < mDisplayAreaGroupHierarchyBuilders.size(); i++) {
            HierarchyBuilder hierarchyBuilder = mDisplayAreaGroupHierarchyBuilders.get(i);
            hierarchyBuilder.build();
            displayAreaGroupRoots.add(hierarchyBuilder.mRoot);
        }
        // Use the default function if it is not specified otherwise.
        if (mSelectRootForWindowFunc == null) {
            mSelectRootForWindowFunc = new DefaultSelectRootForWindowFunction(
                    mRootHierarchyBuilder.mRoot, displayAreaGroupRoots);
        }
        return new Result(wmService, mRootHierarchyBuilder.mRoot, displayAreaGroupRoots,
                mSelectRootForWindowFunc, mSelectTaskDisplayAreaFunc);
    }
    // ......
    
}
```



### HierarchyBuilder.build

执行过程的逻辑是什么？

阶段：

1.初始化参数

2.构建PendingArea树节点

3.添加子节点

4.构建DisplayArea树节点

``` java
static class HierarchyBuilder {

    private void build(@Nullable List<HierarchyBuilder> displayAreaGroupHierarchyBuilders) {
                final WindowManagerPolicy policy = mRoot.mWmService.mPolicy;
                final int maxWindowLayerCount = policy.getMaxWindowLayer() + 1;
                final DisplayArea.Tokens[] displayAreaForLayer =
                        new DisplayArea.Tokens[maxWindowLayerCount];
        		// 1. 初始化arrayMap, key为feature,value为刚初始化的ArrayList
                final Map<Feature, List<DisplayArea<WindowContainer>>> featureAreas =
                        new ArrayMap<>(mFeatures.size());
                for (int i = 0; i < mFeatures.size(); i++) {
                    featureAreas.put(mFeatures.get(i), new ArrayList<>());
                }
			
        		// 这个方法构建了layer 层级，在构建的过程中需要如下的参数信息
        		// 1. 一个map, key为feature value为set,其实key的取值范围为所有的feature
        		// 2. 
                // This method constructs the layer hierarchy with the following properties:
                // (1) Every feature maps to a set of DisplayAreas
                // (2) After adding a window, for every feature the window's type belongs to,
                //     it is a descendant of one of the corresponding DisplayAreas of the feature.
                // (3) Z-order is maintained, i.e. if z-range(area) denotes the set of layers of windows
                //     within a DisplayArea:
                //      for every pair of DisplayArea siblings (a,b), where a is below b, it holds that
                //      max(z-range(a)) <= min(z-range(b))
                //
                // The algorithm below iteratively creates such a hierarchy:
                //  - Initially, all windows are attached to the root.
                //  - For each feature we create a set of DisplayAreas, by looping over the layers
                //    - if the feature does apply to the current layer, we need to find a DisplayArea
                //      for it to satisfy (2)
                //      - we can re-use the previous layer's area if:
                //         the current feature also applies to the previous layer, (to satisfy (3))
                //         and the last feature that applied to the previous layer is the same as
                //           the last feature that applied to the current layer (to satisfy (2))
                //      - otherwise we create a new DisplayArea below the last feature that applied
                //        to the current layer
				// 2. 初始化中间变量
                PendingArea[] areaForLayer = new PendingArea[maxWindowLayerCount];
                final PendingArea root = new PendingArea(null, 0, null);
                Arrays.fill(areaForLayer, root);

                // Create DisplayAreas to cover all defined features.
        		// 使用已添加的所有的features创建DisplayAreas树形节点 
                final int size = mFeatures.size();
                for (int i = 0; i < size; i++) {
                    // Traverse the features with the order they are defined, so that the early defined
                    // feature will be on the top in the hierarchy.
                    final Feature feature = mFeatures.get(i);
                    PendingArea featureArea = null;
                    for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                        if (feature.mWindowLayers[layer]) {
                            // This feature will be applied to this window layer.
                            //
                            // We need to find a DisplayArea for it:
                            // We can reuse the existing one if it was created for this feature for the
                            // previous layer AND the last feature that applied to the previous layer is
                            // the same as the feature that applied to the current layer (so they are ok
                            // to share the same parent DisplayArea).
                            if (featureArea == null || featureArea.mParent != areaForLayer[layer]) {
                                // No suitable DisplayArea:
                                // Create a new one under the previous area (as parent) for this layer.
                                featureArea = new PendingArea(feature, layer, areaForLayer[layer]);
                                areaForLayer[layer].mChildren.add(featureArea);
                            }
                            areaForLayer[layer] = featureArea;
                        } else {
                            // This feature won't be applied to this window layer. If it needs to be
                            // applied to the next layer, we will need to create a new DisplayArea for
                            // that.
                            featureArea = null;
                        }
                    }
                }

                // Create Tokens as leaf for every layer.
                PendingArea leafArea = null;
                int leafType = LEAF_TYPE_TOKENS;
                for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                    int type = typeOfLayer(policy, layer);
                    // Check whether we can reuse the same Tokens with the previous layer. This happens
                    // if the previous layer is the same type as the current layer AND there is no
                    // feature that applies to only one of them.
                    if (leafArea == null || leafArea.mParent != areaForLayer[layer]
                            || type != leafType) {
                        // Create a new Tokens for this layer.
                        leafArea = new PendingArea(null /* feature */, layer, areaForLayer[layer]);
                        areaForLayer[layer].mChildren.add(leafArea);
                        leafType = type;
                        if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
                            // We use the passed in TaskDisplayAreas for task container type of layer.
                            // Skip creating Tokens even if there is no TDA.
                            addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
                            addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                                    displayAreaGroupHierarchyBuilders);
                            leafArea.mSkipTokens = true;
                        } else if (leafType == LEAF_TYPE_IME_CONTAINERS) {
                            // We use the passed in ImeContainer for ime container type of layer.
                            // Skip creating Tokens even if there is no ime container.
                            leafArea.mExisting = mImeContainer;
                            leafArea.mSkipTokens = true;
                        }
                    }
                    leafArea.mMaxLayer = layer;
                }
                root.computeMaxLayer();

                // We built a tree of PendingAreas above with all the necessary info to represent the
                // hierarchy, now create and attach real DisplayAreas to the root.
                root.instantiateChildren(mRoot, displayAreaForLayer, 0, featureAreas);

                // Notify the root that we have finished attaching all the DisplayAreas. Cache all the
                // feature related collections there for fast access.
                mRoot.onHierarchyBuilt(mFeatures, displayAreaForLayer, featureAreas);
        }
        
}
```



#### 初始化

```java
// 获取wmsService
final WindowManagerPolicy policy = mRoot.mWmService.mPolicy;
// 获取最大的layer数目，数值为37
final int maxWindowLayerCount = policy.getMaxWindowLayer() + 1;
// 创建DisplayArea.Tokens数组
final DisplayArea.Tokens[] displayAreaForLayer =
        new DisplayArea.Tokens[maxWindowLayerCount];
// 初始化arrayMap, key为feature,value为刚初始化的ArrayList
final Map<Feature, List<DisplayArea<WindowContainer>>> featureAreas =
        new ArrayMap<>(mFeatures.size());
for (int i = 0; i < mFeatures.size(); i++) {
    featureAreas.put(mFeatures.get(i), new ArrayList<>());
}
```





#### 构建PendingArea树节点

这个方法将会使用如下的特性构建层级节点

1.一个map存储了每个feature到一组DisplayArea的映射

2.在添加窗口以后，对于该窗口所归属的每个feature而言，窗口都是feature所对应的DisplayArea的子层级节点。

3.z轴排序，即若z-range(area)表示显示区域内的窗口图层集合：对于每一对显示区域兄弟节点(a、b)，当a位于b下方时，必须满足 max(z-range(a)) <= min(z-range(b))



下方的算法该将迭代计算生成一个层级。

- a.初始化所有的window都会attach到root上
- b.对于每个feature我们通过遍历所有的layer,创建displayArea的集合，
  - i.如果feature不归属于当前的layer层级，我们需要寻找一个DisplayArea用以满足上述条件2
    - 1.我们可以复用上一个层级的区域如果 当前特征也适用于前一图层（满足条件3的Z轴顺序），前一图层最后应用的特征与当前图层最后应用的特征相同（满足条件2）
    - 2.否则需在当前图层最后应用特征的下方创建新显示区域

```java
// 初始化所有Layer均指向根节点
PendingArea[] areaForLayer = new PendingArea[maxWindowLayerCount];
final PendingArea root = new PendingArea(null, 0, null);
Arrays.fill(areaForLayer, root);

// Create DisplayAreas to cover all defined features.
// 获取feature的个数
final int size = mFeatures.size();
// 遍历所有feature
for (int i = 0; i < size; i++) {
    // Traverse the features with the order they are defined, so that the early defined
    // feature will be on the top in the hierarchy.
    final Feature feature = mFeatures.get(i);
    PendingArea featureArea = null;
    // 遍历所有层级
    for (int layer = 0; layer < maxWindowLayerCount; layer++) {
        // 如果当前feature需要显示到层级上
        if (feature.mWindowLayers[layer]) {
            // This feature will be applied to this window layer.
            //
            // We need to find a DisplayArea for it:
            // We can reuse the existing one if it was created for this feature for the
            // previous layer AND the last feature that applied to the previous layer is
            // the same as the feature that applied to the current layer (so they are ok
            // to share the same parent DisplayArea).
            // 
            // 尝试复用层级
            if (featureArea == null || featureArea.mParent != areaForLayer[layer]) {
                // No suitable DisplayArea:
                // Create a new one under the previous area (as parent) for this layer.
                featureArea = new PendingArea(feature, layer, areaForLayer[layer]);
                areaForLayer[layer].mChildren.add(featureArea);
            }
            // 建立子节点关系
            areaForLayer[layer] = featureArea;
        } else {
            // This feature won't be applied to this window layer. If it needs to be
            // applied to the next layer, we will need to create a new DisplayArea for
            // that.
            featureArea = null;
        }
    }
}
```



最终这一步会生成这样一个节点图

![aosp-window-tree-tree-node.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-window-tree-tree-node.drawio.png)



然后简单对齐后的结构图

![aosp-window-tree-tree-node-aligned.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-window-tree-tree-node-aligned.drawio.png)



#### 添加叶子节点

这一步骤主要是给所有layer层级都加上一层leaf叶子节点。

```java
// Create Tokens as leaf for every layer.
PendingArea leafArea = null;
int leafType = LEAF_TYPE_TOKENS;
// 1.遍历每一层layer节点
for (int layer = 0; layer < maxWindowLayerCount; layer++) {
    // 2. 获取当前层级节点的类型，
    int type = typeOfLayer(policy, layer);
    // Check whether we can reuse the same Tokens with the previous layer. This happens
    // if the previous layer is the same type as the current layer AND there is no
    // feature that applies to only one of them.
    // 3. 查看类型是否能复用，不能复用则会进入分支执行
    if (leafArea == null || leafArea.mParent != areaForLayer[layer]
            || type != leafType) {
        // 4. 不满足复用条件，创建新的layer
        // Create a new Tokens for this layer.
        leafArea = new PendingArea(null /* feature */, layer, areaForLayer[layer]);
        areaForLayer[layer].mChildren.add(leafArea);
        leafType = type;
        if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
            // We use the passed in TaskDisplayAreas for task container type of layer.
            // Skip creating Tokens even if there is no TDA.
            // 添加DisplayAreas
            addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
            addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                    displayAreaGroupHierarchyBuilders);
            leafArea.mSkipTokens = true;
        } else if (leafType == LEAF_TYPE_IME_CONTAINERS) {
            // We use the passed in ImeContainer for ime container type of layer.
            // Skip creating Tokens even if there is no ime container.
            leafArea.mExisting = mImeContainer;
            leafArea.mSkipTokens = true;
        }
    }
    leafArea.mMaxLayer = layer;
}

// 递归计算layer最大层级
root.computeMaxLayer();
```

- typeOfLayer

``` java
// 3类type取值
private static final int LEAF_TYPE_TASK_CONTAINERS = 1;
private static final int LEAF_TYPE_IME_CONTAINERS = 2;
private static final int LEAF_TYPE_TOKENS = 0;


private static int typeOfLayer(WindowManagerPolicy policy, int layer) {
    if (layer == APPLICATION_LAYER) {
        //  layer取值2
        return LEAF_TYPE_TASK_CONTAINERS;
    } else if (layer == policy.getWindowLayerFromTypeLw(TYPE_INPUT_METHOD)
            || layer == policy.getWindowLayerFromTypeLw(TYPE_INPUT_METHOD_DIALOG)) {
        // layer取值13 or 14
        return LEAF_TYPE_IME_CONTAINERS;
    } else {
        return LEAF_TYPE_TOKENS;
    }
}
```

- addTaskDisplayAreasToApplicationLayer

```java
// 将mTaskDisplayAreas中所有的layer添加到parentPendingArea中
private void addTaskDisplayAreasToApplicationLayer(PendingArea parentPendingArea) {
    final int count = mTaskDisplayAreas.size();
    for (int i = 0; i < count; i++) {
        PendingArea leafArea =
                new PendingArea(null /* feature */, APPLICATION_LAYER, parentPendingArea);
        leafArea.mExisting = mTaskDisplayAreas.get(i);
        leafArea.mMaxLayer = APPLICATION_LAYER;
        parentPendingArea.mChildren.add(leafArea);
    }
}
```

- addDisplayAreaGroupsToApplicationLayer

```java
 /** Adds roots of the DisplayAreaGroups to the application layer. */
// 将displayAreaGroupHierarchyBuilders中所有的layer添加到parentPendingArea中
private void addDisplayAreaGroupsToApplicationLayer(
        DisplayAreaPolicyBuilder.PendingArea parentPendingArea,
        @Nullable List<HierarchyBuilder> displayAreaGroupHierarchyBuilders) {
    if (displayAreaGroupHierarchyBuilders == null) {
        return;
    }
    final int count = displayAreaGroupHierarchyBuilders.size();
    for (int i = 0; i < count; i++) {
        DisplayAreaPolicyBuilder.PendingArea
                leafArea = new DisplayAreaPolicyBuilder.PendingArea(
                null /* feature */, APPLICATION_LAYER, parentPendingArea);
        leafArea.mExisting = displayAreaGroupHierarchyBuilders.get(i).mRoot;
        leafArea.mMaxLayer = APPLICATION_LAYER;
        parentPendingArea.mChildren.add(leafArea);
    }
}
```

执行完成后的层级结构如下

![aosp-window-tree-tree-node-leaf.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-window-tree-tree-node-leaf.drawio.png)



简单对齐以后的结构

![aosp-window-tree-tree-node-leaf-align.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-window-tree-tree-node-leaf-align.drawio.png)





#### 构建DisplayArea树节点 

这一步其实非常简单就调用了一下PendingArea.instantiateChildren方法

```java
// We built a tree of PendingAreas above with all the necessary info to represent the
// hierarchy, now create and attach real DisplayAreas to the root.
root.instantiateChildren(mRoot, displayAreaForLayer, 0, featureAreas);
```

我们来看一下具体实现，方法会接受部分参数，并将树按照层序遍历的方式添加到RootDisplayArea中

```java
void instantiateChildren(DisplayArea<DisplayArea> parent, DisplayArea.Tokens[] areaForLayer,
        int level, Map<Feature, List<DisplayArea<WindowContainer>>> areas) {
    // 1. 先将子节点按照layer层级小到大排序
    mChildren.sort(Comparator.comparingInt(pendingArea -> pendingArea.mMinLayer));
    // 2. 迭代遍历所有子节点
    for (int i = 0; i < mChildren.size(); i++) {
        // 2.1 调用PendingArea.createArea为子节点创建
        final PendingArea child = mChildren.get(i);
        final DisplayArea area = child.createArea(parent, areaForLayer);
        // 如果子节点创建为null跳过后续递归逻辑
        if (area == null) {
            // TaskDisplayArea and ImeContainer can be set at different hierarchy, so it can
            // be null.
            continue;
        }
        // 2.2 向DisplayArea中添加元素
        parent.addChild(area, WindowContainer.POSITION_TOP);
        // 2.3 将结果保存到map中，map key为feature, value为一个set
        if (child.mFeature != null) {
            areas.get(child.mFeature).add(area);
        }
        // 2.4 递归调用子节点
        child.instantiateChildren(area, areaForLayer, level + 1, areas);
    }
}
```



##### createArea

该方法用于创建DisplayArea对象，返回的DisplayArea对象最终会添加到RootDisplayArea节点树中。

其中DisplayArea的创建也是有一定的区分的

| Feature                                                      | DisplayArea          |
| :----------------------------------------------------------- | -------------------- |
| WindowedMagnification                                        | DisplayArea.Dimmable |
| HideDisplayCutout/OneHanded/FullscreenMagnification/ImePlaceholder | DisplayArea          |
| 叶子节点                                                     | DisplayArea.Tokens   |

```java
// PendingArea.java
@Nullable
private DisplayArea createArea(DisplayArea<DisplayArea> parent,
        DisplayArea.Tokens[] areaForLayer) {
    // 1. 为applicatiion layer & imeContainer 创建DisplayArea
    if (mExisting != null) {
        if (mExisting.asTokens() != null) {
            // Store the WindowToken container for layers
            fillAreaForLayers(mExisting.asTokens(), areaForLayer);
        }
        return mExisting;
    }
    // 2. 即使ImeContainer & TaskContainer无token直接反馈null,不递归创建DisplayArea.Token
    if (mSkipTokens) {
        return null;
    }
    // 3. 根据layer层级获取layer的类型
    DisplayArea.Type type;
    if (mMinLayer > APPLICATION_LAYER) {
        // 最小layer层级大于APPLICATION_LAYER
        // 当前layer只能包含WindowToken且在APPLICATION_LAYER层级上方
        type = DisplayArea.Type.ABOVE_TASKS;
    } else if (mMaxLayer < APPLICATION_LAYER) {
        // 最大层级小于APPLICATION_LAYER
        // 当前层级只能包含WindowToken且在APPLICATION_LAYER层级下方
        type = DisplayArea.Type.BELOW_TASKS;
    } else {
        // 当前层级可以包含任意内容
        type = DisplayArea.Type.ANY;
    }
    // 只有跟节点和叶子节点feature为null
    if (mFeature == null) {
        // 创建一个DisplayArea.Tokens并返回
        final DisplayArea.Tokens leaf = new DisplayArea.Tokens(parent.mWmService, type,
                "Leaf:" + mMinLayer + ":" + mMaxLayer);
        // 填充占用层级
        fillAreaForLayers(leaf, areaForLayer);
        return leaf;
    } else {
        // 使用Feature创建Area并返回
        // 除开WindowedMagnification是DisplayArea.Dimmable，其余都是DisplayArea
        return mFeature.mNewDisplayAreaSupplier.create(parent.mWmService, type,
                mFeature.mName + ":" + mMinLayer + ":" + mMaxLayer, mFeature.mId);
    }
}
```



##### addChild

instantiateChildren是一个dfs的递归，其中parent父节点会随着递归的进行发生变化。

```java
void instantiateChildren(DisplayArea<DisplayArea> parent, DisplayArea.Tokens[] areaForLayer,
            int level, Map<Feature, List<DisplayArea<WindowContainer>>> areas) {
        // ......
    
        for (int i = 0; i < mChildren.size(); i++) {
            final PendingArea child = mChildren.get(i);
            final DisplayArea area = child.createArea(parent, areaForLayer);
            // ......
            
            
            parent.addChild(area, WindowContainer.POSITION_TOP);
            // ......
            child.instantiateChildren(area, areaForLayer, level + 1, areas);
        }
    }
```

addChild有多个实现类，

WindowContainer, Task, TaskDisplayArea, TaskFragment

```java
// WindowContainer.java
// 只是一个支持添加到top/bottom的add方法。
@CallSuper
void addChild(E child, int index) {
    // 检查一，除开reparent, addChild不允许有设置parent
    if (!child.mReparenting && child.getParent() != null) {
        throw new IllegalArgumentException("addChild: container=" + child.getName()
                + " is already a child of container=" + child.getParent().getName()
                + " can't add to container=" + getName()
                + "\n callers=" + Debug.getCallers(15, "\n"));
    }

    // 检查二，check index的值是否正常。
    if ((index < 0 && index != POSITION_BOTTOM)
            || (index > mChildren.size() && index != POSITION_TOP)) {
        throw new IllegalArgumentException("addChild: invalid position=" + index
                + ", children number=" + mChildren.size());
    }

    // 设置到top, 插入位置为最末尾
    if (index == POSITION_TOP) {
        index = mChildren.size();
    } else if (index == POSITION_BOTTOM) {
        // 设置到bottom, 插入位置为0
        index = 0;
    }
    // 其他case不改变index

    // 插入到特定的index位置
    // mChildren是WindowList,而WindowList是ArrayList的子类。
    mChildren.add(index, child);

    // Set the parent after we've actually added a child in case a subclass depends on this.
    // 设置parent
    child.setParent(this);
}
```

TaskDisplayArea通过instantiate方法创建并传入。

```java
// TaskDisplayArea.java
@Override
public DisplayAreaPolicy instantiate(WindowManagerService wmService,
        DisplayContent content, RootDisplayArea root,
        DisplayArea.Tokens imeContainer) {
    
    // 1. 创建TaskDisplayArea
    final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
            "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);
    final List<TaskDisplayArea> tdaList = new ArrayList<>();
    tdaList.add(defaultTaskDisplayArea);

    final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);
    // 2. 设置ImeContainer，设置TaskDisplayAreas
    rootHierarchy.setImeContainer(imeContainer).setTaskDisplayAreas(tdaList);
    // ......

    // Instantiate the policy with the hierarchy defined above. This will create and attach
    // all the necessary DisplayAreas to the root.
    return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);
}
```



//TODOS

TaskDisplayArea以及Task是如何创建的？



### traversal build

mDisplayAreaGroupHierarchyBuilders大多情况下是空列表。

```java
for (int i = 0; i < mDisplayAreaGroupHierarchyBuilders.size(); i++) {
    HierarchyBuilder hierarchyBuilder = mDisplayAreaGroupHierarchyBuilders.get(i);
    hierarchyBuilder.build();
    displayAreaGroupRoots.add(hierarchyBuilder.mRoot);
}
```





## 其他





### 观





# Refs



[Android T 窗口层级其二 —— 层级结构树的构建(1)](https://juejin.cn/post/7363452651797659674)

[掘金博客——【Android 13源码分析】WindowContainer窗口层级-2-构建流程](https://juejin.cn/post/7338808893529423883#heading-4)











