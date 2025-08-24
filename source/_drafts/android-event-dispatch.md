---
title: android-event-dispatch
tags:
cover:
---



# Android 事件分发



# 内核驱动



1.输入设备

/dev/input/eventX

2.EventHub

通过epoll监听事件变化。读取原始事件





# 系统服务



1.InputReader

2.InputDisatcher



# 应用进程接受调度事件



这里简单给个结论，后续如果有时间记得回头更新下，

应用进程对于不同类型的事件处理方式不一样，对于DOWN,UP事件，系统高优先级处理，对于MOVE事件系统通过合并post INPUT_CALLBACK在ViewRootImpl中执行调度。https://juejin.cn/post/7224777406916902973



## way1

通过从NativeInputEventReceiver::consumeEvents

往下分发到android.view.InputEventReceiver#dispatchInputEvent



### 注册



当Activity onResume回调执行完毕后，会将View设置到contentView中去即setView，而该过程中会做一些初始化的操作。

> 堆栈如下

```
<init>:79, InputEventReceiver (android.view)
<init>:9304, ViewRootImpl$WindowInputEventReceiver (android.view)
setView:1401, ViewRootImpl (android.view)
addView:405, WindowManagerGlobal (android.view)
addView:148, WindowManagerImpl (android.view)
handleResumeActivity:4859, ActivityThread (android.app)
execute:57, ResumeActivityItem (android.app.servertransaction)
execute:45, ActivityTransactionItem (android.app.servertransaction)
executeLifecycleState:179, TransactionExecutor (android.app.servertransaction)
execute:97, TransactionExecutor (android.app.servertransaction)
handleMessage:2306, ActivityThread$H (android.app)
dispatchMessage:106, Handler (android.os)
loopOnce:201, Looper (android.os)
loop:288, Looper (android.os)
main:7918, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:548, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:936, ZygoteInit (com.android.internal.os)
```



- setView

内容过多只贴关注的部分

``` java
 public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
  
     
     if (inputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
         		// 创建inputEventReceiver 
                mInputEventReceiver = new WindowInputEventReceiver(inputChannel,
                        Looper.myLooper());

                if (ENABLE_INPUT_LATENCY_TRACKING && mAttachInfo.mThreadedRenderer != null) {
                    InputMetricsListener listener = new InputMetricsListener();
                    mHardwareRendererObserver = new HardwareRendererObserver(
                            listener, listener.data, mHandler, true /*waitForPresentTime*/);
                    mAttachInfo.mThreadedRenderer.addObserver(mHardwareRendererObserver);
                }
                // Update unbuffered request when set the root view.
                mUnbufferedInputSource = mView.mUnbufferedInputSource;
            }
     
     
 }
```



- EventReceiver创建

``` java
final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            // 回调父类
            super(inputChannel, looper);
        }

}

public abstract class InputEventReceiver {

   public InputEventReceiver(InputChannel inputChannel, Looper looper) {
       // 参数验证，非空判断
        if (inputChannel == null) {
            throw new IllegalArgumentException("inputChannel must not be null");
        }
        if (looper == null) {
            throw new IllegalArgumentException("looper must not be null");
        }
		// 全局变量初始化
        mInputChannel = inputChannel;
        mMessageQueue = looper.getQueue();
       // JNI创建，注意传入了this对象，inputChannel,messageQueue
        mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this),
                inputChannel, mMessageQueue);

        mCloseGuard.open("InputEventReceiver.dispose");
    }
    
}
```



- nativeInit



```c++
// 1.通过对JNI代码的分析不难发现，nativeInit绑定的实现为 nativeInit

static const JNINativeMethod gMethods[] = {
        /* name, signature, funcPtr */
    	// nativeInit绑定实现为  nativeInit
        {"nativeInit",
         "(Ljava/lang/ref/WeakReference;Landroid/view/InputChannel;Landroid/os/MessageQueue;)J",
         (void*)nativeInit},
        {"nativeDispose", "(J)V", (void*)nativeDispose},
        {"nativeFinishInputEvent", "(JIZ)V", (void*)nativeFinishInputEvent},
        {"nativeReportTimeline", "(JIJJ)V", (void*)nativeReportTimeline},
        {"nativeConsumeBatchedInputEvents", "(JJ)Z", (void*)nativeConsumeBatchedInputEvents},
        {"nativeDump", "(JLjava/lang/String;)Ljava/lang/String;", (void*)nativeDump},
};

int register_android_view_InputEventReceiver(JNIEnv* env) {
    int res = RegisterMethodsOrDie(env, "android/view/InputEventReceiver",
            gMethods, NELEM(gMethods));

    jclass clazz = FindClassOrDie(env, "android/view/InputEventReceiver");
    gInputEventReceiverClassInfo.clazz = MakeGlobalRefOrDie(env, clazz);

    gInputEventReceiverClassInfo.dispatchInputEvent = GetMethodIDOrDie(env,
            gInputEventReceiverClassInfo.clazz,
            "dispatchInputEvent", "(ILandroid/view/InputEvent;)V");
    gInputEventReceiverClassInfo.onFocusEvent =
            GetMethodIDOrDie(env, gInputEventReceiverClassInfo.clazz, "onFocusEvent", "(Z)V");
    gInputEventReceiverClassInfo.onPointerCaptureEvent =
            GetMethodIDOrDie(env, gInputEventReceiverClassInfo.clazz, "onPointerCaptureEvent",
                             "(Z)V");
    gInputEventReceiverClassInfo.onDragEvent =
            GetMethodIDOrDie(env, gInputEventReceiverClassInfo.clazz, "onDragEvent", "(ZFF)V");
    gInputEventReceiverClassInfo.onTouchModeChanged =
            GetMethodIDOrDie(env, gInputEventReceiverClassInfo.clazz, "onTouchModeChanged", "(Z)V");
    gInputEventReceiverClassInfo.onBatchedInputEventPending =
            GetMethodIDOrDie(env, gInputEventReceiverClassInfo.clazz, "onBatchedInputEventPending",
                             "(I)V");

    return res;
}


static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject inputChannelObj, jobject messageQueueObj) {
    // 1.获取inputChannel
    std::shared_ptr<InputChannel> inputChannel =
            android_view_InputChannel_getInputChannel(env, inputChannelObj);
    // 判空
    if (inputChannel == nullptr) {
        jniThrowRuntimeException(env, "InputChannel is not initialized.");
        return 0;
    }
	// 2.获取messageQueue
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    // 判空
    if (messageQueue == nullptr) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }
	// 3.创建NativeInputEventReceiver
    sp<NativeInputEventReceiver> receiver = new NativeInputEventReceiver(env,
            receiverWeak, inputChannel, messageQueue);
    // 4.初始化，核心操作。将当前的callback注册到native looper的 epool中
    // 具体可见下方setFdEvents
    status_t status = receiver->initialize();
    if (status) {
        std::string message = android::base::
                StringPrintf("Failed to initialize input event receiver.  status=%s(%d)",
                             statusToString(status).c_str(), status);
        jniThrowRuntimeException(env, message.c_str());
        return 0;
    }

    receiver->incStrong(gInputEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}


void NativeInputEventReceiver::setFdEvents(int events) {
    if (mFdEvents != events) {
        mFdEvents = events;
        int fd = mInputConsumer.getChannel()->getFd();
        if (events) {
            // 注意这里的callback是this,也就是说，只要有native事件传递下来就会回调到
            mMessageQueue->getLooper()->addFd(fd, 0, events, this, nullptr);
        } else {
            mMessageQueue->getLooper()->removeFd(fd);
        }
    }
}

// Callback回调处理。
int NativeInputEventReceiver::handleEvent(int receiveFd, int events, void* data) {
    // Allowed return values of this function as documented in LooperCallback::handleEvent
    constexpr int REMOVE_CALLBACK = 0;
    constexpr int KEEP_CALLBACK = 1;

    if (events & (ALOOPER_EVENT_ERROR | ALOOPER_EVENT_HANGUP)) {
        // This error typically occurs when the publisher has closed the input channel
        // as part of removing a window or finishing an IME session, in which case
        // the consumer will soon be disposed as well.
        if (kDebugDispatchCycle) {
            ALOGD("channel '%s' ~ Publisher closed input channel or an error occurred. events=0x%x",
                  getInputChannelName().c_str(), events);
        }
        return REMOVE_CALLBACK;
    }

    if (events & ALOOPER_EVENT_INPUT) {
        JNIEnv* env = AndroidRuntime::getJNIEnv();
        status_t status = consumeEvents(env, false /*consumeBatches*/, -1, nullptr);
        mMessageQueue->raiseAndClearException(env, "handleReceiveCallback");
        return status == OK || status == NO_MEMORY ? KEEP_CALLBACK : REMOVE_CALLBACK;
    }

    if (events & ALOOPER_EVENT_OUTPUT) {
        const status_t status = processOutboundEvents();
        if (status == OK || status == WOULD_BLOCK) {
            return KEEP_CALLBACK;
        } else {
            return REMOVE_CALLBACK;
        }
    }

    ALOGW("channel '%s' ~ Received spurious callback for unhandled poll event.  events=0x%x",
          getInputChannelName().c_str(), events);
    return KEEP_CALLBACK;
}
```

