---
title: aosp-surfaceflinger-dump
tags:
cover:
---
# 背景


最近迫切想要了解UI显示系统相关的一些内容，特别是SurfaceFlinger渲染相关的原理性内容。但是目前对SurfaceFlinger处于是一种两眼一抹黑的情况。
因此迫切需要一个解析的突破口，而本文是通过分析SurfaceFlinger的dump分析的方式尝试分析SurfaceFlinger的相关逻辑。

源码基于android-14.0.0_r73


# 样例数据

```shell

adb shell dumpsys SurfaceFlinger
Build configuration: [sf PRESENT_TIME_OFFSET=0 FORCE_HWC_FOR_RBG_TO_YUV=0 MAX_VIRT_DISPLAY_DIM=0 RUNNING_WITHOUT_SYNC_FRAMEWORK=0 NUM_FRAMEBUFFER_SURFACE_BUFFERS=3]

Display identification data:
Display 4619827677550801152 (HWC display 0): port=0 pnpId=GGL displayName="Common Panel"

Wide-Color information:
Device supports wide color: 1
DisplayColorSetting: Enhanced
Display 4619827677550801152 color modes:
    ColorMode::NATIVE (0)
    ColorMode::SRGB (7)
    ColorMode::DISPLAY_P3 (9)
    Current color mode: ColorMode::SRGB (7)

HDR events for display 4619827677550801152
11279338546: numHdrLayers(0), size(0x0), flags(0), desiredRatio(0.00)

Sync configuration: [using: EGL_ANDROID_native_fence_sync EGL_KHR_wait_sync]

Scheduler:
Features
    PresentFences=true
    KernelIdleTimer=false
    ContentDetection=true

Policy
    pacesetterDisplayId=4619827677550801152
    layerHistory={size=113, active=0}
	GameFrameRateOverrides=
		(uid, gameModeOverride, gameDefaultOverride)=
    touchTimer=200.000 ms
    displayPowerTimer=1000.000 ms

FrameRateOverrides=none

           app phase:      6233334 ns	         SF phase:      6166667 ns
           app duration:  16600000 ns	         SF duration:  10500000 ns
     early app phase:       133334 ns	   early SF phase:        66667 ns
     early app duration:  16600000 ns	   early SF duration:  16600000 ns
  GL early app phase:       133334 ns	GL early SF phase:        66667 ns
  GL early app duration:  16600000 ns	GL early SF duration:  16600000 ns
       HWC min duration:         0 ns

ScreenOff: 1d23:40:27.309
90.00 Hz: 0d09:04:49.210
60.00 Hz: 0d07:33:29.220
30.00 Hz: 0d00:00:00.532

Frame Targeting
    Pacesetter Display 4619827677550801152
        Total missed frame count: 8771
        HWC missed frame count: 8771
        GPU missed frame count: 7046



debugDisplayModeSetByBackdoor=false

         present offset:         0 ns	        VSYNC period:  16666667 ns

app: state=VSync VSyncState={displayId=4619827677550801152, count=16338235}
mWorkDuration=16.60 mReadyDuration=10.50 last vsync time 18.50ms relative to now
  pending events (count=0):
  connections (count=31):
    Connection{0xb4000071a706e950, VSyncRequest::None}
    Connection{0xb4000071a7095f50, VSyncRequest::None}
    Connection{0xb4000071a709b950, VSyncRequest::None}
    Connection{0xb4000071a709b710, VSyncRequest::None}
    Connection{0xb4000071a7072550, VSyncRequest::None}
    Connection{0xb4000071a706dbd0, VSyncRequest::None}
    Connection{0xb4000071a7073f90, VSyncRequest::None}
    Connection{0xb4000071a7076690, VSyncRequest::None}
    Connection{0xb4000071a7098350, VSyncRequest::None}
    Connection{0xb4000071a7097510, VSyncRequest::None}
    Connection{0xb4000071a7098d10, VSyncRequest::None}
    Connection{0xb4000071a70984d0, VSyncRequest::None}
    Connection{0xb4000071a7092590, VSyncRequest::None}
    Connection{0xb4000071a70a2550, VSyncRequest::None}
    Connection{0xb4000071a709c010, VSyncRequest::None}
    Connection{0xb4000071a7094690, VSyncRequest::None}
    Connection{0xb4000071a709d750, VSyncRequest::None}
    Connection{0xb4000071a70a44d0, VSyncRequest::None}
    Connection{0xb4000071a709f3d0, VSyncRequest::None}
    Connection{0xb4000071a709ff10, VSyncRequest::None}
    Connection{0xb4000071a70a5850, VSyncRequest::None}
    Connection{0xb4000071a709bdd0, VSyncRequest::Single}
    Connection{0xb4000071a70a3d50, VSyncRequest::None}
    Connection{0xb4000071a709c790, VSyncRequest::None}
    Connection{0xb4000071a70a35d0, VSyncRequest::None}
    Connection{0xb4000071a70a2a90, VSyncRequest::None}
    Connection{0xb4000071a709ffd0, VSyncRequest::None}
    Connection{0xb4000071a709edd0, VSyncRequest::None}
    Connection{0xb4000071a70a5790, VSyncRequest::None}
    Connection{0xb4000071a7094b10, VSyncRequest::None}
    Connection{0xb4000071a70a0390, VSyncRequest::None}

VsyncSchedule for pacesetter 4619827677550801152:
hwVsyncState=Disabled
pendingHwVsyncState=Disabled

VsyncController:
VsyncReactor in use
Has 0 unfired fences
mInternalIgnoreFences=0 mExternalIgnoreFences=0
mMoreSamplesNeeded=0 mPeriodConfirmationInProgress=0
mModePtrTransitioningTo=nullptr
No Last HW vsync
VSyncTracker:
	mDisplayModePtr={id=0, hwcId=36, resolution=1080x2400, vsyncRate=60.00 Hz, dpi=409.43x411.89, group=0, vrrConfig=N/A}
	Refresh Rate Map:
		For ideal period 11.11ms: period = 11.11ms, intercept = 0
		For ideal period 16.67ms: period = 16.71ms, intercept = -29334
VsyncDispatch:
	Timer:
		DebugState: Waiting
	mTimerSlack: 0.50ms mMinVsyncDistance: 3.00ms
	mIntendedWakeupTime: 7.94ms from now
	mLastTimerCallback: 8.64ms ago mLastTimerSchedule: 8.18ms ago
	Callbacks:
		sf:  
			workDuration: 16.60ms readyDuration: 0.00ms lastVsync: -25210.48ms relative to now
			mLastDispatchTime: 25199.37ms ago
		app:  [wake up in 7.92ms deadline in 24.52ms for vsync 35.02ms from now]
			workDuration: 16.60ms readyDuration: 10.50ms lastVsync: 18.31ms relative to now
			mLastDispatchTime: -18.31ms ago
		appSf:  
			workDuration: 11.11ms readyDuration: 10.50ms lastVsync: -27913.27ms relative to now
			mLastDispatchTime: 27902.14ms ago

SurfaceFlinger New Frontend Enabled:true
Active Layers - layers with client handles (count = 113)

Composition list
LayerStack=0
  Layer [6395] com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395
    visible reason= buffer=21006685044739 frame=14 color{< 0, 0, 0, 1 >}
    bounds={0,0,2400,1080}
    input{(0x0) canOccludePresentation touchCropId=6385 touchableRegion={0,0,2400,1080}}
  Layer [85] StatusBar#85
    visible reason= buffer=8087423418396 frame=6396 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY) touchableRegion={0,0,128,1080}}
  Layer [81] NavigationBar0#81
    visible reason= buffer=8087423418382 frame=30832 color{< 0, 0, 0, 1 >}
    bounds={0,2274,2400,1080} toDisplayTransform={ tx=0.0000 ty=2274.0000 }
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | WATCH_OUTSIDE_TOUCH | SLIPPERY) touchableRegion={0,2274,2400,1080}}
  Layer [66] ScreenDecorOverlay#66
    visible reason= buffer=8087423418370 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={492,0,128,610}}
  Layer [67] ScreenDecorOverlayBottom#67
    visible reason= buffer=8087423418376 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,2326,2400,1080} toDisplayTransform={ tx=0.0000 ty=2326.0000 }
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={0,2326,2326,0}}

Input list
LayerStack=0
  Layer [67] ScreenDecorOverlayBottom#67
    visible reason= buffer=8087423418376 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,2326,2400,1080} toDisplayTransform={ tx=0.0000 ty=2326.0000 }
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={0,2326,2326,0}}
  Layer [66] ScreenDecorOverlay#66
    visible reason= buffer=8087423418370 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={492,0,128,610}}
  Layer [79] [Gesture Monitor] swipe-up#79
    invisible reason=nothing to draw
    bounds={-10800,-24000,24000,10800}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | SPY) replaceTouchableRegionWithCrop touchableRegion={-10799,-23999,24000,10800}}
  Layer [81] NavigationBar0#81
    visible reason= buffer=8087423418382 frame=30832 color{< 0, 0, 0, 1 >}
    bounds={0,2274,2400,1080} toDisplayTransform={ tx=0.0000 ty=2274.0000 }
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | WATCH_OUTSIDE_TOUCH | SLIPPERY) touchableRegion={0,2274,2400,1080}}
  Layer [85] StatusBar#85
    visible reason= buffer=8087423418396 frame=6396 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY) touchableRegion={0,0,128,1080}}
  Layer [6395] com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395
    visible reason= buffer=21006685044739 frame=14 color{< 0, 0, 0, 1 >}
    bounds={0,0,2400,1080}
    input{(0x0) canOccludePresentation touchCropId=6385 touchableRegion={0,0,2400,1080}}
  Layer [6393] df705d4 ActivityRecordInputSink com.google.android.dialer/com.android.dialer.main.impl.MainActivity#6393
    invisible reason=nothing to draw
    bounds={-10800,-24000,24000,10800}
    input{(NO_INPUT_CHANNEL | NOT_FOCUSABLE) replaceTouchableRegionWithCrop touchableRegion={-10799,-23999,24000,10800}}
  Layer [6357] 97c0a45 ActivityRecordInputSink com.android.chrome/org.chromium.chrome.browser.ChromeTabbedActivity#6357
    invisible reason=hidden by parent or layer flag
    bounds={675.364,2114.09,2261.09,822.364} toDisplayTransform={ scale x=0.1361 y=0.1361  tx=675.3639 ty=2114.0933 }
    input{(NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE) replaceTouchableRegionWithCrop touchableRegion={675,2114,2261,822}}
  Layer [6235] 75c2726 ActivityRecordInputSink com.kwai.video/com.yxcorp.gifshow.tiny.TinyLaunchActivity#6235
    invisible reason=hidden by parent or layer flag
    bounds={0,302,2702,1080} toDisplayTransform={ tx=0.0000 ty=302.0000 }
    input{(NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE) replaceTouchableRegionWithCrop touchableRegion={0,302,2702,1080}}
  Layer [93] b2c2147 ActivityRecordInputSink com.android.launcher3/.uioverrides.QuickstepLauncher#93
    invisible reason=hidden by parent or layer flag
    bounds={0,0,2400,1080}
    input{(NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE) replaceTouchableRegionWithCrop touchableRegion={0,0,2400,1080}}
  Layer [70] com.android.systemui.wallpapers.ImageWallpaper#70
    invisible reason=hidden by parent or layer flag
    bounds={-10800,-24000,24000,10800} toDisplayTransform={ scale x=2.5782 y=2.5781  tx=-450.0000 ty=-120.0000 }
    input{(NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE | PREVENT_SPLITTING | IS_WALLPAPER) touchableRegion={-449,-119,-119,-449}}

Layer Hierarchy
 ROOT
 ├─ Display 0 name="Built-in Screen"#3
 │  ├─ WindowedMagnification:0:31#4
 │  │  ├─ HideDisplayCutout:0:14#5
 │  │  │  └─ OneHanded:0:14#6
 │  │  │     ├─ FullscreenMagnification:0:12#7
 │  │  │     │  ├─ Leaf:0:1#8
 │  │  │     │  │  └─ WallpaperWindowToken{955eacd token=android.os.Binder@e0c7864}#68
 │  │  │     │  │     └─ e217400 com.android.systemui.wallpapers.ImageWallpaper#69
 │  │  │     │  │        └─ com.android.systemui.wallpapers.ImageWallpaper#70
 │  │  │     │  │           └─ Wallpaper BBQ wrapper#71
 │  │  │     │  ├─ DefaultTaskDisplayArea#9
 │  │  │     │  │  ├─ Task=1#47
 │  │  │     │  │  │  ├─ Task=2087#86
 │  │  │     │  │  │  │  └─ ActivityRecord{1ec2161 u0 com.androi[...]verrides.QuickstepLauncher t2087}#87
 │  │  │     │  │  │  │     ├─ b2c2147 ActivityRecordInputSink com.[...]r3/.uioverrides.QuickstepLauncher#93
 │  │  │     │  │  │  │     └─ e3afbbd com.android.launcher3/com.an[...]er3.uioverrides.QuickstepLauncher#89
 │  │  │     │  │  │  └─ home_task_overlay_container#48
 │  │  │     │  │  ├─ Task=2#49
 │  │  │     │  │  │  ├─ Task=3#50
 │  │  │     │  │  │  │  └─ Dim layer#54
 │  │  │     │  │  │  └─ Task=4#51
 │  │  │     │  │  │     └─ Dim layer#55
 │  │  │     │  │  ├─ Task=2118#6227
 │  │  │     │  │  │  └─ ActivityRecord{a38fa81 u0 com.kwai.v[...].tiny.TinyLaunchActivity t2118}#6228
 │  │  │     │  │  │     ├─ 75c2726 ActivityRecordInputSink com.[...]gifshow.tiny.TinyLaunchActivity#6235
 │  │  │     │  │  │     └─ d7a4455 com.kwai.video/com.yxcorp.gifshow.tiny.TinyLaunchActivity#6238
 │  │  │     │  │  │        └─ cce5cda Panel:com.kwai.video/com.yxcorp.gifshow.tiny.TinyLaunchActivity#6255
 │  │  │     │  │  ├─ Task=2119#6350
 │  │  │     │  │  │  └─ ActivityRecord{9b20abc u0 com.androi[...]android.apps.chrome.Main t2119}#6351
 │  │  │     │  │  │     ├─ 97c0a45 ActivityRecordInputSink com.[...]me.browser.ChromeTabbedActivity#6357
 │  │  │     │  │  │     └─ f0063f3 com.android.chrome/com.google.android.apps.chrome.Main#6361
 │  │  │     │  │  └─ Task=2120#6385
 │  │  │     │  │     └─ ActivityRecord{f7a8f27 u0 com.google[...].GoogleDialtactsActivity t2120}#6386
 │  │  │     │  │        ├─ df705d4 ActivityRecordInputSink com.[...]d.dialer.main.impl.MainActivity#6393
 │  │  │     │  │        ├─ 734fecd com.google.android.dialer/co[...]ensions.GoogleDialtactsActivity#6394
 │  │  │     │  │        │  ├─ com.google.android.dialer/com.google[...]ensions.GoogleDialtactsActivity#6395
 │  │  │     │  │        │  └─ (Relative) ImeContainer#12 parent=6386
 │  │  │     │  │        │     └─ WindowToken{66faee type=2011 android.os.Binder@76f2069}#5163
 │  │  │     │  │        │        └─ Surface(name=def733c InputMethod)/@0[...]ation-leash of insets_animation#6401
 │  │  │     │  │        │           └─ def733c InputMethod#5164
 │  │  │     │  └─ Leaf:3:12#10
 │  │  │     │     └─ WindowToken{594c626 type=2038 android.os.BinderProxy@d7779a2}#52
 │  │  │     │        └─ c8b57c ShellDropTarget#53
 │  │  │     └─ ImePlaceholder:13:14#11
 │  │  ├─ OneHanded:15:15#13
 │  │  │  └─ FullscreenMagnification:15:15#14
 │  │  │     └─ Leaf:15:15#15
 │  │  │        └─ WindowToken{8d3b857 type=2000 android.os.BinderProxy@aa77df1}#77
 │  │  │           └─ Surface(name=e9a2f44 StatusBar)/@0x9[...]ation-leash of insets_animation#6399
 │  │  │              └─ e9a2f44 StatusBar#78
 │  │  │                 └─ StatusBar#85
 │  │  ├─ HideDisplayCutout:16:16#16
 │  │  │  └─ OneHanded:16:16#17
 │  │  │     └─ FullscreenMagnification:16:16#18
 │  │  │        └─ Leaf:16:16#19
 │  │  ├─ OneHanded:17:17#20
 │  │  │  └─ FullscreenMagnification:17:17#21
 │  │  │     └─ Leaf:17:17#22
 │  │  │        └─ WindowToken{5f35f75 type=2040 android.os.BinderProxy@f689c5f}#75
 │  │  │           └─ 9337b0a NotificationShade#76
 │  │  ├─ HideDisplayCutout:18:23#23
 │  │  │  └─ OneHanded:18:23#24
 │  │  │     └─ FullscreenMagnification:18:23#25
 │  │  │        └─ Leaf:18:23#26
 │  │  ├─ Leaf:24:25#27
 │  │  │  └─ WindowToken{65e11d9 type=2019 android.os.BinderProxy@fe89852}#73
 │  │  │     └─ Surface(name=47ff27f NavigationBar0)[...]ation-leash of insets_animation#6398
 │  │  │        └─ 47ff27f NavigationBar0#74
 │  │  │           └─ NavigationBar0#81
 │  │  └─ HideDisplayCutout:26:31#28
 │  │     └─ OneHanded:26:31#29
 │  │        ├─ FullscreenMagnification:26:27#30
 │  │        │  └─ Leaf:26:27#31
 │  │        ├─ Leaf:28:28#32
 │  │        └─ FullscreenMagnification:29:31#33
 │  │           └─ Leaf:29:31#34
 │  ├─ HideDisplayCutout:32:35#35
 │  │  ├─ OneHanded:32:32#36
 │  │  │  └─ Leaf:32:32#37
 │  │  ├─ FullscreenMagnification:33:33#38
 │  │  │  └─ Leaf:33:33#39
 │  │  └─ OneHanded:34:35#40
 │  │     └─ FullscreenMagnification:34:35#41
 │  │        └─ Leaf:34:35#42
 │  ├─ Leaf:36:36#43
 │  ├─ Accessibility Overlays#46
 │  ├─ Input Overlays#45
 │  │  └─ [Gesture Monitor] swipe-up#79
 │  └─ Display Overlays#44
 ├─ WindowToken{de6b26f type=2024 android.os.BinderProxy@27ef14e}#60
 │  └─ 5d6068b ScreenDecorOverlay#61
 │     └─ ScreenDecorOverlay#66
 └─ WindowToken{543080a type=2024 android.os.BinderProxy@3850075}#64
    └─ bb9a87b ScreenDecorOverlayBottom#65
       └─ ScreenDecorOverlayBottom#67
Offscreen Hierarchy
 ROOT
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0xe9f2399_transition-leash#6049
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0x6f4da8f_transition-leash#6107
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0x33494e8_transition-leash#6348
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0x924f67c_transition-leash#6383
 ├─ Surface(name=f0063f3 com.android.chr[...]mation-leash of starting_reveal#6365
 ├─ Surface(name=47ff27f NavigationBar0)[...]ation-leash of insets_animation#6378
 ├─ Surface(name=e9a2f44 StatusBar)/@0x9[...]ation-leash of insets_animation#6379
 ├─ Surface(name=734fecd com.google.andr[...]mation-leash of starting_reveal#6400
 ├─ (Relative) Input Consumer recents_animation_input_consumer#88
 ├─ Surface(name=Task=2118)/@0xc4d825_transition-leash#6336
 ├─ Surface(name=Task=1)/@0xab07a9d_transition-leash#6047
 ├─ Surface(name=Task=1)/@0x398e33_transition-leash#6105
 ├─ Surface(name=Task=1)/@0x3cc24fc_transition-leash#6346
 ├─ Surface(name=Task=1)/@0x70fd550_transition-leash#6381
 ├─ Surface(name=Task=2116)/@0x15ad2e3_transition-leash#6048
 ├─ Surface(name=Task=2117)/@0xd481769_transition-leash#6106
 ├─ Surface(name=Task=2118)/@0x4ed9da_transition-leash#6347
 └─ Surface(name=Task=2119)/@0xc4cb44e_transition-leash#6382

Display 4619827677550801152 (active) HWC layers:
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Layer name
           Z |  Window Type |  Comp Type |  Transform |   Disp Frame (LTRB) |          Source Crop (LTRB) |     Frame Rate (Explicit) (Seamlessness) [Focused]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 com.google.android.dialer/com.google[...]ensions.GoogleDialtactsActivity#6395
           5 |            1 |     DEVICE |          0 |    0    0 1080 2400 |    0.0    0.0 1080.0 2400.0 |                                              [*]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 StatusBar#85
           6 |         2000 |     DEVICE |          0 |    0    0 1080  128 |    0.0    0.0 1080.0  128.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 NavigationBar0#81
           7 |         2019 |     DEVICE |          0 |    0 2274 1080 2400 |    0.0    0.0 1080.0  126.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 ScreenDecorOverlay#66
           9 |         2024 |     DEVICE |          0 |    0    0 1080  128 |    0.0    0.0 1080.0  128.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 ScreenDecorOverlayBottom#67
          10 |         2024 |     DEVICE |          0 |    0 2326 1080 2400 |    0.0    0.0 1080.0   74.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------

Displays (1 entries)
Display 4619827677550801152
    connectionType=Internal
    colorModes=
        ColorMode::NATIVE
        ColorMode::SRGB
        ColorMode::DISPLAY_P3
    deviceProductInfo={name="Common Panel", manufacturerPnpId=GGL, productId=0, manufactureWeek=1, manufactureYear=1990, relativeAddress=[]}
    name="Common Panel"
    powerMode=On
    activeMode=60.00 Hz (60.00 Hz(60.00 Hz))
    displayModes=
        {id=0, hwcId=36, resolution=1080x2400, vsyncRate=60.00 Hz, dpi=409.43x411.89, group=0, vrrConfig=N/A}
        {id=1, hwcId=37, resolution=1080x2400, vsyncRate=90.00 Hz, dpi=409.43x411.89, group=0, vrrConfig=N/A}
    displayManagerPolicy={defaultModeId=1, allowGroupSwitching=false, primaryRanges={physical=[0.00 Hz, inf Hz], render=[0.00 Hz, 90.00 Hz]}, appRequestRanges={physical=[0.00 Hz, inf Hz], render=[0.00 Hz, 90.00 Hz]}}
    frameRateOverrideConfig=Enabled
    idleTimer=
        interval=1500.000 ms
        controller=Platform

Display 4619827677550801152 (physical, "Common Panel")
   Composition Display State:
   isEnabled=true isSecure=true usesDeviceComposition=true 
   usesClientComposition=false flipClientTarget=false reusedClientComposition=false 
   layerFilter={layerStack=0 toInternalDisplay=true }
   transform (ROT_0) (IDENTITY)
   layerStackSpace=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} 
   framebufferSpace=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} 
   orientedDisplaySpace=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} 
   displaySpace=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} 
   needsFiltering=false 
   colorMode=SRGB (7) renderIntent=ENHANCE (1) dataspace=V0_SRGB (142671872) 
   colorTransformMatrix=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] 
   displayBrightnessNits=-1.000000 sdrWhitePointNits=-1.000000 clientTargetBrightness=1.000000 displayBrightness=nullopt 
   compositionStrategyPredictionState=DISABLED 
   
   treat170mAsSrgb=true 


   Composition Display Color State:
   HWC Support: wideColorGamut=true hdr10plus=true hdr10=true hlg=true dv=false metadata=7 
   Hdr Luminance Info:desiredMinLuminance=0.000500 desiredMaxLuminance=800.000000 desiredMaxAverageLuminance=120.000000 

   Composition RenderSurface State:
   size=[1080 2400] ANativeWindow=0xb4000071870a99e0 (format 1) 
   FramebufferSurface
      mDataspace=Default (0)
      mAbandoned=0
      - BufferQueue mMaxAcquiredBufferCount=2 mMaxDequeuedBufferCount=1
        mDequeueBufferCannotBlock=0 mAsyncMode=0
        mQueueBufferCanDrop=0 mLegacyBufferDrop=1
        default-size=[1080x2400] default-format=1         transform-hint=00 frame-counter=928835
        mTransformHintInUse=00 mAutoPrerotation=0
      FIFO(0):
      (mConsumerName=FramebufferSurface, mConnectedApi=1, mConsumerUsageBits=6656, mId=1e900000000, producer=[489:/system/bin/surfaceflinger], consumer=[489:/system/bin/surfaceflinger])
      Slots:
       >[01:0xb40000708707a4d0] state=ACQUIRED 0xb4000070f706d170 frame=928835 [1080x2400:1088,  1]
        [00:0xb4000070870792d0] state=FREE     0xb4000070f70714d0 frame=928833 [1080x2400:1088,  1]
        [02:0xb40000708707a950] state=FREE     0xb4000070f7071210 frame=928834 [1080x2400:1088,  1]

   5 Layers
  - Output Layer 0xb4000071e71453f0(com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395)
        Region visibleRegion (this=0xb4000071e7145408, count=1)
    [  0,   0, 1080, 2400]
        Region visibleNonTransparentRegion (this=0xb4000071e7145470, count=1)
    [  0,   0, 1080, 2400]
        Region coveredRegion (this=0xb4000071e71454d8, count=2)
    [  0,   0, 1080, 128]
    [  0, 2274, 1080, 2400]
        Region output visibleRegion (this=0xb4000071e71455b0, count=1)
    [  0,   0, 1080, 2400]
        Region shadowRegion (this=0xb4000071e7145618, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e71456b0, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 0 1080 2400] sourceCrop=[0.000000 0.000000 1080.000000 2400.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e7145760, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e71457c8, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590ab640 composition=DEVICE (2) 
  - Output Layer 0xb4000071e7136c30(StatusBar#85)
        Region visibleRegion (this=0xb4000071e7136c48, count=1)
    [  0,   0, 1080, 128]
        Region visibleNonTransparentRegion (this=0xb4000071e7136cb0, count=1)
    [  0,   0, 1080, 128]
        Region coveredRegion (this=0xb4000071e7136d18, count=1)
    [  0,   0, 1080, 128]
        Region output visibleRegion (this=0xb4000071e7136df0, count=1)
    [  0,   0, 1080, 128]
        Region shadowRegion (this=0xb4000071e7136e58, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e7136ef0, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 0 1080 128] sourceCrop=[0.000000 0.000000 1080.000000 128.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e7136fa0, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e7137008, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590b1c70 composition=DEVICE (2) 
  - Output Layer 0xb4000071e715eff0(NavigationBar0#81)
        Region visibleRegion (this=0xb4000071e715f008, count=1)
    [  0, 2274, 1080, 2400]
        Region visibleNonTransparentRegion (this=0xb4000071e715f070, count=1)
    [  0, 2274, 1080, 2400]
        Region coveredRegion (this=0xb4000071e715f0d8, count=1)
    [  0, 2326, 1080, 2400]
        Region output visibleRegion (this=0xb4000071e715f1b0, count=1)
    [  0, 2274, 1080, 2400]
        Region shadowRegion (this=0xb4000071e715f218, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e715f2b0, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 2274 1080 2400] sourceCrop=[0.000000 0.000000 1080.000000 126.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e715f360, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e715f3c8, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590afa60 composition=DEVICE (2) 
  - Output Layer 0xb4000071e7131210(ScreenDecorOverlay#66)
        Region visibleRegion (this=0xb4000071e7131228, count=1)
    [  0,   0, 1080, 128]
        Region visibleNonTransparentRegion (this=0xb4000071e7131290, count=1)
    [  0,   0, 1080, 128]
        Region coveredRegion (this=0xb4000071e71312f8, count=1)
    [  0,   0,   0,   0]
        Region output visibleRegion (this=0xb4000071e71313d0, count=1)
    [  0,   0, 1080, 128]
        Region shadowRegion (this=0xb4000071e7131438, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e71314d0, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 0 1080 128] sourceCrop=[0.000000 0.000000 1080.000000 128.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e7131580, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e71315e8, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590ad850 composition=DEVICE (2) 
  - Output Layer 0xb4000071e712d1b0(ScreenDecorOverlayBottom#67)
        Region visibleRegion (this=0xb4000071e712d1c8, count=1)
    [  0, 2326, 1080, 2400]
        Region visibleNonTransparentRegion (this=0xb4000071e712d230, count=1)
    [  0, 2326, 1080, 2400]
        Region coveredRegion (this=0xb4000071e712d298, count=1)
    [  0,   0,   0,   0]
        Region output visibleRegion (this=0xb4000071e712d370, count=1)
    [  0, 2326, 1080, 2400]
        Region shadowRegion (this=0xb4000071e712d3d8, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e712d470, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 2326 1080 2400] sourceCrop=[0.000000 0.000000 1080.000000 74.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e712d520, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e712d588, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590b6090 composition=DEVICE (2) 


SurfaceFlinger global state:

 ------------RE GLES------------
EGL implementation : 1.5 Android META-EGL
EGL_ANDROID_front_buffer_auto_refresh EGL_ANDROID_get_native_client_buffer EGL_ANDROID_presentation_time EGL_EXT_surface_CTA861_3_metadata EGL_EXT_surface_SMPTE2086_metadata EGL_KHR_get_all_proc_addresses EGL_KHR_swap_buffers_with_damage EGL_ANDROID_get_frame_timestamps EGL_EXT_gl_colorspace_scrgb EGL_EXT_gl_colorspace_scrgb_linear EGL_EXT_gl_colorspace_display_p3_linear EGL_EXT_gl_colorspace_display_p3 EGL_EXT_gl_colorspace_display_p3_passthrough EGL_EXT_gl_colorspace_bt2020_hlg EGL_EXT_gl_colorspace_bt2020_linear EGL_EXT_gl_colorspace_bt2020_pq EGL_ANDROID_image_native_buffer EGL_ANDROID_native_fence_sync EGL_ANDROID_recordable EGL_EXT_create_context_robustness EGL_EXT_image_gl_colorspace EGL_EXT_pixel_format_float EGL_EXT_protected_content EGL_EXT_yuv_surface EGL_IMG_context_priority EGL_KHR_config_attribs EGL_KHR_create_context EGL_KHR_fence_sync EGL_KHR_gl_colorspace EGL_KHR_gl_renderbuffer_image EGL_KHR_gl_texture_2D_image EGL_KHR_gl_texture_3D_image EGL_KHR_gl_texture_cubemap_image EGL_KHR_image EGL_KHR_image_base EGL_KHR_mutable_render_buffer EGL_KHR_no_config_context EGL_KHR_partial_update EGL_KHR_surfaceless_context EGL_KHR_wait_sync EGL_NV_context_priority_realtime 
GLES: ARM, Mali-G78, OpenGL ES 3.2 v1.r47p0-01eac0.32ea38cfcac3afe9a9b43f4ca33f49a9
GL_EXT_debug_marker GL_ARM_rgba8 GL_ARM_mali_shader_binary GL_OES_depth24 GL_OES_depth_texture GL_OES_depth_texture_cube_map GL_OES_packed_depth_stencil GL_OES_rgb8_rgba8 GL_EXT_read_format_bgra GL_OES_compressed_paletted_texture GL_OES_compressed_ETC1_RGB8_texture GL_OES_standard_derivatives GL_OES_EGL_image GL_OES_EGL_image_external GL_OES_EGL_image_external_essl3 GL_OES_EGL_sync GL_OES_texture_npot GL_OES_vertex_half_float GL_OES_required_internalformat GL_OES_vertex_array_object GL_OES_mapbuffer GL_EXT_texture_format_BGRA8888 GL_EXT_texture_rg GL_EXT_texture_type_2_10_10_10_REV GL_OES_fbo_render_mipmap GL_OES_element_index_uint GL_EXT_shadow_samplers GL_OES_texture_compression_astc GL_KHR_texture_compression_astc_ldr GL_KHR_texture_compression_astc_hdr GL_KHR_texture_compression_astc_sliced_3d GL_EXT_texture_compression_astc_decode_mode GL_EXT_texture_compression_astc_decode_mode_rgb9e5 GL_KHR_debug GL_EXT_occlusion_query_boolean GL_EXT_disjoint_timer_query GL_EXT_blend_minmax GL_EXT_discard_framebuffer GL_OES_get_program_binary GL_OES_texture_3D GL_EXT_texture_storage GL_EXT_multisampled_render_to_texture GL_EXT_multisampled_render_to_texture2 GL_OES_surfaceless_context GL_OES_texture_stencil8 GL_EXT_shader_pixel_local_storage GL_ARM_shader_framebuffer_fetch GL_ARM_shader_framebuffer_fetch_depth_stencil GL_ARM_mali_program_binary GL_EXT_sRGB GL_EXT_sRGB_write_control GL_EXT_texture_sRGB_decode GL_EXT_texture_sRGB_R8 GL_EXT_texture_sRGB_RG8 GL_KHR_blend_equation_advanced GL_KHR_blend_equation_advanced_coherent GL_OES_texture_storage_multisample_2d_array GL_OES_shader_image_atomic GL_EXT_robustness GL_EXT_draw_buffers_indexed GL_OES_draw_buffers_indexed GL_EXT_texture_border_clamp GL_OES_texture_border_clamp GL_EXT_texture_cube_map_array GL_OES_texture_cube_map_array GL_OES_sample_variables GL_OES_sample_shading GL_OES_shader_multisample_interpolation GL_EXT_shader_io_blocks GL_OES_shader_io_blocks GL_EXT_tessellation_shader GL_OES_tessellation_shader GL_EXT_primitive_bounding_box GL_OES_primitive_bounding_box GL_EXT_geometry_shader GL_OES_geometry_shader GL_ANDROID_extension_pack_es31a GL_EXT_gpu_shader5 GL_OES_gpu_shader5 GL_EXT_texture_buffer GL_OES_texture_buffer GL_EXT_copy_image GL_OES_copy_image GL_EXT_shader_non_constant_global_initializers GL_EXT_color_buffer_half_float GL_EXT_unpack_subimage GL_EXT_color_buffer_float GL_EXT_float_blend GL_EXT_YUV_target GL_OVR_multiview GL_OVR_multiview2 GL_OVR_multiview_multisampled_render_to_texture GL_KHR_robustness GL_KHR_robust_buffer_access_behavior GL_EXT_draw_elements_base_vertex GL_OES_draw_elements_base_vertex GL_EXT_protected_textures GL_EXT_buffer_storage GL_EXT_external_buffer GL_EXT_EGL_image_array GL_EXT_clear_texture GL_ARM_shader_core_properties GL_EXT_texture_filter_anisotropic GL_OES_texture_float_linear GL_ARM_texture_unnormalized_coordinates GL_EXT_shader_framebuffer_fetch GL_EXT_clip_control GL_EXT_polygon_offset_clamp GL_EXT_EGL_image_storage 
RenderEngine supports protected context: 1
RenderEngine is in protected context: 0
RenderEngine shaders cached since last dump/primeCache: 0
Skia CPU Caches:  0 bytes, 0.00 bytes (0.00 bytes is purgeable)
Skia's GPU Caches:  12444996 bytes, 11.87 MB (11.32 MB is purgeable)
  Texture/RenderBuffer: 11.32 MB (7 entries)
    skia/gpu_resources/resource_452/texture_renderbuffer: size[9.89 MB] type[RenderTarget] label[ResourceAllocatorRegister] category[Scratch] purgeable_size[9.89 MB]
    skia/gpu_resources/resource_155/texture_renderbuffer: size[128.00 KB] type[RenderTarget] label[TextureRenderTarget_FullyLazyProxy] category[Scratch] purgeable_size[128.00 KB]
    skia/gpu_resources/resource_14/texture_renderbuffer: size[4.00 KB] type[RenderTarget] label[ResourceAllocatorRegister] category[Scratch] purgeable_size[4.00 KB]
    skia/gpu_resources/resource_13/texture_renderbuffer: size[64.00 KB] type[RenderTarget] label[MakeDevice] category[Scratch] purgeable_size[64.00 KB]
    skia/gpu_resources/resource_453/texture_renderbuffer: size[632.81 KB] type[RenderTarget] label[ResourceAllocatorRegister] category[Scratch] purgeable_size[632.81 KB]
    skia/gpu_resources/resource_15/texture_renderbuffer: size[4.00 KB] type[RenderTarget] label[ResourceAllocatorRegister] category[Scratch] purgeable_size[4.00 KB]
    skia/gpu_resources/resource_459/texture_renderbuffer: size[632.81 KB] type[RenderTarget] label[ResourceAllocatorRegister] category[Scratch] purgeable_size[632.81 KB]
  Texture: 192.00 bytes (2 entries)
    skia/gpu_resources/resource_38/texture: size[64.00 bytes] type[Texture] label[ProxyProvider_CreateNonMippedProxyFromBitmap] category[Image] purgeable_size[64.00 bytes]
    skia/gpu_resources/resource_8/texture: size[128.00 bytes] type[Texture] label[ProxyProvider_CreateNonMippedProxyFromBitmap] category[Scratch]
  Other: 562.50 KB (6 entries)
    skia/gpu_resources/resource_2: size[180.00 bytes] type[Buffer Object] label[MakeGlBuffer] category[Other] purgeable_size[180.00 bytes]
    skia/gpu_resources/resource_156: size[272.00 bytes] type[Buffer Object] label[MakeGlBuffer] category[Other] purgeable_size[272.00 bytes]
    skia/gpu_resources/resource_158: size[512.00 KB] type[StencilAttachment] label[GLAttachmentMakeStencil] category[Other]
    skia/gpu_resources/resource_32208: size[48.00 KB] type[Buffer Object] label[MakeGlBuffer] category[Scratch]
    skia/gpu_resources/resource_157: size[192.00 bytes] type[Buffer Object] label[MakeGlBuffer] category[Other] purgeable_size[192.00 bytes]
    skia/gpu_resources/resource_3: size[1.88 KB] type[Buffer Object] label[MakeGlBuffer] category[Other] purgeable_size[1.88 KB]
Skia's Wrapped Objects:
  Texture/RenderBuffer: 59.33 MB (6 entries)
    skia/gpu_resources/resource_296/texture_renderbuffer: size[9.89 MB]
    skia/gpu_resources/resource_294/texture_renderbuffer: size[9.89 MB]
    skia/gpu_resources/resource_309/texture_renderbuffer: size[9.89 MB]
    skia/gpu_resources/resource_303/texture_renderbuffer: size[9.89 MB]
    skia/gpu_resources/resource_306/texture_renderbuffer: size[9.89 MB]
    skia/gpu_resources/resource_292/texture_renderbuffer: size[9.89 MB]
  Texture: 29.66 MB (3 entries)
    skia/gpu_resources/resource_626283/texture: size[9.89 MB]
    skia/gpu_resources/resource_629241/texture: size[9.89 MB]
    skia/gpu_resources/resource_619619/texture: size[9.89 MB]
RenderEngine tracked buffers: 27
Dumping buffer ids...
- 0x131b00000001 - 1 refs 
- 0x131b00000002 - 1 refs 
- 0x131b00000003 - 1 refs 
- 0x131b00000000 - 1 refs 
- 0x75b0000000f - 1 refs 
- 0x75b0000001e - 1 refs 
- 0x75b0000001b - 1 refs 
- 0x75b0000000e - 1 refs 
- 0x75b00000010 - 1 refs 
- 0x75b00000011 - 1 refs 
- 0x75b0000001c - 1 refs 
- 0x75b00000001 - 1 refs 
- 0x1e900000000 - 1 refs 
- 0x75b00000003 - 1 refs 
- 0x1e900000002 - 1 refs 
- 0x75b0000001d - 1 refs 
- 0x75b00000002 - 1 refs 
- 0x1e900000001 - 1 refs 
- 0x1e900000003 - 1 refs 
- 0x75b0000000c - 1 refs 
- 0x75b0000000b - 1 refs 
- 0x75b0000000a - 1 refs 
- 0x75b00000009 - 1 refs 
- 0x75b00000008 - 1 refs 
- 0x75b00000000 - 1 refs 
- 0x1e900000005 - 1 refs 
- 0x1e900000004 - 1 refs 
RenderEngine AHB/BackendTexture cache size: 27
Dumping buffer ids...
- 0x131b00000001
- 0x131b00000002
- 0x131b00000003
- 0x131b00000000
- 0x75b0000000f
- 0x75b0000001e
- 0x75b0000001b
- 0x75b0000000e
- 0x75b00000010
- 0x75b00000011
- 0x75b0000001c
- 0x75b00000001
- 0x1e900000000
- 0x75b00000003
- 0x1e900000002
- 0x75b0000001d
- 0x75b00000002
- 0x1e900000001
- 0x1e900000003
- 0x75b0000000c
- 0x75b0000000b
- 0x75b0000000a
- 0x75b00000009
- 0x75b00000008
- 0x75b00000000
- 0x1e900000005
- 0x1e900000004

Skia's GPU Protected Caches:  0 bytes, 0.00 bytes (0.00 bytes is purgeable)
Skia's Protected Wrapped Objects:

RenderEngine runtime effects: 10
- inputDataspace: DCI-P3 sRGB Extended range
- outputDataspace: DCI-P3 sRGB Full range
undoPremultipliedAlpha: false
- inputDataspace: BT2020 SMPTE 2084 Limited range
- outputDataspace: DCI-P3 sRGB Full range
undoPremultipliedAlpha: false
- inputDataspace: BT2020 SMPTE 2084 Limited range
- outputDataspace: BT2020 SMPTE_170M Full range
undoPremultipliedAlpha: false
- inputDataspace: (deprecated) sRGB sRGB Full range
- outputDataspace: DCI-P3 sRGB Full range
undoPremultipliedAlpha: true
- inputDataspace: (deprecated) sRGB sRGB Full range
- outputDataspace: (deprecated) sRGB sRGB Full range
undoPremultipliedAlpha: true
- inputDataspace: DCI-P3 sRGB Extended range
- outputDataspace: (deprecated) sRGB sRGB Full range
undoPremultipliedAlpha: false
- inputDataspace: (deprecated) sRGB sRGB Full range
- outputDataspace: (deprecated) sRGB sRGB Full range
undoPremultipliedAlpha: false
- inputDataspace: DCI-P3 sRGB Extended range
- outputDataspace: DCI-P3 sRGB Full range
undoPremultipliedAlpha: false
- inputDataspace: (deprecated) sRGB sRGB Full range
- outputDataspace: DCI-P3 sRGB Full range
undoPremultipliedAlpha: false
- inputDataspace: Default
- outputDataspace: (deprecated) sRGB sRGB Full range
undoPremultipliedAlpha: false

ClientCache state:
 Cache owner: 0xb4000070d7093d10
 Cache owner: 0xb4000070d7095260
	ID: 8087423418396, size: 1080x128
	ID: 8087423418382, size: 1080x126
	ID: 8087423418383, size: 1080x126
	ID: 8087423418397, size: 1080x128
	ID: 8087423418398, size: 1080x128
	ID: 8087423418384, size: 1080x126
	ID: 8087423418395, size: 1080x128
	ID: 8087423418385, size: 1080x126
	ID: 8087423418378, size: 1080x74
	ID: 8087423418368, size: 1080x128
	ID: 8087423418380, size: 922x1024
	ID: 8087423418369, size: 1080x128
	ID: 8087423418377, size: 1080x74
	ID: 8087423418376, size: 1080x74
	ID: 8087423418370, size: 1080x128
	ID: 8087423418371, size: 1080x128
	ID: 8087423418379, size: 1080x74
 Cache owner: 0xb4000070d709ee70
 Cache owner: 0xb4000070d70c51a0
	ID: 21006685044737, size: 1080x2400
	ID: 21006685044738, size: 1080x2400
	ID: 21006685044739, size: 1080x2400
	ID: 21006685044736, size: 1080x2400
 Cache owner: 0xb4000070d71068b0
 Cache owner: 0xb4000070d7117a50
 Cache owner: 0xb4000070d7136740
  Region undefinedRegion (this=0xb40000703706e6e8, count=1)
    [  0,   0,   0,   0]
  orientation=ROTATION_0, isPoweredOn=1
  transaction-flags         : 00000000
  refresh-rate              : 60.00 Hz
  x-dpi                     : 409.43
  y-dpi                     : 411.89
  transaction time: 0.000000 us

Transaction tracing: enabled
  queued transactions=0 created layers=0 states=97
  number of entries: 3280 (0.50MB / 0.50MB) duration: 12992120ms

Planner info for display [Common Panel]
  Current layers:
  + Fingerprint 10635f0385e89a39, last update 25.230 sec ago, age 110
    Override buffer: 0xb4000070870791b0
    HolePunchLayer: 0x0	
    Cached set of:
      Layer [com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395]
       Buffer 0xb4000070870e2ff0    Format[RGBA_8888]       Protected [false]
      Layer [StatusBar#85]
       Buffer 0xb4000070870d6b10    Format[RGBA_8888]       Protected [false]
      Layer [NavigationBar0#81]
       Buffer 0xb4000070870cf610    Format[RGBA_8888]       Protected [false]
      Layer [ScreenDecorOverlay#66]
       Buffer 0xb4000070870d3510    Format[RGBA_8888]       Protected [false]
      Layer [ScreenDecorOverlayBottom#67]
       Buffer 0xb4000070870d0b70    Format[RGBA_8888]       Protected [false]
    Creation cost: 5676480
    Display cost: 2592000
h/w composer state:
  h/w composer enabled


lastUeventTime(08:00:00.000) lastTimestamp(0)
lastEnableVsyncTime(00:19:14.760)
lastDisableVsyncTime(00:19:14.868)
lastValidateTime(00:19:14.776)
lastPresentTime(00:19:14.780)

Resource Manager:
[RGB Restrictions]
HW-Node 1-1:
    maxDownScale = 1, maxUpscale = 1
    maxFullWidth = 8190, maxFullHeight = 8190
    minFullWidth = 16, minFullHeight = 16
    fullWidthAlign = 1, fullHeightAlign = 1
    maxCropWidth = 4096, maxCropHeight = 4096
    minCropWidth = 16, minCropHeight = 16
    cropXAlign = 1, cropYAlign = 1
    cropWidthAlign = 1, cropHeightAlign = 1

HW-Node 1-2:
Same as above

HW-Node 7-1:
    maxDownScale = 2, maxUpscale = 8
    maxFullWidth = 8190, maxFullHeight = 8190
    minFullWidth = 16, minFullHeight = 16
    fullWidthAlign = 1, fullHeightAlign = 1
    maxCropWidth = 4096, maxCropHeight = 4096
    minCropWidth = 16, minCropHeight = 16
    cropXAlign = 1, cropYAlign = 1
    cropWidthAlign = 1, cropHeightAlign = 1

HW-Node 7-2:
Same as above

HW-Node 10-1:
    maxDownScale = 4, maxUpscale = 8
    maxFullWidth = 8192, maxFullHeight = 8192
    minFullWidth = 1, minFullHeight = 1
    fullWidthAlign = 1, fullHeightAlign = 1
    maxCropWidth = 8192, maxCropHeight = 8192
    minCropWidth = 1, minCropHeight = 1
    cropXAlign = 1, cropYAlign = 1
    cropWidthAlign = 1, cropHeightAlign = 1

HW-Node 10-2:
Same as above

[YUV Restrictions]
HW-Node 1-1:
    maxDownScale = 1, maxUpscale = 1
    maxFullWidth = 8190, maxFullHeight = 8190
    minFullWidth = 16, minFullHeight = 16
    fullWidthAlign = 2, fullHeightAlign = 2
    maxCropWidth = 4096, maxCropHeight = 4096
    minCropWidth = 32, minCropHeight = 32
    cropXAlign = 2, cropYAlign = 2
    cropWidthAlign = 2, cropHeightAlign = 2

HW-Node 1-2:
Same as above

HW-Node 7-1:
    maxDownScale = 2, maxUpscale = 8
    maxFullWidth = 8190, maxFullHeight = 8190
    minFullWidth = 16, minFullHeight = 16
    fullWidthAlign = 2, fullHeightAlign = 2
    maxCropWidth = 4096, maxCropHeight = 4096
    minCropWidth = 32, minCropHeight = 32
    cropXAlign = 2, cropYAlign = 2
    cropWidthAlign = 2, cropHeightAlign = 2

HW-Node 7-2:
Same as above

HW-Node 10-1:
    maxDownScale = 4, maxUpscale = 8
    maxFullWidth = 8192, maxFullHeight = 8192
    minFullWidth = 1, minFullHeight = 1
    fullWidthAlign = 2, fullHeightAlign = 2
    maxCropWidth = 8192, maxCropHeight = 8192
    minCropWidth = 1, minCropHeight = 1
    cropXAlign = 2, cropYAlign = 2
    cropWidthAlign = 2, cropHeightAlign = 2

HW-Node 10-2:
Same as above

[MPP Dump]
DPP_GF0: types mppType(1), (p:1, l:0x 2), indexs(p:0, l:0), preAssignDisplay(0x 1)
	Enable: 1, HWState: 1, AssignedState: 3, assignedDisplay(0)
	PrevAssignedState: 3, PrevAssignedDisplayType: -1, ReservedDisplay: 0
	assinedSourceNum(1), Capacity(-1.000000), CapaUsed(0.000000), mCurrentDstBuf(0)
DPP_GF1: types mppType(1), (p:1, l:0x 2), indexs(p:1, l:0), preAssignDisplay(0x 1)
	Enable: 1, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: 0
	assinedSourceNum(0), Capacity(-1.000000), CapaUsed(0.000000), mCurrentDstBuf(0)
DPP_GF2: types mppType(1), (p:1, l:0x 2), indexs(p:2, l:0), preAssignDisplay(0x 2)
	Enable: 1, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: 0
	assinedSourceNum(0), Capacity(-1.000000), CapaUsed(0.000000), mCurrentDstBuf(0)
DPP_VGRFS0: types mppType(1), (p:7, l:0x40), indexs(p:0, l:0), preAssignDisplay(0x 1)
	Enable: 1, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: 0
	assinedSourceNum(0), Capacity(-1.000000), CapaUsed(0.000000), mCurrentDstBuf(0)
DPP_VGRFS1: types mppType(1), (p:7, l:0x40), indexs(p:1, l:0), preAssignDisplay(0x 1)
	Enable: 1, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: 0
	assinedSourceNum(0), Capacity(-1.000000), CapaUsed(0.000000), mCurrentDstBuf(0)
DPP_VGRFS2: types mppType(1), (p:7, l:0x40), indexs(p:2, l:0), preAssignDisplay(0x 1)
	Enable: 1, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: 0
	assinedSourceNum(0), Capacity(-1.000000), CapaUsed(0.000000), mCurrentDstBuf(0)
G2D0-YUV_PRI: types mppType(2), (p:10, l:0x1000), indexs(p:0, l:0), preAssignDisplay(0x 1)
	Enable: 1, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: 0, ReservedDisplay: 0
	assinedSourceNum(0), Capacity(3.500000), CapaUsed(0.000000), mCurrentDstBuf(1)
G2D0-YUV_PRI: types mppType(2), (p:10, l:0x1000), indexs(p:0, l:1), preAssignDisplay(0x 1)
	Enable: 1, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: 0
	assinedSourceNum(0), Capacity(3.500000), CapaUsed(0.000000), mCurrentDstBuf(0)
G2D0-YUV_EXT: types mppType(2), (p:10, l:0x1000), indexs(p:0, l:2), preAssignDisplay(0x 2)
	Enable: 1, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: 0
	assinedSourceNum(0), Capacity(3.500000), CapaUsed(0.000000), mCurrentDstBuf(0)
G2D0-RGB_PRI: types mppType(2), (p:10, l:0x2000), indexs(p:0, l:3), preAssignDisplay(0x 1)
	Enable: 0, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: -1
	assinedSourceNum(0), Capacity(3.500000), CapaUsed(0.000000), mCurrentDstBuf(0)
G2D0-RGB_EXT: types mppType(2), (p:10, l:0x2000), indexs(p:0, l:4), preAssignDisplay(0x 2)
	Enable: 0, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: -1
	assinedSourceNum(0), Capacity(3.500000), CapaUsed(0.000000), mCurrentDstBuf(0)
G2D0-COMBO_VIR: types mppType(2), (p:10, l:0x4000), indexs(p:0, l:5), preAssignDisplay(0x 2)
	Enable: 1, HWState: 0, AssignedState: 1, assignedDisplay(-1)
	PrevAssignedState: 1, PrevAssignedDisplayType: -1, ReservedDisplay: 0
	assinedSourceNum(0), Capacity(3.500000), CapaUsed(0.000000), mCurrentDstBuf(0)
special plane num: 0:

[PrimaryDisplay] display information size: 1080 x 2400, vsyncState: 2, colorMode: 7, colorTransformHint: 0, orientation 0
CompositionInfo (1)
mHasCompositionLayer(0)
	internal_format: 0x1, afbc: 1

CompositionInfo (2)
mHasCompositionLayer(0)

PanelGammaSource (0)

============================== dump layers ===========================================
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+---------+
| zOrder | priority |       handle       |     fd     |         compression         |  format   | dataSpace | colorTr | blend | planeAlpha |   fps   |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+---------+
|   0    |    1     | 0xb4000074a90a3370 | 40, 58, -1 | AFBC(mod:0x800000000000041) | RGBA_8888 | 0x8810000 |    0    |  0x2  |     1      | 4.58558 |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+---------+
+------------------+------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
|    sourceCrop    |    dispFrame     | blockRect  | tr  | windowIndex | type | exynosType | validateType | overlayInfo | GrallocBufferId |
+------------------+------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
| 0, 0, 1080, 2400 | 0, 0, 1080, 2400 | 0, 0, 0, 0 | 0x0 |      0      |  2   |     2      |      2       |     0x0     |  2126008811525  |
+------------------+------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
+---------+-----------+
| MPPFlag | dim ratio |
+---------+-----------+
| 0x6042  |     1     |
+---------+-----------+
MPPFlags for otfMPP
[DPP_GF0: 0x0] [DPP_GF1: 0x0] [DPP_GF2: 0x0] [DPP_VGRFS0: 0x0] [DPP_VGRFS1: 0x0] [DPP_VGRFS2: 0x0] 
MPPFlags for m2mMPP
[G2D0-YUV_PRI: 0x80] [G2D0-YUV_PRI: 0x80] [G2D0-YUV_EXT: 0x80] [G2D0-RGB_PRI: 0x0] [G2D0-RGB_EXT: 0x0] 
[G2D0-COMBO_VIR: 0x0] 
acquireFence: -1
	assignedMPP: DPP_GF0
	dump midImg
	bufferHandle: 0x0, fullWidth: 1080, fullHeight: 2400, x: 0, y: 0, w: 1080, h: 2400, format: RGBA_8888
	usageFlags: 0xb00, layerFlags: 0x       0, acquireFenceFd: -1, releaseFenceFd: -1
	dataSpace(142671872), blending(2), transform(0x 0), compression: None

============================== dump ignore layers ===========================================
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+---------+
| zOrder | priority |       handle       |     fd     |         compression         |  format   | dataSpace | colorTr | blend | planeAlpha |   fps   |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+---------+
|   1    |    1     | 0xb4000074a90a2c90 | 37, 38, -1 | AFBC(mod:0x800000000000041) | RGBA_8888 | 0x8810000 |    0    |  0x2  |     0      | 4.73082 |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+---------+
+-----------------+-----------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
|   sourceCrop    |    dispFrame    | blockRect  | tr  | windowIndex | type | exynosType | validateType | overlayInfo | GrallocBufferId |
+-----------------+-----------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
| 0, 0, 1080, 128 | 0, 0, 1080, 128 | 0, 0, 0, 0 | 0x0 |      0      |  2   |     2      |      2       |     0x0     |  2126008811564  |
+-----------------+-----------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
+---------+-----------+
| MPPFlag | dim ratio |
+---------+-----------+
| 0x6042  |     1     |
+---------+-----------+
MPPFlags for otfMPP
[DPP_GF0: 0x0] [DPP_GF1: 0x0] [DPP_GF2: 0x0] [DPP_VGRFS0: 0x0] [DPP_VGRFS1: 0x0] [DPP_VGRFS2: 0x0] 
MPPFlags for m2mMPP
[G2D0-YUV_PRI: 0x80] [G2D0-YUV_PRI: 0x80] [G2D0-YUV_EXT: 0x80] [G2D0-RGB_PRI: 0x0] [G2D0-RGB_EXT: 0x0] 
[G2D0-COMBO_VIR: 0x0] 
acquireFence: -1
	resource is not assigned.
	dump midImg
	bufferHandle: 0x0, fullWidth: 1080, fullHeight: 2400, x: 0, y: 0, w: 1080, h: 128, format: RGBA_8888
	usageFlags: 0xb00, layerFlags: 0x       0, acquireFenceFd: -1, releaseFenceFd: -1
	dataSpace(142671872), blending(2), transform(0x 0), compression: None
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+-----+
| zOrder | priority |       handle       |     fd     |         compression         |  format   | dataSpace | colorTr | blend | planeAlpha | fps |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+-----+
|   4    |    1     | 0xb4000074a90a0350 | 49, 50, -1 | AFBC(mod:0x800000000000041) | RGBA_8888 | 0x8810000 |    0    |  0x2  |     0      | 120 |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+-----+
+----------------+---------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
|   sourceCrop   |      dispFrame      | blockRect  | tr  | windowIndex | type | exynosType | validateType | overlayInfo | GrallocBufferId |
+----------------+---------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
| 0, 0, 1080, 74 | 0, 2326, 1080, 2400 | 0, 0, 0, 0 | 0x0 |      0      |  2   |     2      |      2       |     0x0     |  2126008811544  |
+----------------+---------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
+---------+-----------+
| MPPFlag | dim ratio |
+---------+-----------+
| 0x6042  |     1     |
+---------+-----------+
MPPFlags for otfMPP
[DPP_GF0: 0x0] [DPP_GF1: 0x0] [DPP_GF2: 0x0] [DPP_VGRFS0: 0x0] [DPP_VGRFS1: 0x0] [DPP_VGRFS2: 0x0] 
MPPFlags for m2mMPP
[G2D0-YUV_PRI: 0x80] [G2D0-YUV_PRI: 0x80] [G2D0-YUV_EXT: 0x80] [G2D0-RGB_PRI: 0x0] [G2D0-RGB_EXT: 0x0] 
[G2D0-COMBO_VIR: 0x0] 
acquireFence: -1
	resource is not assigned.
	dump midImg
	bufferHandle: 0x0, fullWidth: 1080, fullHeight: 2400, x: 0, y: 2326, w: 1080, h: 74, format: RGBA_8888
	usageFlags: 0xb00, layerFlags: 0x       0, acquireFenceFd: -1, releaseFenceFd: -1
	dataSpace(142671872), blending(2), transform(0x 0), compression: None
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+---------+
| zOrder | priority |       handle       |     fd     |         compression         |  format   | dataSpace | colorTr | blend | planeAlpha |   fps   |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+---------+
|   2    |    1     | 0xb4000074a90a1110 | 41, 44, -1 | AFBC(mod:0x800000000000041) | RGBA_8888 | 0x8810000 |    0    |  0x2  |     0      | 19.3821 |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+---------+
+-----------------+---------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
|   sourceCrop    |      dispFrame      | blockRect  | tr  | windowIndex | type | exynosType | validateType | overlayInfo | GrallocBufferId |
+-----------------+---------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
| 0, 0, 1080, 126 | 0, 2274, 1080, 2400 | 0, 0, 0, 0 | 0x0 |      0      |  2   |     2      |      2       |     0x0     |  2126008811550  |
+-----------------+---------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
+---------+-----------+
| MPPFlag | dim ratio |
+---------+-----------+
| 0x6042  |     1     |
+---------+-----------+
MPPFlags for otfMPP
[DPP_GF0: 0x0] [DPP_GF1: 0x0] [DPP_GF2: 0x0] [DPP_VGRFS0: 0x0] [DPP_VGRFS1: 0x0] [DPP_VGRFS2: 0x0] 
MPPFlags for m2mMPP
[G2D0-YUV_PRI: 0x80] [G2D0-YUV_PRI: 0x80] [G2D0-YUV_EXT: 0x80] [G2D0-RGB_PRI: 0x0] [G2D0-RGB_EXT: 0x0] 
[G2D0-COMBO_VIR: 0x0] 
acquireFence: -1
	resource is not assigned.
	dump midImg
	bufferHandle: 0x0, fullWidth: 1080, fullHeight: 2400, x: 0, y: 2274, w: 1080, h: 126, format: RGBA_8888
	usageFlags: 0xb00, layerFlags: 0x       0, acquireFenceFd: -1, releaseFenceFd: -1
	dataSpace(142671872), blending(2), transform(0x 0), compression: None
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+----------+
| zOrder | priority |       handle       |     fd     |         compression         |  format   | dataSpace | colorTr | blend | planeAlpha |   fps    |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+----------+
|   3    |    1     | 0xb4000074a90a6bd0 | 45, 46, -1 | AFBC(mod:0x800000000000041) | RGBA_8888 | 0x8810000 |    0    |  0x2  |     0      | 0.304247 |
+--------+----------+--------------------+------------+-----------------------------+-----------+-----------+---------+-------+------------+----------+
+------------------+------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
|    sourceCrop    |    dispFrame     | blockRect  | tr  | windowIndex | type | exynosType | validateType | overlayInfo | GrallocBufferId |
+------------------+------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
| 0, 0, 1080, 2400 | 0, 0, 1080, 2400 | 0, 0, 0, 0 | 0x0 |      0      |  2   |     2      |      2       |     0x0     |  2126008811538  |
+------------------+------------------+------------+-----+-------------+------+------------+--------------+-------------+-----------------+
+---------+-----------+
| MPPFlag | dim ratio |
+---------+-----------+
| 0x6042  |     1     |
+---------+-----------+
MPPFlags for otfMPP
[DPP_GF0: 0x0] [DPP_GF1: 0x0] [DPP_GF2: 0x0] [DPP_VGRFS0: 0x0] [DPP_VGRFS1: 0x0] [DPP_VGRFS2: 0x0] 
MPPFlags for m2mMPP
[G2D0-YUV_PRI: 0x80] [G2D0-YUV_PRI: 0x80] [G2D0-YUV_EXT: 0x80] [G2D0-RGB_PRI: 0x0] [G2D0-RGB_EXT: 0x0] 
[G2D0-COMBO_VIR: 0x0] 
acquireFence: -1
	resource is not assigned.
	dump midImg
	bufferHandle: 0x0, fullWidth: 1080, fullHeight: 2400, x: 0, y: 0, w: 1080, h: 2400, format: RGBA_8888
	usageFlags: 0xb00, layerFlags: 0x       0, acquireFenceFd: -1, releaseFenceFd: -1
	dataSpace(142671872), blending(2), transform(0x 0), compression: None

BrightnessController:
	sysfs support 1, max 4095, valid brightness table 1, lhbm supported 1, ghbm supported 1
	requests: enhance hbm 0, lhbm 0, brightness 0.174297, instant hbm 0, DimBrightness 0
	states: brighntess level 578, ghbm 0, dimming 1, lhbm 0	hdr layer state 0, unchecked lhbm request 0(0), unchecked ghbm request 0(0)
	dimming usage 0, hbm dimming 0, time us 5000000
	white point nits current 0.000000, previous 0.000000
	cabc supported 0, cabcMode 0
	ignore brightness update request 0
	acl mode supported 0, acl mode 0
	operation rate 0

Display idle timer: disabled
	[0] vote to 0 ns
	[1] vote to 0 ns
Min idle refresh rate: 0, default: 0
	[0] vote to 0 hz
	[1] vote to 0 hz
	[2] vote to 0 hz
	[3] vote to 0 hz
Refresh rate delay: 0 ns
	[0] vote to 0 ns
	[1] vote to 0 ns
	[2] vote to 0 ns
	[3] vote to 0 ns
GraphicBufferAllocator buffers:
        Handle |        Size |     W (Stride) x H | Layers |   Format |      Usage | Requestor
0xb4000070f706d170 | 10200.00 KiB | 1080 (1088) x 2400 |      1 |        1 | 0x    1b00 | FramebufferSurface
0xb4000070f706e770 | 10200.00 KiB | 1080 (1088) x 2400 |      1 |        1 | 0x     b00 | Planner
0xb4000070f706f270 | 10200.00 KiB | 1080 (1088) x 2400 |      1 |        1 | 0x     b00 | Planner
0xb4000070f706f690 | 10200.00 KiB | 1080 (1088) x 2400 |      1 |        1 | 0x     b00 | Planner
0xb4000070f7071210 | 10200.00 KiB | 1080 (1088) x 2400 |      1 |        1 | 0x    1b00 | FramebufferSurface
0xb4000070f70714d0 | 10200.00 KiB | 1080 (1088) x 2400 |      1 |        1 | 0x    1b00 | FramebufferSurface
Total allocated by GraphicBufferAllocator (estimate): 61200.00 KB
Imported gralloc buffers:
+ name:VRI[GoogleDialtactsActivity]#0(BLAST Consumer)0, id:2126008817486, size:10360.25KiB, w/h:1080x2400, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
+ name:VRI[GoogleDialtactsActivity]#0(BLAST Consumer)0, id:2126008817487, size:10360.25KiB, w/h:1080x2400, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
+ name:VRI[GoogleDialtactsActivity]#0(BLAST Consumer)0, id:2126008817488, size:10360.25KiB, w/h:1080x2400, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
+ name:VRI[GoogleDialtactsActivity]#0(BLAST Consumer)0, id:2126008817485, size:10360.25KiB, w/h:1080x2400, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
+ name:VRI[ScreenDecorOverlay]#0(BLAST Consumer)0, id:2126008811539, size:553.25KiB, w/h:1080x128, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:VRI[StatusBar]#5(BLAST Consumer)5, id:2126008811566, size:553.25KiB, w/h:1080x128, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:VRI[ScreenDecorOverlayBottom]#1(BLAST Consumer)1, id:2126008811545, size:346.25KiB, w/h:1080x74, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x80, stride:4352 bytes, size:354560
+ name:VRI[StatusBar]#5(BLAST Consumer)5, id:2126008811564, size:553.25KiB, w/h:1080x128, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:VRI[NavigationBar0]#3(BLAST Consumer)3, id:2126008811551, size:553.25KiB, w/h:1080x126, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:FramebufferSurface, id:2126008811520, size:10360.25KiB, w/h:1080x2400, usage: 0x1b00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
+ name:VRI[StatusBar]#5(BLAST Consumer)5, id:2126008811565, size:553.25KiB, w/h:1080x128, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:FramebufferSurface, id:2126008811521, size:10360.25KiB, w/h:1080x2400, usage: 0x1b00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
+ name:VRI[NavigationBar0]#3(BLAST Consumer)3, id:2126008811552, size:553.25KiB, w/h:1080x126, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:FramebufferSurface, id:2126008811522, size:10360.25KiB, w/h:1080x2400, usage: 0x1b00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
+ name:VRI[StatusBar]#5(BLAST Consumer)5, id:2126008811563, size:553.25KiB, w/h:1080x128, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:VRI[NavigationBar0]#3(BLAST Consumer)3, id:2126008811550, size:553.25KiB, w/h:1080x126, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:VRI[ScreenDecorOverlayBottom]#1(BLAST Consumer)1, id:2126008811547, size:346.25KiB, w/h:1080x74, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x80, stride:4352 bytes, size:354560
+ name:VRI[ScreenDecorOverlay]#0(BLAST Consumer)0, id:2126008811537, size:553.25KiB, w/h:1080x128, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:VRI[NavigationBar0]#3(BLAST Consumer)3, id:2126008811553, size:553.25KiB, w/h:1080x126, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:VRI[ScreenDecorOverlayBottom]#1(BLAST Consumer)1, id:2126008811544, size:346.25KiB, w/h:1080x74, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x80, stride:4352 bytes, size:354560
+ name:Planner, id:2126008811525, size:10360.25KiB, w/h:1080x2400, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x0, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
+ name:VRI[ScreenDecorOverlay]#0(BLAST Consumer)0, id:2126008811538, size:553.25KiB, w/h:1080x128, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:VRI[ScreenDecorOverlay]#0(BLAST Consumer)0, id:2126008811536, size:553.25KiB, w/h:1080x128, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x128, stride:4352 bytes, size:566528
+ name:Wallpaper#2(BLAST Consumer)2, id:2126008811548, size:3770.25KiB, w/h:922x1024, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:928x1024, stride:3712 bytes, size:3860736
+ name:VRI[ScreenDecorOverlayBottom]#1(BLAST Consumer)1, id:2126008811546, size:346.25KiB, w/h:1080x74, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x8810000, compressed: true
	planes: R/G/B/A:	 w/h:1088x80, stride:4352 bytes, size:354560
+ name:Planner, id:2126008811524, size:10360.25KiB, w/h:1080x2400, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x0, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
+ name:Planner, id:2126008811523, size:10360.25KiB, w/h:1080x2400, usage: 0xb00, req fmt:1, fourcc/mod:875708993/576460752303423553, dataspace: 0x0, compressed: true
	planes: R/G/B/A:	 w/h:1088x2400, stride:4352 bytes, size:10608896
Total imported by gralloc: 115396.75KiB
FlagManager values: 
use_adpf_cpu_hint: true
use_skia_tracing: false
refresh_rate_overlay_on_external_display: false
connected_display: true
enable_small_area_detection: true
misc1: true
vrr_config: true
hotplug2: false
hdcp_level_hal: false
multithreaded_present: false
add_sf_skipped_frames_to_trace: false
use_known_refresh_rate_for_fps_consistency: false
cache_when_source_crop_layer_only_moved: false
enable_fro_dependent_features: true
display_protected: true
fp16_client_target: false
game_default_frame_rate: false
enable_layer_command_batching: true
screenshot_fence_preservation: false
vulkan_renderengine: false
renderable_buffer_usage: false
restore_blur_step: false
dont_skip_on_early_ro: false
protected_if_client: true
TimeStats miniDump:
Number of layers currently being tracked is 0
Number of layers in the stats pool is 0

Window Infos:
  max send vsync id: -1
  max send delay (ns): 0 ns
  unsent messages: 0


```

# dump入口位置



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


> 源代码如下：
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

> 简单总结一下主要有几个部分

| 日志类型                        | 详细信息                                                                                                           |
| --------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Build configuration         | surfaceflinger相关的一些配置，可能和构建参数有关                                                                                |
| Display identification data | 输出DisplayIdentification 数据                                                                                     |
| Wide-Color information      | 屏幕的色宽信息，HDR信息                                                                                                  |
| Sync configuration          | 同Build configuration只不过是sync相关的配置                                                                              |
| Scheduler                   | SurfaceFlinger Scheduler信息，包含如下信息<br>1.输出Scheduler的相关配置信息<br>2.输出EventThread的相关配置信息<br>3.输出VsyncSchedule相关配置信息 |

> 此处对应的日志输出如下

```shell
# 1.Build configuration日志开始
Build configuration: [sf PRESENT_TIME_OFFSET=0 FORCE_HWC_FOR_RBG_TO_YUV=0 MAX_VIRT_DISPLAY_DIM=0 RUNNING_WITHOUT_SYNC_FRAMEWORK=0 NUM_FRAMEBUFFER_SURFACE_BUFFERS=3]

# 2.Display identification data日志开始
Display identification data:
Display 4619827677550801152 (HWC display 0): port=0 pnpId=GGL displayName="Common Panel"

# 3.Wide-Color information日志开始
Wide-Color information:
Device supports wide color: 1
DisplayColorSetting: Enhanced
Display 4619827677550801152 color modes:
    ColorMode::NATIVE (0)
    ColorMode::SRGB (7)
    ColorMode::DISPLAY_P3 (9)
    Current color mode: ColorMode::SRGB (7)

HDR events for display 4619827677550801152
11279338546: numHdrLayers(0), size(0x0), flags(0), desiredRatio(0.00)

# 4.Sync configuration日志开始
Sync configuration: [using: EGL_ANDROID_native_fence_sync EGL_KHR_wait_sync]

# 5.Scheduler日志开始
Scheduler:
# 5.1 Scheduler相关配置
Features
    PresentFences=true
    KernelIdleTimer=false
    ContentDetection=true

Policy
    pacesetterDisplayId=4619827677550801152
    layerHistory={size=113, active=0}
	GameFrameRateOverrides=
		(uid, gameModeOverride, gameDefaultOverride)=
    touchTimer=200.000 ms
    displayPowerTimer=1000.000 ms

FrameRateOverrides=none

           app phase:      6233334 ns	         SF phase:      6166667 ns
           app duration:  16600000 ns	         SF duration:  10500000 ns
     early app phase:       133334 ns	   early SF phase:        66667 ns
     early app duration:  16600000 ns	   early SF duration:  16600000 ns
  GL early app phase:       133334 ns	GL early SF phase:        66667 ns
  GL early app duration:  16600000 ns	GL early SF duration:  16600000 ns
       HWC min duration:         0 ns

ScreenOff: 1d23:40:27.309
90.00 Hz: 0d09:04:49.210
60.00 Hz: 0d07:33:29.220
30.00 Hz: 0d00:00:00.532

Frame Targeting
    Pacesetter Display 4619827677550801152
        Total missed frame count: 8771
        HWC missed frame count: 8771
        GPU missed frame count: 7046



debugDisplayModeSetByBackdoor=false

         present offset:         0 ns	        VSYNC period:  16666667 ns
# 5.2 Vsync分发EventThread配置信息
app: state=VSync VSyncState={displayId=4619827677550801152, count=16338235}
mWorkDuration=16.60 mReadyDuration=10.50 last vsync time 18.50ms relative to now
  pending events (count=0):
  connections (count=31):
    Connection{0xb4000071a706e950, VSyncRequest::None}
    Connection{0xb4000071a7095f50, VSyncRequest::None}
    Connection{0xb4000071a709b950, VSyncRequest::None}
    Connection{0xb4000071a709b710, VSyncRequest::None}
    Connection{0xb4000071a7072550, VSyncRequest::None}
    Connection{0xb4000071a706dbd0, VSyncRequest::None}
    Connection{0xb4000071a7073f90, VSyncRequest::None}
    Connection{0xb4000071a7076690, VSyncRequest::None}
    Connection{0xb4000071a7098350, VSyncRequest::None}
    Connection{0xb4000071a7097510, VSyncRequest::None}
    Connection{0xb4000071a7098d10, VSyncRequest::None}
    Connection{0xb4000071a70984d0, VSyncRequest::None}
    Connection{0xb4000071a7092590, VSyncRequest::None}
    Connection{0xb4000071a70a2550, VSyncRequest::None}
    Connection{0xb4000071a709c010, VSyncRequest::None}
    Connection{0xb4000071a7094690, VSyncRequest::None}
    Connection{0xb4000071a709d750, VSyncRequest::None}
    Connection{0xb4000071a70a44d0, VSyncRequest::None}
    Connection{0xb4000071a709f3d0, VSyncRequest::None}
    Connection{0xb4000071a709ff10, VSyncRequest::None}
    Connection{0xb4000071a70a5850, VSyncRequest::None}
    Connection{0xb4000071a709bdd0, VSyncRequest::Single}
    Connection{0xb4000071a70a3d50, VSyncRequest::None}
    Connection{0xb4000071a709c790, VSyncRequest::None}
    Connection{0xb4000071a70a35d0, VSyncRequest::None}
    Connection{0xb4000071a70a2a90, VSyncRequest::None}
    Connection{0xb4000071a709ffd0, VSyncRequest::None}
    Connection{0xb4000071a709edd0, VSyncRequest::None}
    Connection{0xb4000071a70a5790, VSyncRequest::None}
    Connection{0xb4000071a7094b10, VSyncRequest::None}
    Connection{0xb4000071a70a0390, VSyncRequest::None}
# 5.3 VsyncSchedule相关配置
VsyncSchedule for pacesetter 4619827677550801152:
hwVsyncState=Disabled
pendingHwVsyncState=Disabled

VsyncController:
VsyncReactor in use
Has 0 unfired fences
mInternalIgnoreFences=0 mExternalIgnoreFences=0
mMoreSamplesNeeded=0 mPeriodConfirmationInProgress=0
mModePtrTransitioningTo=nullptr
No Last HW vsync
VSyncTracker:
	mDisplayModePtr={id=0, hwcId=36, resolution=1080x2400, vsyncRate=60.00 Hz, dpi=409.43x411.89, group=0, vrrConfig=N/A}
	Refresh Rate Map:
		For ideal period 11.11ms: period = 11.11ms, intercept = 0
		For ideal period 16.67ms: period = 16.71ms, intercept = -29334
VsyncDispatch:
	Timer:
		DebugState: Waiting
	mTimerSlack: 0.50ms mMinVsyncDistance: 3.00ms
	mIntendedWakeupTime: 7.94ms from now
	mLastTimerCallback: 8.64ms ago mLastTimerSchedule: 8.18ms ago
	Callbacks:
		sf:  
			workDuration: 16.60ms readyDuration: 0.00ms lastVsync: -25210.48ms relative to now
			mLastDispatchTime: 25199.37ms ago
		app:  [wake up in 7.92ms deadline in 24.52ms for vsync 35.02ms from now]
			workDuration: 16.60ms readyDuration: 10.50ms lastVsync: 18.31ms relative to now
			mLastDispatchTime: -18.31ms ago
		appSf:  
			workDuration: 11.11ms readyDuration: 10.50ms lastVsync: -27913.27ms relative to now
			mLastDispatchTime: 27902.14ms ago
```


## Dump the visible layer list



> 源代码如下

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



> 简单总结一下主要有几个部分


| 日志类型                                | 详细信息                                        |
| ----------------------------------- | ------------------------------------------- |
| SurfaceFlinger New Frontend Enabled | 取值为bool值，含义为是否启用SurfaceFlinger New Frontend |
| Active Layers                       | 取值为int值，含义为active layers数值                  |
| composition List                    |                                             |
| Displays                            |                                             |
| dumpDisplays                        |                                             |
| dumpCompositionDisplays             |                                             |
| mCompositionEngine->dump            |                                             |

> 日志输出如下

```shell

SurfaceFlinger New Frontend Enabled:true
Active Layers - layers with client handles (count = 113)

Composition list
LayerStack=0
  Layer [6395] com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395
    visible reason= buffer=21006685044739 frame=14 color{< 0, 0, 0, 1 >}
    bounds={0,0,2400,1080}
    input{(0x0) canOccludePresentation touchCropId=6385 touchableRegion={0,0,2400,1080}}
  Layer [85] StatusBar#85
    visible reason= buffer=8087423418396 frame=6396 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY) touchableRegion={0,0,128,1080}}
  Layer [81] NavigationBar0#81
    visible reason= buffer=8087423418382 frame=30832 color{< 0, 0, 0, 1 >}
    bounds={0,2274,2400,1080} toDisplayTransform={ tx=0.0000 ty=2274.0000 }
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | WATCH_OUTSIDE_TOUCH | SLIPPERY) touchableRegion={0,2274,2400,1080}}
  Layer [66] ScreenDecorOverlay#66
    visible reason= buffer=8087423418370 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={492,0,128,610}}
  Layer [67] ScreenDecorOverlayBottom#67
    visible reason= buffer=8087423418376 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,2326,2400,1080} toDisplayTransform={ tx=0.0000 ty=2326.0000 }
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={0,2326,2326,0}}

Input list
LayerStack=0
  Layer [67] ScreenDecorOverlayBottom#67
    visible reason= buffer=8087423418376 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,2326,2400,1080} toDisplayTransform={ tx=0.0000 ty=2326.0000 }
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={0,2326,2326,0}}
  Layer [66] ScreenDecorOverlay#66
    visible reason= buffer=8087423418370 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={492,0,128,610}}
  Layer [79] [Gesture Monitor] swipe-up#79
    invisible reason=nothing to draw
    bounds={-10800,-24000,24000,10800}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | SPY) replaceTouchableRegionWithCrop touchableRegion={-10799,-23999,24000,10800}}
  Layer [81] NavigationBar0#81
    visible reason= buffer=8087423418382 frame=30832 color{< 0, 0, 0, 1 >}
    bounds={0,2274,2400,1080} toDisplayTransform={ tx=0.0000 ty=2274.0000 }
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | WATCH_OUTSIDE_TOUCH | SLIPPERY) touchableRegion={0,2274,2400,1080}}
  Layer [85] StatusBar#85
    visible reason= buffer=8087423418396 frame=6396 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY) touchableRegion={0,0,128,1080}}
  Layer [6395] com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395
    visible reason= buffer=21006685044739 frame=14 color{< 0, 0, 0, 1 >}
    bounds={0,0,2400,1080}
    input{(0x0) canOccludePresentation touchCropId=6385 touchableRegion={0,0,2400,1080}}
  Layer [6393] df705d4 ActivityRecordInputSink com.google.android.dialer/com.android.dialer.main.impl.MainActivity#6393
    invisible reason=nothing to draw
    bounds={-10800,-24000,24000,10800}
    input{(NO_INPUT_CHANNEL | NOT_FOCUSABLE) replaceTouchableRegionWithCrop touchableRegion={-10799,-23999,24000,10800}}
  Layer [6357] 97c0a45 ActivityRecordInputSink com.android.chrome/org.chromium.chrome.browser.ChromeTabbedActivity#6357
    invisible reason=hidden by parent or layer flag
    bounds={675.364,2114.09,2261.09,822.364} toDisplayTransform={ scale x=0.1361 y=0.1361  tx=675.3639 ty=2114.0933 }
    input{(NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE) replaceTouchableRegionWithCrop touchableRegion={675,2114,2261,822}}
  Layer [6235] 75c2726 ActivityRecordInputSink com.kwai.video/com.yxcorp.gifshow.tiny.TinyLaunchActivity#6235
    invisible reason=hidden by parent or layer flag
    bounds={0,302,2702,1080} toDisplayTransform={ tx=0.0000 ty=302.0000 }
    input{(NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE) replaceTouchableRegionWithCrop touchableRegion={0,302,2702,1080}}
  Layer [93] b2c2147 ActivityRecordInputSink com.android.launcher3/.uioverrides.QuickstepLauncher#93
    invisible reason=hidden by parent or layer flag
    bounds={0,0,2400,1080}
    input{(NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE) replaceTouchableRegionWithCrop touchableRegion={0,0,2400,1080}}
  Layer [70] com.android.systemui.wallpapers.ImageWallpaper#70
    invisible reason=hidden by parent or layer flag
    bounds={-10800,-24000,24000,10800} toDisplayTransform={ scale x=2.5782 y=2.5781  tx=-450.0000 ty=-120.0000 }
    input{(NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE | PREVENT_SPLITTING | IS_WALLPAPER) touchableRegion={-449,-119,-119,-449}}

Layer Hierarchy
 ROOT
 ├─ Display 0 name="Built-in Screen"#3
 │  ├─ WindowedMagnification:0:31#4
 │  │  ├─ HideDisplayCutout:0:14#5
 │  │  │  └─ OneHanded:0:14#6
 │  │  │     ├─ FullscreenMagnification:0:12#7
 │  │  │     │  ├─ Leaf:0:1#8
 │  │  │     │  │  └─ WallpaperWindowToken{955eacd token=android.os.Binder@e0c7864}#68
 │  │  │     │  │     └─ e217400 com.android.systemui.wallpapers.ImageWallpaper#69
 │  │  │     │  │        └─ com.android.systemui.wallpapers.ImageWallpaper#70
 │  │  │     │  │           └─ Wallpaper BBQ wrapper#71
 │  │  │     │  ├─ DefaultTaskDisplayArea#9
 │  │  │     │  │  ├─ Task=1#47
 │  │  │     │  │  │  ├─ Task=2087#86
 │  │  │     │  │  │  │  └─ ActivityRecord{1ec2161 u0 com.androi[...]verrides.QuickstepLauncher t2087}#87
 │  │  │     │  │  │  │     ├─ b2c2147 ActivityRecordInputSink com.[...]r3/.uioverrides.QuickstepLauncher#93
 │  │  │     │  │  │  │     └─ e3afbbd com.android.launcher3/com.an[...]er3.uioverrides.QuickstepLauncher#89
 │  │  │     │  │  │  └─ home_task_overlay_container#48
 │  │  │     │  │  ├─ Task=2#49
 │  │  │     │  │  │  ├─ Task=3#50
 │  │  │     │  │  │  │  └─ Dim layer#54
 │  │  │     │  │  │  └─ Task=4#51
 │  │  │     │  │  │     └─ Dim layer#55
 │  │  │     │  │  ├─ Task=2118#6227
 │  │  │     │  │  │  └─ ActivityRecord{a38fa81 u0 com.kwai.v[...].tiny.TinyLaunchActivity t2118}#6228
 │  │  │     │  │  │     ├─ 75c2726 ActivityRecordInputSink com.[...]gifshow.tiny.TinyLaunchActivity#6235
 │  │  │     │  │  │     └─ d7a4455 com.kwai.video/com.yxcorp.gifshow.tiny.TinyLaunchActivity#6238
 │  │  │     │  │  │        └─ cce5cda Panel:com.kwai.video/com.yxcorp.gifshow.tiny.TinyLaunchActivity#6255
 │  │  │     │  │  ├─ Task=2119#6350
 │  │  │     │  │  │  └─ ActivityRecord{9b20abc u0 com.androi[...]android.apps.chrome.Main t2119}#6351
 │  │  │     │  │  │     ├─ 97c0a45 ActivityRecordInputSink com.[...]me.browser.ChromeTabbedActivity#6357
 │  │  │     │  │  │     └─ f0063f3 com.android.chrome/com.google.android.apps.chrome.Main#6361
 │  │  │     │  │  └─ Task=2120#6385
 │  │  │     │  │     └─ ActivityRecord{f7a8f27 u0 com.google[...].GoogleDialtactsActivity t2120}#6386
 │  │  │     │  │        ├─ df705d4 ActivityRecordInputSink com.[...]d.dialer.main.impl.MainActivity#6393
 │  │  │     │  │        ├─ 734fecd com.google.android.dialer/co[...]ensions.GoogleDialtactsActivity#6394
 │  │  │     │  │        │  ├─ com.google.android.dialer/com.google[...]ensions.GoogleDialtactsActivity#6395
 │  │  │     │  │        │  └─ (Relative) ImeContainer#12 parent=6386
 │  │  │     │  │        │     └─ WindowToken{66faee type=2011 android.os.Binder@76f2069}#5163
 │  │  │     │  │        │        └─ Surface(name=def733c InputMethod)/@0[...]ation-leash of insets_animation#6401
 │  │  │     │  │        │           └─ def733c InputMethod#5164
 │  │  │     │  └─ Leaf:3:12#10
 │  │  │     │     └─ WindowToken{594c626 type=2038 android.os.BinderProxy@d7779a2}#52
 │  │  │     │        └─ c8b57c ShellDropTarget#53
 │  │  │     └─ ImePlaceholder:13:14#11
 │  │  ├─ OneHanded:15:15#13
 │  │  │  └─ FullscreenMagnification:15:15#14
 │  │  │     └─ Leaf:15:15#15
 │  │  │        └─ WindowToken{8d3b857 type=2000 android.os.BinderProxy@aa77df1}#77
 │  │  │           └─ Surface(name=e9a2f44 StatusBar)/@0x9[...]ation-leash of insets_animation#6399
 │  │  │              └─ e9a2f44 StatusBar#78
 │  │  │                 └─ StatusBar#85
 │  │  ├─ HideDisplayCutout:16:16#16
 │  │  │  └─ OneHanded:16:16#17
 │  │  │     └─ FullscreenMagnification:16:16#18
 │  │  │        └─ Leaf:16:16#19
 │  │  ├─ OneHanded:17:17#20
 │  │  │  └─ FullscreenMagnification:17:17#21
 │  │  │     └─ Leaf:17:17#22
 │  │  │        └─ WindowToken{5f35f75 type=2040 android.os.BinderProxy@f689c5f}#75
 │  │  │           └─ 9337b0a NotificationShade#76
 │  │  ├─ HideDisplayCutout:18:23#23
 │  │  │  └─ OneHanded:18:23#24
 │  │  │     └─ FullscreenMagnification:18:23#25
 │  │  │        └─ Leaf:18:23#26
 │  │  ├─ Leaf:24:25#27
 │  │  │  └─ WindowToken{65e11d9 type=2019 android.os.BinderProxy@fe89852}#73
 │  │  │     └─ Surface(name=47ff27f NavigationBar0)[...]ation-leash of insets_animation#6398
 │  │  │        └─ 47ff27f NavigationBar0#74
 │  │  │           └─ NavigationBar0#81
 │  │  └─ HideDisplayCutout:26:31#28
 │  │     └─ OneHanded:26:31#29
 │  │        ├─ FullscreenMagnification:26:27#30
 │  │        │  └─ Leaf:26:27#31
 │  │        ├─ Leaf:28:28#32
 │  │        └─ FullscreenMagnification:29:31#33
 │  │           └─ Leaf:29:31#34
 │  ├─ HideDisplayCutout:32:35#35
 │  │  ├─ OneHanded:32:32#36
 │  │  │  └─ Leaf:32:32#37
 │  │  ├─ FullscreenMagnification:33:33#38
 │  │  │  └─ Leaf:33:33#39
 │  │  └─ OneHanded:34:35#40
 │  │     └─ FullscreenMagnification:34:35#41
 │  │        └─ Leaf:34:35#42
 │  ├─ Leaf:36:36#43
 │  ├─ Accessibility Overlays#46
 │  ├─ Input Overlays#45
 │  │  └─ [Gesture Monitor] swipe-up#79
 │  └─ Display Overlays#44
 ├─ WindowToken{de6b26f type=2024 android.os.BinderProxy@27ef14e}#60
 │  └─ 5d6068b ScreenDecorOverlay#61
 │     └─ ScreenDecorOverlay#66
 └─ WindowToken{543080a type=2024 android.os.BinderProxy@3850075}#64
    └─ bb9a87b ScreenDecorOverlayBottom#65
       └─ ScreenDecorOverlayBottom#67
Offscreen Hierarchy
 ROOT
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0xe9f2399_transition-leash#6049
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0x6f4da8f_transition-leash#6107
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0x33494e8_transition-leash#6348
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0x924f67c_transition-leash#6383
 ├─ Surface(name=f0063f3 com.android.chr[...]mation-leash of starting_reveal#6365
 ├─ Surface(name=47ff27f NavigationBar0)[...]ation-leash of insets_animation#6378
 ├─ Surface(name=e9a2f44 StatusBar)/@0x9[...]ation-leash of insets_animation#6379
 ├─ Surface(name=734fecd com.google.andr[...]mation-leash of starting_reveal#6400
 ├─ (Relative) Input Consumer recents_animation_input_consumer#88
 ├─ Surface(name=Task=2118)/@0xc4d825_transition-leash#6336
 ├─ Surface(name=Task=1)/@0xab07a9d_transition-leash#6047
 ├─ Surface(name=Task=1)/@0x398e33_transition-leash#6105
 ├─ Surface(name=Task=1)/@0x3cc24fc_transition-leash#6346
 ├─ Surface(name=Task=1)/@0x70fd550_transition-leash#6381
 ├─ Surface(name=Task=2116)/@0x15ad2e3_transition-leash#6048
 ├─ Surface(name=Task=2117)/@0xd481769_transition-leash#6106
 ├─ Surface(name=Task=2118)/@0x4ed9da_transition-leash#6347
 └─ Surface(name=Task=2119)/@0xc4cb44e_transition-leash#6382

Display 4619827677550801152 (active) HWC layers:
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Layer name
           Z |  Window Type |  Comp Type |  Transform |   Disp Frame (LTRB) |          Source Crop (LTRB) |     Frame Rate (Explicit) (Seamlessness) [Focused]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 com.google.android.dialer/com.google[...]ensions.GoogleDialtactsActivity#6395
           5 |            1 |     DEVICE |          0 |    0    0 1080 2400 |    0.0    0.0 1080.0 2400.0 |                                              [*]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 StatusBar#85
           6 |         2000 |     DEVICE |          0 |    0    0 1080  128 |    0.0    0.0 1080.0  128.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 NavigationBar0#81
           7 |         2019 |     DEVICE |          0 |    0 2274 1080 2400 |    0.0    0.0 1080.0  126.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 ScreenDecorOverlay#66
           9 |         2024 |     DEVICE |          0 |    0    0 1080  128 |    0.0    0.0 1080.0  128.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 ScreenDecorOverlayBottom#67
          10 |         2024 |     DEVICE |          0 |    0 2326 1080 2400 |    0.0    0.0 1080.0   74.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------

Displays (1 entries)
Display 4619827677550801152
    connectionType=Internal
    colorModes=
        ColorMode::NATIVE
        ColorMode::SRGB
        ColorMode::DISPLAY_P3
    deviceProductInfo={name="Common Panel", manufacturerPnpId=GGL, productId=0, manufactureWeek=1, manufactureYear=1990, relativeAddress=[]}
    name="Common Panel"
    powerMode=On
    activeMode=60.00 Hz (60.00 Hz(60.00 Hz))
    displayModes=
        {id=0, hwcId=36, resolution=1080x2400, vsyncRate=60.00 Hz, dpi=409.43x411.89, group=0, vrrConfig=N/A}
        {id=1, hwcId=37, resolution=1080x2400, vsyncRate=90.00 Hz, dpi=409.43x411.89, group=0, vrrConfig=N/A}
    displayManagerPolicy={defaultModeId=1, allowGroupSwitching=false, primaryRanges={physical=[0.00 Hz, inf Hz], render=[0.00 Hz, 90.00 Hz]}, appRequestRanges={physical=[0.00 Hz, inf Hz], render=[0.00 Hz, 90.00 Hz]}}
    frameRateOverrideConfig=Enabled
    idleTimer=
        interval=1500.000 ms
        controller=Platform

Display 4619827677550801152 (physical, "Common Panel")
   Composition Display State:
   isEnabled=true isSecure=true usesDeviceComposition=true 
   usesClientComposition=false flipClientTarget=false reusedClientComposition=false 
   layerFilter={layerStack=0 toInternalDisplay=true }
   transform (ROT_0) (IDENTITY)
   layerStackSpace=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} 
   framebufferSpace=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} 
   orientedDisplaySpace=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} 
   displaySpace=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} 
   needsFiltering=false 
   colorMode=SRGB (7) renderIntent=ENHANCE (1) dataspace=V0_SRGB (142671872) 
   colorTransformMatrix=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] 
   displayBrightnessNits=-1.000000 sdrWhitePointNits=-1.000000 clientTargetBrightness=1.000000 displayBrightness=nullopt 
   compositionStrategyPredictionState=DISABLED 
   
   treat170mAsSrgb=true 


   Composition Display Color State:
   HWC Support: wideColorGamut=true hdr10plus=true hdr10=true hlg=true dv=false metadata=7 
   Hdr Luminance Info:desiredMinLuminance=0.000500 desiredMaxLuminance=800.000000 desiredMaxAverageLuminance=120.000000 

   Composition RenderSurface State:
   size=[1080 2400] ANativeWindow=0xb4000071870a99e0 (format 1) 
   FramebufferSurface
      mDataspace=Default (0)
      mAbandoned=0
      - BufferQueue mMaxAcquiredBufferCount=2 mMaxDequeuedBufferCount=1
        mDequeueBufferCannotBlock=0 mAsyncMode=0
        mQueueBufferCanDrop=0 mLegacyBufferDrop=1
        default-size=[1080x2400] default-format=1         transform-hint=00 frame-counter=928835
        mTransformHintInUse=00 mAutoPrerotation=0
      FIFO(0):
      (mConsumerName=FramebufferSurface, mConnectedApi=1, mConsumerUsageBits=6656, mId=1e900000000, producer=[489:/system/bin/surfaceflinger], consumer=[489:/system/bin/surfaceflinger])
      Slots:
       >[01:0xb40000708707a4d0] state=ACQUIRED 0xb4000070f706d170 frame=928835 [1080x2400:1088,  1]
        [00:0xb4000070870792d0] state=FREE     0xb4000070f70714d0 frame=928833 [1080x2400:1088,  1]
        [02:0xb40000708707a950] state=FREE     0xb4000070f7071210 frame=928834 [1080x2400:1088,  1]

   5 Layers
  - Output Layer 0xb4000071e71453f0(com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395)
        Region visibleRegion (this=0xb4000071e7145408, count=1)
    [  0,   0, 1080, 2400]
        Region visibleNonTransparentRegion (this=0xb4000071e7145470, count=1)
    [  0,   0, 1080, 2400]
        Region coveredRegion (this=0xb4000071e71454d8, count=2)
    [  0,   0, 1080, 128]
    [  0, 2274, 1080, 2400]
        Region output visibleRegion (this=0xb4000071e71455b0, count=1)
    [  0,   0, 1080, 2400]
        Region shadowRegion (this=0xb4000071e7145618, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e71456b0, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 0 1080 2400] sourceCrop=[0.000000 0.000000 1080.000000 2400.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e7145760, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e71457c8, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590ab640 composition=DEVICE (2) 
  - Output Layer 0xb4000071e7136c30(StatusBar#85)
        Region visibleRegion (this=0xb4000071e7136c48, count=1)
    [  0,   0, 1080, 128]
        Region visibleNonTransparentRegion (this=0xb4000071e7136cb0, count=1)
    [  0,   0, 1080, 128]
        Region coveredRegion (this=0xb4000071e7136d18, count=1)
    [  0,   0, 1080, 128]
        Region output visibleRegion (this=0xb4000071e7136df0, count=1)
    [  0,   0, 1080, 128]
        Region shadowRegion (this=0xb4000071e7136e58, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e7136ef0, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 0 1080 128] sourceCrop=[0.000000 0.000000 1080.000000 128.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e7136fa0, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e7137008, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590b1c70 composition=DEVICE (2) 
  - Output Layer 0xb4000071e715eff0(NavigationBar0#81)
        Region visibleRegion (this=0xb4000071e715f008, count=1)
    [  0, 2274, 1080, 2400]
        Region visibleNonTransparentRegion (this=0xb4000071e715f070, count=1)
    [  0, 2274, 1080, 2400]
        Region coveredRegion (this=0xb4000071e715f0d8, count=1)
    [  0, 2326, 1080, 2400]
        Region output visibleRegion (this=0xb4000071e715f1b0, count=1)
    [  0, 2274, 1080, 2400]
        Region shadowRegion (this=0xb4000071e715f218, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e715f2b0, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 2274 1080 2400] sourceCrop=[0.000000 0.000000 1080.000000 126.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e715f360, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e715f3c8, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590afa60 composition=DEVICE (2) 
  - Output Layer 0xb4000071e7131210(ScreenDecorOverlay#66)
        Region visibleRegion (this=0xb4000071e7131228, count=1)
    [  0,   0, 1080, 128]
        Region visibleNonTransparentRegion (this=0xb4000071e7131290, count=1)
    [  0,   0, 1080, 128]
        Region coveredRegion (this=0xb4000071e71312f8, count=1)
    [  0,   0,   0,   0]
        Region output visibleRegion (this=0xb4000071e71313d0, count=1)
    [  0,   0, 1080, 128]
        Region shadowRegion (this=0xb4000071e7131438, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e71314d0, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 0 1080 128] sourceCrop=[0.000000 0.000000 1080.000000 128.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e7131580, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e71315e8, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590ad850 composition=DEVICE (2) 
  - Output Layer 0xb4000071e712d1b0(ScreenDecorOverlayBottom#67)
        Region visibleRegion (this=0xb4000071e712d1c8, count=1)
    [  0, 2326, 1080, 2400]
        Region visibleNonTransparentRegion (this=0xb4000071e712d230, count=1)
    [  0, 2326, 1080, 2400]
        Region coveredRegion (this=0xb4000071e712d298, count=1)
    [  0,   0,   0,   0]
        Region output visibleRegion (this=0xb4000071e712d370, count=1)
    [  0, 2326, 1080, 2400]
        Region shadowRegion (this=0xb4000071e712d3d8, count=1)
    [  0,   0,   0,   0]
        Region outputSpaceBlockingRegionHint (this=0xb4000071e712d470, count=1)
    [  0,   0,   0,   0]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 2326 1080 2400] sourceCrop=[0.000000 0.000000 1080.000000 74.000000] bufferTransform=0 (0) dataspace=V0_SRGB (142671872) whitePointNits=-1.000000 dimmingRatio=1.000000 override buffer=0xb4000070e70b92f8 override acquire fence=0xb4000070b70a79b0 override display frame=[0 0 1080 2400] override dataspace=V0_SRGB (142671872) override display space=ProjectionSpace{bounds=Rect(0, 0, 1080, 2400), content=Rect(0, 0, 1080, 2400), orientation=ROTATION_0} override damage region=  Region  (this=0xb4000071e712d520, count=1)
    [  0,   0,  -1,  -1]
 override visible region=  Region  (this=0xb4000071e712d588, count=1)
    [  0,   0, 1080, 2400]
 override peekThroughLayer=0x0 override disableBackgroundBlur=false 
      hwc: layer=0x08b4000074590b6090 composition=DEVICE (2) 

```

### compositionLayers

> 输出如下

```shell
Composition list
LayerStack=0
  Layer [6395] com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395
    visible reason= buffer=21006685044739 frame=14 color{< 0, 0, 0, 1 >}
    bounds={0,0,2400,1080}
    input{(0x0) canOccludePresentation touchCropId=6385 touchableRegion={0,0,2400,1080}}
  Layer [85] StatusBar#85
    visible reason= buffer=8087423418396 frame=6396 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY) touchableRegion={0,0,128,1080}}
  Layer [81] NavigationBar0#81
    visible reason= buffer=8087423418382 frame=30832 color{< 0, 0, 0, 1 >}
    bounds={0,2274,2400,1080} toDisplayTransform={ tx=0.0000 ty=2274.0000 }
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | WATCH_OUTSIDE_TOUCH | SLIPPERY) touchableRegion={0,2274,2400,1080}}
  Layer [66] ScreenDecorOverlay#66
    visible reason= buffer=8087423418370 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={492,0,128,610}}
  Layer [67] ScreenDecorOverlayBottom#67
    visible reason= buffer=8087423418376 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,2326,2400,1080} toDisplayTransform={ tx=0.0000 ty=2326.0000 }
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={0,2326,2326,0}}

Input list
LayerStack=0
  Layer [67] ScreenDecorOverlayBottom#67
    visible reason= buffer=8087423418376 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,2326,2400,1080} toDisplayTransform={ tx=0.0000 ty=2326.0000 }
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={0,2326,2326,0}}
  Layer [66] ScreenDecorOverlay#66
    visible reason= buffer=8087423418370 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={492,0,128,610}}
  Layer [79] [Gesture Monitor] swipe-up#79
    invisible reason=nothing to draw
    bounds={-10800,-24000,24000,10800}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | SPY) replaceTouchableRegionWithCrop touchableRegion={-10799,-23999,24000,10800}}
  Layer [81] NavigationBar0#81
    visible reason= buffer=8087423418382 frame=30832 color{< 0, 0, 0, 1 >}
    bounds={0,2274,2400,1080} toDisplayTransform={ tx=0.0000 ty=2274.0000 }
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | WATCH_OUTSIDE_TOUCH | SLIPPERY) touchableRegion={0,2274,2400,1080}}
  Layer [85] StatusBar#85
    visible reason= buffer=8087423418396 frame=6396 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY) touchableRegion={0,0,128,1080}}
  Layer [6395] com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395
    visible reason= buffer=21006685044739 frame=14 color{< 0, 0, 0, 1 >}
    bounds={0,0,2400,1080}
    input{(0x0) canOccludePresentation touchCropId=6385 touchableRegion={0,0,2400,1080}}
  Layer [6393] df705d4 ActivityRecordInputSink com.google.android.dialer/com.android.dialer.main.impl.MainActivity#6393
    invisible reason=nothing to draw
    bounds={-10800,-24000,24000,10800}
    input{(NO_INPUT_CHANNEL | NOT_FOCUSABLE) replaceTouchableRegionWithCrop touchableRegion={-10799,-23999,24000,10800}}
  Layer [6357] 97c0a45 ActivityRecordInputSink com.android.chrome/org.chromium.chrome.browser.ChromeTabbedActivity#6357
    invisible reason=hidden by parent or layer flag
    bounds={675.364,2114.09,2261.09,822.364} toDisplayTransform={ scale x=0.1361 y=0.1361  tx=675.3639 ty=2114.0933 }
    input{(NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE) replaceTouchableRegionWithCrop touchableRegion={675,2114,2261,822}}
  Layer [6235] 75c2726 ActivityRecordInputSink com.kwai.video/com.yxcorp.gifshow.tiny.TinyLaunchActivity#6235
    invisible reason=hidden by parent or layer flag
    bounds={0,302,2702,1080} toDisplayTransform={ tx=0.0000 ty=302.0000 }
    input{(NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE) replaceTouchableRegionWithCrop touchableRegion={0,302,2702,1080}}
  Layer [93] b2c2147 ActivityRecordInputSink com.android.launcher3/.uioverrides.QuickstepLauncher#93
    invisible reason=hidden by parent or layer flag
    bounds={0,0,2400,1080}
    input{(NO_INPUT_CHANNEL | NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE) replaceTouchableRegionWithCrop touchableRegion={0,0,2400,1080}}
  Layer [70] com.android.systemui.wallpapers.ImageWallpaper#70
    invisible reason=hidden by parent or layer flag
    bounds={-10800,-24000,24000,10800} toDisplayTransform={ scale x=2.5782 y=2.5781  tx=-450.0000 ty=-120.0000 }
    input{(NOT_VISIBLE | NOT_FOCUSABLE | NOT_TOUCHABLE | PREVENT_SPLITTING | IS_WALLPAPER) touchableRegion={-449,-119,-119,-449}}

Layer Hierarchy
 ROOT
 ├─ Display 0 name="Built-in Screen"#3
 │  ├─ WindowedMagnification:0:31#4
 │  │  ├─ HideDisplayCutout:0:14#5
 │  │  │  └─ OneHanded:0:14#6
 │  │  │     ├─ FullscreenMagnification:0:12#7
 │  │  │     │  ├─ Leaf:0:1#8
 │  │  │     │  │  └─ WallpaperWindowToken{955eacd token=android.os.Binder@e0c7864}#68
 │  │  │     │  │     └─ e217400 com.android.systemui.wallpapers.ImageWallpaper#69
 │  │  │     │  │        └─ com.android.systemui.wallpapers.ImageWallpaper#70
 │  │  │     │  │           └─ Wallpaper BBQ wrapper#71
 │  │  │     │  ├─ DefaultTaskDisplayArea#9
 │  │  │     │  │  ├─ Task=1#47
 │  │  │     │  │  │  ├─ Task=2087#86
 │  │  │     │  │  │  │  └─ ActivityRecord{1ec2161 u0 com.androi[...]verrides.QuickstepLauncher t2087}#87
 │  │  │     │  │  │  │     ├─ b2c2147 ActivityRecordInputSink com.[...]r3/.uioverrides.QuickstepLauncher#93
 │  │  │     │  │  │  │     └─ e3afbbd com.android.launcher3/com.an[...]er3.uioverrides.QuickstepLauncher#89
 │  │  │     │  │  │  └─ home_task_overlay_container#48
 │  │  │     │  │  ├─ Task=2#49
 │  │  │     │  │  │  ├─ Task=3#50
 │  │  │     │  │  │  │  └─ Dim layer#54
 │  │  │     │  │  │  └─ Task=4#51
 │  │  │     │  │  │     └─ Dim layer#55
 │  │  │     │  │  ├─ Task=2118#6227
 │  │  │     │  │  │  └─ ActivityRecord{a38fa81 u0 com.kwai.v[...].tiny.TinyLaunchActivity t2118}#6228
 │  │  │     │  │  │     ├─ 75c2726 ActivityRecordInputSink com.[...]gifshow.tiny.TinyLaunchActivity#6235
 │  │  │     │  │  │     └─ d7a4455 com.kwai.video/com.yxcorp.gifshow.tiny.TinyLaunchActivity#6238
 │  │  │     │  │  │        └─ cce5cda Panel:com.kwai.video/com.yxcorp.gifshow.tiny.TinyLaunchActivity#6255
 │  │  │     │  │  ├─ Task=2119#6350
 │  │  │     │  │  │  └─ ActivityRecord{9b20abc u0 com.androi[...]android.apps.chrome.Main t2119}#6351
 │  │  │     │  │  │     ├─ 97c0a45 ActivityRecordInputSink com.[...]me.browser.ChromeTabbedActivity#6357
 │  │  │     │  │  │     └─ f0063f3 com.android.chrome/com.google.android.apps.chrome.Main#6361
 │  │  │     │  │  └─ Task=2120#6385
 │  │  │     │  │     └─ ActivityRecord{f7a8f27 u0 com.google[...].GoogleDialtactsActivity t2120}#6386
 │  │  │     │  │        ├─ df705d4 ActivityRecordInputSink com.[...]d.dialer.main.impl.MainActivity#6393
 │  │  │     │  │        ├─ 734fecd com.google.android.dialer/co[...]ensions.GoogleDialtactsActivity#6394
 │  │  │     │  │        │  ├─ com.google.android.dialer/com.google[...]ensions.GoogleDialtactsActivity#6395
 │  │  │     │  │        │  └─ (Relative) ImeContainer#12 parent=6386
 │  │  │     │  │        │     └─ WindowToken{66faee type=2011 android.os.Binder@76f2069}#5163
 │  │  │     │  │        │        └─ Surface(name=def733c InputMethod)/@0[...]ation-leash of insets_animation#6401
 │  │  │     │  │        │           └─ def733c InputMethod#5164
 │  │  │     │  └─ Leaf:3:12#10
 │  │  │     │     └─ WindowToken{594c626 type=2038 android.os.BinderProxy@d7779a2}#52
 │  │  │     │        └─ c8b57c ShellDropTarget#53
 │  │  │     └─ ImePlaceholder:13:14#11
 │  │  ├─ OneHanded:15:15#13
 │  │  │  └─ FullscreenMagnification:15:15#14
 │  │  │     └─ Leaf:15:15#15
 │  │  │        └─ WindowToken{8d3b857 type=2000 android.os.BinderProxy@aa77df1}#77
 │  │  │           └─ Surface(name=e9a2f44 StatusBar)/@0x9[...]ation-leash of insets_animation#6399
 │  │  │              └─ e9a2f44 StatusBar#78
 │  │  │                 └─ StatusBar#85
 │  │  ├─ HideDisplayCutout:16:16#16
 │  │  │  └─ OneHanded:16:16#17
 │  │  │     └─ FullscreenMagnification:16:16#18
 │  │  │        └─ Leaf:16:16#19
 │  │  ├─ OneHanded:17:17#20
 │  │  │  └─ FullscreenMagnification:17:17#21
 │  │  │     └─ Leaf:17:17#22
 │  │  │        └─ WindowToken{5f35f75 type=2040 android.os.BinderProxy@f689c5f}#75
 │  │  │           └─ 9337b0a NotificationShade#76
 │  │  ├─ HideDisplayCutout:18:23#23
 │  │  │  └─ OneHanded:18:23#24
 │  │  │     └─ FullscreenMagnification:18:23#25
 │  │  │        └─ Leaf:18:23#26
 │  │  ├─ Leaf:24:25#27
 │  │  │  └─ WindowToken{65e11d9 type=2019 android.os.BinderProxy@fe89852}#73
 │  │  │     └─ Surface(name=47ff27f NavigationBar0)[...]ation-leash of insets_animation#6398
 │  │  │        └─ 47ff27f NavigationBar0#74
 │  │  │           └─ NavigationBar0#81
 │  │  └─ HideDisplayCutout:26:31#28
 │  │     └─ OneHanded:26:31#29
 │  │        ├─ FullscreenMagnification:26:27#30
 │  │        │  └─ Leaf:26:27#31
 │  │        ├─ Leaf:28:28#32
 │  │        └─ FullscreenMagnification:29:31#33
 │  │           └─ Leaf:29:31#34
 │  ├─ HideDisplayCutout:32:35#35
 │  │  ├─ OneHanded:32:32#36
 │  │  │  └─ Leaf:32:32#37
 │  │  ├─ FullscreenMagnification:33:33#38
 │  │  │  └─ Leaf:33:33#39
 │  │  └─ OneHanded:34:35#40
 │  │     └─ FullscreenMagnification:34:35#41
 │  │        └─ Leaf:34:35#42
 │  ├─ Leaf:36:36#43
 │  ├─ Accessibility Overlays#46
 │  ├─ Input Overlays#45
 │  │  └─ [Gesture Monitor] swipe-up#79
 │  └─ Display Overlays#44
 ├─ WindowToken{de6b26f type=2024 android.os.BinderProxy@27ef14e}#60
 │  └─ 5d6068b ScreenDecorOverlay#61
 │     └─ ScreenDecorOverlay#66
 └─ WindowToken{543080a type=2024 android.os.BinderProxy@3850075}#64
    └─ bb9a87b ScreenDecorOverlayBottom#65
       └─ ScreenDecorOverlayBottom#67
Offscreen Hierarchy
 ROOT
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0xe9f2399_transition-leash#6049
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0x6f4da8f_transition-leash#6107
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0x33494e8_transition-leash#6348
 ├─ Surface(name=WallpaperWindowToken{95[...]4})/@0x924f67c_transition-leash#6383
 ├─ Surface(name=f0063f3 com.android.chr[...]mation-leash of starting_reveal#6365
 ├─ Surface(name=47ff27f NavigationBar0)[...]ation-leash of insets_animation#6378
 ├─ Surface(name=e9a2f44 StatusBar)/@0x9[...]ation-leash of insets_animation#6379
 ├─ Surface(name=734fecd com.google.andr[...]mation-leash of starting_reveal#6400
 ├─ (Relative) Input Consumer recents_animation_input_consumer#88
 ├─ Surface(name=Task=2118)/@0xc4d825_transition-leash#6336
 ├─ Surface(name=Task=1)/@0xab07a9d_transition-leash#6047
 ├─ Surface(name=Task=1)/@0x398e33_transition-leash#6105
 ├─ Surface(name=Task=1)/@0x3cc24fc_transition-leash#6346
 ├─ Surface(name=Task=1)/@0x70fd550_transition-leash#6381
 ├─ Surface(name=Task=2116)/@0x15ad2e3_transition-leash#6048
 ├─ Surface(name=Task=2117)/@0xd481769_transition-leash#6106
 ├─ Surface(name=Task=2118)/@0x4ed9da_transition-leash#6347
 └─ Surface(name=Task=2119)/@0xc4cb44e_transition-leash#6382

Display 4619827677550801152 (active) HWC layers:
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Layer name
           Z |  Window Type |  Comp Type |  Transform |   Disp Frame (LTRB) |          Source Crop (LTRB) |     Frame Rate (Explicit) (Seamlessness) [Focused]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 com.google.android.dialer/com.google[...]ensions.GoogleDialtactsActivity#6395
           5 |            1 |     DEVICE |          0 |    0    0 1080 2400 |    0.0    0.0 1080.0 2400.0 |                                              [*]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 StatusBar#85
           6 |         2000 |     DEVICE |          0 |    0    0 1080  128 |    0.0    0.0 1080.0  128.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 NavigationBar0#81
           7 |         2019 |     DEVICE |          0 |    0 2274 1080 2400 |    0.0    0.0 1080.0  126.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 ScreenDecorOverlay#66
           9 |         2024 |     DEVICE |          0 |    0    0 1080  128 |    0.0    0.0 1080.0  128.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 ScreenDecorOverlayBottom#67
          10 |         2024 |     DEVICE |          0 |    0 2326 1080 2400 |    0.0    0.0 1080.0   74.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
```


> 源码如下

```c++

	// Traversal of drawing state must happen on the main thread.
    // Otherwise, SortedVector may have shared ownership during concurrent
    // traversals, which can result in use-after-frees.
    std::string compositionLayers;
    mScheduler
            ->schedule([&]() FTL_FAKE_GUARD(mStateLock) FTL_FAKE_GUARD(kMainThreadContext) {
                dumpVisibleFrontEnd(compositionLayers);
            })
            .get();
```

```cpp
mScheduler
            ->schedule([&]() FTL_FAKE_GUARD(mStateLock) FTL_FAKE_GUARD(kMainThreadContext) {
                dumpVisibleFrontEnd(compositionLayers);
            })
            .get();
dumpAll(args, compositionLayers, result);
```

```c++
void SurfaceFlinger::dumpVisibleFrontEnd(std::string& result) {
    if (!mLayerLifecycleManagerEnabled) {
        StringAppendF(&result, "Composition layers\n");
        mDrawingState.traverseInZOrder([&](Layer* layer) {
            auto* compositionState = layer->getCompositionState();
            if (!compositionState || !compositionState->isVisible) return;
            android::base::StringAppendF(&result, "* Layer %p (%s)\n", layer,
                                         layer->getDebugName() ? layer->getDebugName()
                                                               : "<unknown>");
            compositionState->dump(result);
        });

        StringAppendF(&result, "Offscreen Layers\n");
        for (Layer* offscreenLayer : mOffscreenLayers) {
            offscreenLayer->traverse(LayerVector::StateSet::Drawing,
                                     [&](Layer* layer) { layer->dumpOffscreenDebugInfo(result); });
        }
    } else {
        std::ostringstream out;
        // 输出Composition List
        out << "\nComposition list\n";
        ui::LayerStack lastPrintedLayerStackHeader = ui::INVALID_LAYER_STACK;
        mLayerSnapshotBuilder.forEachVisibleSnapshot(
                [&](std::unique_ptr<frontend::LayerSnapshot>& snapshot) {
                    if (snapshot->hasSomethingToDraw()) {
                        if (lastPrintedLayerStackHeader != snapshot->outputFilter.layerStack) {
                            lastPrintedLayerStackHeader = snapshot->outputFilter.layerStack;
                            out << "LayerStack=" << lastPrintedLayerStackHeader.id << "\n";
                        }
                        out << "  " << *snapshot << "\n";
                    }
                });
		// 输出Input List
        out << "\nInput list\n";
        lastPrintedLayerStackHeader = ui::INVALID_LAYER_STACK;
        mLayerSnapshotBuilder.forEachInputSnapshot([&](const frontend::LayerSnapshot& snapshot) {
            if (lastPrintedLayerStackHeader != snapshot.outputFilter.layerStack) {
                lastPrintedLayerStackHeader = snapshot.outputFilter.layerStack;
                out << "LayerStack=" << lastPrintedLayerStackHeader.id << "\n";
            }
            out << "  " << snapshot << "\n";
        });
		// 输出Layer Hierarchy
        out << "\nLayer Hierarchy\n"
            << mLayerHierarchyBuilder.getHierarchy() << "\nOffscreen Hierarchy\n"
            << mLayerHierarchyBuilder.getOffscreenHierarchy() << "\n\n";
        result = out.str();
        dumpHwcLayersMinidump(result);
    }
}
```


> 总结下主要有如下

| 日志类型             | 详细信息                                       |
| ---------------- | ------------------------------------------ |
| Composition List |                                            |
| Input List       |                                            |
| Layer Hierarchy  | 1.Layer Hierarchy<br>2.Offscreen Hierarchy |
| HWC layers       |                                            |
|                  |                                            |

#### Composition List


```shell
Composition list
LayerStack=0
  Layer [6395] com.google.android.dialer/com.google.android.dialer.extensions.GoogleDialtactsActivity#6395
    visible reason= buffer=21006685044739 frame=14 color{< 0, 0, 0, 1 >}
    bounds={0,0,2400,1080}
    input{(0x0) canOccludePresentation touchCropId=6385 touchableRegion={0,0,2400,1080}}
  Layer [85] StatusBar#85
    visible reason= buffer=8087423418396 frame=6396 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY) touchableRegion={0,0,128,1080}}
  Layer [81] NavigationBar0#81
    visible reason= buffer=8087423418382 frame=30832 color{< 0, 0, 0, 1 >}
    bounds={0,2274,2400,1080} toDisplayTransform={ tx=0.0000 ty=2274.0000 }
    input{(NOT_FOCUSABLE | TRUSTED_OVERLAY | WATCH_OUTSIDE_TOUCH | SLIPPERY) touchableRegion={0,2274,2400,1080}}
  Layer [66] ScreenDecorOverlay#66
    visible reason= buffer=8087423418370 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,0,128,1080}
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={492,0,128,610}}
  Layer [67] ScreenDecorOverlayBottom#67
    visible reason= buffer=8087423418376 frame=87 color{< 0, 0, 0, 1 >}
    bounds={0,2326,2400,1080} toDisplayTransform={ tx=0.0000 ty=2326.0000 }
    input{(NOT_FOCUSABLE | NOT_TOUCHABLE | TRUSTED_OVERLAY | SLIPPERY) touchableRegion={0,2326,2326,0}}
```

#### Input List


#### LayerHierarchy


#### HWC layers



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
