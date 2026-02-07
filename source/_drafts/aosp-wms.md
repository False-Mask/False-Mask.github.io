---
title: aosp-wms
tags:
cover:
---





# WMS逻辑解析


WMS是WindowManagerService的缩写，他是Window的管理类，其中Window是一个图层对象。
包含应用层的Surface，WindowManagerService需要处理与SurfaceFlinger的通信 & Input逻辑的处理



# Window处理


## Window创建


- Window创建

 在Activity的attachBaseContext后会执行Activity.attach再次过程中会创建mWindow对象。堆栈如下：
```text
attach:8850, Activity (android.app)
performLaunchActivity:3944, ActivityThread (android.app)
handleLaunchActivity:4173, ActivityThread (android.app)
execute:114, LaunchActivityItem (android.app.servertransaction)
executeNonLifecycleItem:231, TransactionExecutor (android.app.servertransaction)
executeTransactionItems:152, TransactionExecutor (android.app.servertransaction)
execute:93, TransactionExecutor (android.app.servertransaction)
handleMessage:2595, ActivityThread$H (android.app)
dispatchMessage:107, Handler (android.os)
loopOnce:232, Looper (android.os)
loop:317, Looper (android.os)
run:67, KwaiExcludeUncaughtExceptionHandler$1 (com.yxcorp.gifshow.apm.crash)
handleCallback:959, Handler (android.os)
dispatchMessage:100, Handler (android.os)
loopOnce:232, Looper (android.os)
loop:317, Looper (android.os)
main:8592, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:580, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:878, ZygoteInit (com.android.internal.os)
```

attach执行逻辑如下
```java
public class Activity extends ContextThemeWrapper  
        implements LayoutInflater.Factory2,  
        Window.Callback, KeyEvent.Callback,  
        OnCreateContextMenuListener, ComponentCallbacks2,  
        Window.OnWindowDismissedCallback,  
        ContentCaptureManager.ContentCaptureClient {
        
	final void attach(Context context, ActivityThread aThread,  
        Instrumentation instr, IBinder token, int ident,  
        Application application, Intent intent, ActivityInfo info,  
        CharSequence title, Activity parent, String id,  
        NonConfigurationInstances lastNonConfigurationInstances,  
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,  
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,  
        IBinder shareableActivityToken, IBinder initialCallerInfoAccessToken) { 
        
        // ......
        
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(mWindowControllerCallback);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        
        // ......
	        
    }

}
```


## IWindowSession.addToDisplayAsUser


### App进程

- IWindowSession.addToDisplayAsUser
```text
setView:1476, ViewRootImpl (android.view)
addView:441, WindowManagerGlobal (android.view)
addView:158, WindowManagerImpl (android.view)
handleResumeActivity:5330, ActivityThread (android.app)
execute:57, ResumeActivityItem (android.app.servertransaction)
execute:60, ActivityTransactionItem (android.app.servertransaction)
executeLifecycleItem:282, TransactionExecutor (android.app.servertransaction)
executeTransactionItems:150, TransactionExecutor (android.app.servertransaction)
execute:93, TransactionExecutor (android.app.servertransaction)
handleMessage:2595, ActivityThread$H (android.app)
dispatchMessage:107, Handler (android.os)
loopOnce:232, Looper (android.os)
loop:317, Looper (android.os)
run:67, KwaiExcludeUncaughtExceptionHandler$1 (com.yxcorp.gifshow.apm.crash)
handleCallback:959, Handler (android.os)
dispatchMessage:100, Handler (android.os)
loopOnce:232, Looper (android.os)
loop:317, Looper (android.os)
main:8592, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:580, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:878, ZygoteInit (com.android.internal.os)
```
ViewRootImpl.setView通过调用Binder对象IWindowSession.addToDisplayAsUser方法与WindowManager进行IPC通信。最终将Window对象添加到WMS中
```java
public final class ViewRootImpl implements ViewParent,  
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks,  
        AttachedSurfaceControl {
        
	public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {  
		// ......        
    
	    res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId,
                            mInsetsController.getRequestedVisibleTypes(), inputChannel, mTempInsets,
                            mTempControls, attachedFrame, compatScale);
                            
        // ......
    }

}
```


### system server进程

- Session
Session类用于处理客户端的IPC请求。(可能是为了抽离部分公开的API吧)
```java
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {

	final WindowManagerService mService;

	@Override
    public int addToDisplayAsUser(IWindow window, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, int userId, @InsetsType int requestedVisibleTypes,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl.Array outActiveControls, Rect outAttachedFrame,
            float[] outSizeCompatScale) {
        return mService.addWindow(this, window, attrs, viewVisibility, displayId, userId,
                requestedVisibleTypes, outInputChannel, outInsetsState, outActiveControls,
                outAttachedFrame, outSizeCompatScale);
    }

}
```

- WindowManagerService
可以发现addWindow的逻辑非常的长，但是核心的逻辑其实不多。主要是：
第六步——创建WindowState
第九步——将WindowState添加到WMS中的Window Tree中

```java
public class WindowManagerService extends IWindowManager.Stub  
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
        
	
	public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
            int displayId, int requestUserId, @InsetsType int requestedVisibleTypes,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl.Array outActiveControls, Rect outAttachedFrame,
            float[] outSizeCompatScale) {
        outActiveControls.set(null);
        int[] appOp = new int[1];
        final boolean isRoundedCornerOverlay = (attrs.privateFlags
                & PRIVATE_FLAG_IS_ROUNDED_CORNERS_OVERLAY) != 0;
        int res = mPolicy.checkAddPermission(attrs.type, isRoundedCornerOverlay, attrs.packageName,
                appOp);
        if (res != ADD_OKAY) {
            return res;
        }

        WindowState parentWindow = null;
        final int callingUid = Binder.getCallingUid();
        final int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        final int type = attrs.type;

        synchronized (mGlobalLock) {
            // 异常逻辑检测
            // ......
            
            // 去重防止重复添加Window
            if (mWindowMap.containsKey(client.asBinder())) {
                ProtoLog.w(WM_ERROR, "Window %s is already added", client);
                return WindowManagerGlobal.ADD_DUPLICATE_ADD;
            }
			// 根据WindowManager.LayoutParams类型做处理操作。
			// 而我们普通activity的Type会在android.app.ActivityThread#handleResumeActivity中
			// 被赋值为WindowManager.LayoutParams.TYPE_BASE_APPLICATION也就是1
			
			// 1. 如果是子窗口即type属于1000 ～ 1999 确保parentwindow不为null、确保parentWindow类型不是子窗口。
			// ......

			// 2. 处理Window类型为TYPE_PRESENTATION(2037)、TYPE_PRIVATE_PRESENTATION(2030)
			// 的情况（进行callback回调、异常状态的处理）
            // ......


            ActivityRecord activity = null;
            final boolean hasParent = parentWindow != null;
            // 3. 从缓存中获取WindowToken对象
            // 其中当前window是sub window则使用parentWindow的token(Binder对象)获取WindowToken，否则使用自己的token获取WindowToken
            WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);
            // 4. 获取window type,逻辑类似#3
            final int rootType = hasParent ? parentWindow.mAttrs.type : type;

            boolean addToastWindowRequiresToken = false;

            final IBinder windowContextToken = attrs.mWindowContextToken;
            
            // 5. 针对与不同的case做兜底
			//   a. 若token为空创建token
			//   b. 若为APPLICATION_WINDOW的sub layers则检验parent
			//   c. egg......
            if (token == null) {
                if (!unprivilegedAppCanCreateTokenWith(parentWindow, callingUid, type,
                        rootType, attrs.token, attrs.packageName)) {
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (hasParent) {
                    // Use existing parent window token for child windows.
                    token = parentWindow.mToken;
                } else if (mWindowContextListenerController.hasListener(windowContextToken)) {
                    // Respect the window context token if the user provided it.
                    final IBinder binder = attrs.token != null ? attrs.token : windowContextToken;
                    final Bundle options = mWindowContextListenerController
                            .getOptions(windowContextToken);
                    token = new WindowToken.Builder(this, binder, type)
                            .setDisplayContent(displayContent)
                            .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                            .setRoundedCornerOverlay(isRoundedCornerOverlay)
                            .setFromClientToken(true)
                            .setOptions(options)
                            .build();
                } else {
                    final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
                    token = new WindowToken.Builder(this, binder, type)
                            .setDisplayContent(displayContent)
                            .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                            .setRoundedCornerOverlay(isRoundedCornerOverlay)
                            .build();
                }
            } else if (rootType >= FIRST_APPLICATION_WINDOW
                    && rootType <= LAST_APPLICATION_WINDOW) { // 处理
                activity = token.asActivityRecord();
                if (activity == null) {
                    ProtoLog.w(WM_ERROR, "Attempted to add window with non-application token "
                            + ".%s Aborting.", token);
                    return WindowManagerGlobal.ADD_NOT_APP_TOKEN;
                } else if (activity.getParent() == null) {
                    ProtoLog.w(WM_ERROR, "Attempted to add window with exiting application token "
                            + ".%s Aborting.", token);
                    return WindowManagerGlobal.ADD_APP_EXITING;
                } else if (type == TYPE_APPLICATION_STARTING) {
                    if (activity.mStartingWindow != null) {
                        ProtoLog.w(WM_ERROR, "Attempted to add starting window to "
                                + "token with already existing starting window");
                        return WindowManagerGlobal.ADD_DUPLICATE_ADD;
                    }
                    if (activity.mStartingData == null) {
                        ProtoLog.w(WM_ERROR, "Attempted to add starting window to "
                                + "token but already cleaned");
                        return WindowManagerGlobal.ADD_DUPLICATE_ADD;
                    }
                }
            } else if (rootType == TYPE_INPUT_METHOD) {
                if (token.windowType != TYPE_INPUT_METHOD) {
                    ProtoLog.w(WM_ERROR, "Attempted to add input method window with bad token "
                            + "%s.  Aborting.", attrs.token);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType == TYPE_VOICE_INTERACTION) {
                if (token.windowType != TYPE_VOICE_INTERACTION) {
                    ProtoLog.w(WM_ERROR, "Attempted to add voice interaction window with bad token "
                            + "%s.  Aborting.", attrs.token);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType == TYPE_WALLPAPER) {
                if (token.windowType != TYPE_WALLPAPER) {
                    ProtoLog.w(WM_ERROR, "Attempted to add wallpaper window with bad token "
                            + "%s.  Aborting.", attrs.token);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (rootType == TYPE_ACCESSIBILITY_OVERLAY) {
                if (token.windowType != TYPE_ACCESSIBILITY_OVERLAY) {
                    ProtoLog.w(WM_ERROR,
                            "Attempted to add Accessibility overlay window with bad token "
                                    + "%s.  Aborting.", attrs.token);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_TOAST) {
                // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                addToastWindowRequiresToken = doesAddToastWindowRequireToken(attrs.packageName,
                        callingUid, parentWindow);
                if (addToastWindowRequiresToken && token.windowType != TYPE_TOAST) {
                    ProtoLog.w(WM_ERROR, "Attempted to add a toast window with bad token "
                            + "%s.  Aborting.", attrs.token);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_QS_DIALOG) {
                if (token.windowType != TYPE_QS_DIALOG) {
                    ProtoLog.w(WM_ERROR, "Attempted to add QS dialog window with bad token "
                            + "%s.  Aborting.", attrs.token);
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (token.asActivityRecord() != null) {
                ProtoLog.w(WM_ERROR, "Non-null activity for system window of rootType=%d",
                        rootType);
                // It is not valid to use an app token with other system types; we will
                // instead make a new token for it (as if null had been passed in for the token).
                attrs.token = null;
                token = new WindowToken.Builder(this, client.asBinder(), type)
                        .setDisplayContent(displayContent)
                        .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                        .build();
            }
            
            
            // 6. 创建WindowState
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], attrs, viewVisibility, session.mUid, userId,
                    session.mCanAddInternalSystemWindow);
            final DisplayPolicy displayPolicy = displayContent.getDisplayPolicy();
            displayPolicy.adjustWindowParamsLw(win, win.mAttrs);
            attrs.flags = sanitizeFlagSlippery(attrs.flags, win.getName(), callingUid, callingPid);
            attrs.inputFeatures = sanitizeSpyWindow(attrs.inputFeatures, win.getName(), callingUid,
                    callingPid);
            win.setRequestedVisibleTypes(requestedVisibleTypes);

            res = displayPolicy.validateAddingWindowLw(attrs, callingPid, callingUid);
            
            // 中途出现异常直接return
            if (res != ADD_OKAY) {
                return res;
            }

			// 是否启用input(大多数情况应该都会开启。)
            final boolean openInputChannels = (outInputChannel != null
                    && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
            if  (openInputChannels) {
                win.openInputChannel(outInputChannel);
            }

            // 7. 处理toast的消失逻辑，post一条delay消息进行隐藏
            if (type == TYPE_TOAST) {
                if (!displayContent.canAddToastWindowForUid(callingUid)) {
                    ProtoLog.w(WM_ERROR, "Adding more than one toast window for UID at a time.");
                    return WindowManagerGlobal.ADD_DUPLICATE_ADD;
                }
                // Make sure this happens before we moved focus as one can make the
                // toast focusable to force it not being hidden after the timeout.
                // Focusable toasts are always timed out to prevent a focused app to
                // show a focusable toasts while it has focus which will be kept on
                // the screen after the activity goes away.
                if (addToastWindowRequiresToken
                        || (attrs.flags & FLAG_NOT_FOCUSABLE) == 0
                        || displayContent.mCurrentFocus == null
                        || displayContent.mCurrentFocus.mOwnerUid != callingUid) {
                    mH.sendMessageDelayed(
                            mH.obtainMessage(H.WINDOW_HIDE_TIMEOUT, win),
                            win.mAttrs.hideTimeoutMilliseconds);
                }
            }

		    // ......

			// 8. 将windowState添加到一个WMS的map中去，于此同时更新windowState的部分字段
            win.mSession.onWindowAdded(win);
            mWindowMap.put(client.asBinder(), win);
            win.initAppOpsState();

            final boolean suspended = mPmInternal.isPackageSuspended(win.getOwningPackage(),
                    UserHandle.getUserId(win.getOwningUid()));
            win.setHiddenWhileSuspended(suspended);

            final boolean hideSystemAlertWindows = !mHidingNonSystemOverlayWindows.isEmpty();
            win.setForceHideNonSystemOverlayWindowIfNeeded(hideSystemAlertWindows);

            boolean imMayMove = true;
			// 9. 将WindowState添加到WMS中的Window Tree中
            win.mToken.addWindow(win);
            displayPolicy.addWindowLw(win, attrs);
            displayPolicy.setDropInputModePolicy(win, win.mAttrs);
            
            // 10. 针对于TYPE_APPLICATION_STARTING、TYPE_INPUT_METHOD、TYPE_INPUT_METHOD_DIALOG
            // TYPE_WALLPAPER进行处理。（普通activity等逻辑不会走入这里的分支）
            if (type == TYPE_APPLICATION_STARTING && activity != null) {
                activity.attachStartingWindow(win);
                ProtoLog.v(WM_DEBUG_STARTING_WINDOW, "addWindow: %s startingWindow=%s",
                        activity, win);
            } else if (type == TYPE_INPUT_METHOD
                    // IME window is always touchable.
                    // Ignore non-touchable windows e.g. Stylus InkWindow.java.
                    && (win.getAttrs().flags & FLAG_NOT_TOUCHABLE) == 0) {
                displayContent.setInputMethodWindowLocked(win);
                imMayMove = false;
            } else if (type == TYPE_INPUT_METHOD_DIALOG) {
                displayContent.computeImeTarget(true /* updateImeTarget */);
                imMayMove = false;
            } else {
                if (type == TYPE_WALLPAPER) {
                    displayContent.mWallpaperController.clearLastWallpaperTimeoutTime();
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                } else if (win.hasWallpaper()) {
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                } else if (displayContent.mWallpaperController.isBelowWallpaperTarget(win)) {
                    // If there is currently a wallpaper being shown, and
                    // the base layer of the new window is below the current
                    // layer of the target window, then adjust the wallpaper.
                    // This is to avoid a new window being placed between the
                    // wallpaper and its target.
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                }
            }

            // ......
        }


        return res;
    }

}
```


其中win.mToken.addWindow中逻辑如下：
核心是初始化mSurfaceControl并建立父子关系。
```java
class WindowToken extends WindowContainer<WindowState> {

	  void addWindow(final WindowState win) {
        ProtoLog.d(WM_DEBUG_FOCUS,
                "addWindow: win=%s Callers=%s", win, Debug.getCallers(5));

        if (win.isChildWindow()) {
            // Child windows are added to their parent windows.
            return;
        }
        // This token is created from WindowContext and the client requests to addView now, create a
        // surface for this token.
        // 如果当前Window未关联SurfaceControl则创建新的surfaceControl(在此过程中会和parentWindow建立父子关系)
        if (mSurfaceControl == null) {
            createSurfaceControl(true /* force */);

            // Layers could have been assigned before the surface was created, update them again
            reassignLayer(getSyncTransaction());
        }
        if (!mChildren.contains(win)) {
            ProtoLog.v(WM_DEBUG_ADD_REMOVE, "Adding %s to %s", win, this);
            addChild(win, mWindowComparator);
            mWmService.mWindowsChanged = true;
            // TODO: Should we also be setting layout needed here and other places?
        }
    }


}
```

DefaultTaskDisplayArea 
	->  ActivityRecord{XXX package/activity}
		 ->  Window{XXX package/activity}

```text
onParentChanged:665, WindowContainer (com.android.server.wm)
onParentChanged:1246, WindowState (com.android.server.wm)
setParent:647, WindowContainer (com.android.server.wm)
addChild:778, WindowContainer (com.android.server.wm)
addWindow:322, WindowToken (com.android.server.wm)
addWindow:4579, ActivityRecord (com.android.server.wm)
addWindow:1763, WindowManagerService (com.android.server.wm)
addToDisplayAsUser:255, Session (com.android.server.wm)
onTransact:645, IWindowSession$Stub (android.view)
onTransact:204, Session (com.android.server.wm)
execTransactInternal:1500, Binder (android.os)
execTransact:1444, Binder (android.os)
```


### relayoutWindow

```text
getSurfaceControl:201, WindowSurfaceController (com.android.server.wm)
createSurfaceControl:2665, WindowManagerService (com.android.server.wm)
relayoutWindow:2413, WindowManagerService (com.android.server.wm)
relayout:290, Session (com.android.server.wm)
onTransact:725, IWindowSession$Stub (android.view)
onTransact:204, Session (com.android.server.wm)
execTransactInternal:1500, Binder (android.os)
execTransact:1444, Binder (android.os)
```

relayoutWindow中有大学问：
1.relayoutWindow以前SurfaceControl只是一个空壳。
2.在app调用relayoutWindow的时候应用进程会把SurfaceControl传递到WMS进程
3.WMS进程会创建一个新的SurfaceControl,这个SurfaceControl是addWindow中创建的SurfaceControl的child
4.WMS会修改应用进程的SurfaceControl对象
5.WMS进程执行完毕后回掉App进程，App进程会将新的SurfaceControl作为入参更新BLASTBufferQueue(Buffer Layer All by Transitions) 最终通过BLASTBufferQueue创建Surface
6.到此整个relayoutWindow结束，Window等一系列准备操作结束。


### Window分类

前面在分析的过程中提到了addWindow有很多跟Window.type相关的逻辑，这里简单说明下Window的分类。
在 Android 窗口系统中，窗口类型分为三大类：

1. **应用窗口（Application Window）**​
    
    - **范围**：`FIRST_APPLICATION_WINDOW (1)`~ `LAST_APPLICATION_WINDOW (99)`
        
    - **示例**：Activity 的顶层窗口（`TYPE_BASE_APPLICATION`）、对话框（`TYPE_APPLICATION_PANEL`）等。
        
    
2. **子窗口（Sub-Window）**​
    
    - **范围**：`FIRST_SUB_WINDOW (1000)`~ `LAST_SUB_WINDOW (1999)`
        
    - **示例**：`PopupWindow`（`TYPE_APPLICATION_PANEL`）、媒体播放器控件（`TYPE_APPLICATION_MEDIA`）等。
        
    - **特点**：必须依附于父窗口（如 Activity 的根窗口），不能独立存在。
        
    
3. **系统窗口（System Window）**​
    
    - **范围**：`FIRST_SYSTEM_WINDOW (2000)`~ `LAST_SYSTEM_WINDOW (2999)`
        
    - **示例**：状态栏（`TYPE_STATUS_BAR`）、Toast（`TYPE_TOAST`）等。


## Window通信

- 相互通信

app侧：
ViewRootImpl#mWindowSession

WMS侧：
使用WMS通过ViewRootImpl#mWindow进行向应用层的ViewRootImpl.W发送信息


# Input处理



# WMS & SurfaceFlinger交互


WMS向SurfaceFlinger传递数据
WindowManagerService -> DisplayContent -> SurfaceSession -> SurfaceFlinger





# refs



[掘金专栏-WMS](https://juejin.cn/column/7339827208086929444)

[【Android 13源码分析】WindowContainer窗口层级-1-初识窗口层级树](https://juejin.cn/post/7339827208086896676)

[【Android 源码分析】Activity短暂的一生 -- 目录篇 （持续更新）](https://juejin.cn/post/7345105816242552871)
