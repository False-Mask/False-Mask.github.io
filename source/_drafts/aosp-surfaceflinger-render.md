---
title: aosp-surfaceflinger-render
tags:
cover:
---

# SurfaceFlinger Render渲染逻辑


# App侧更新


## App创建Surface

### 外部调用处
// TODO
需要详细的调用堆栈，App 调用到createSurface的完整路径

```c++
tring8 name("displaydemo");
sp<SurfaceControl> surfaceControl = 
            surfaceComposerClient->createSurface(name, resolution.getWidth(), 
                                                    resolution.getHeight(), PIXEL_FORMAT_RGBA_8888,
                                                    ISurfaceComposerClient::eFXSurfaceBufferState,/*parent*/ nullptr);

```

### SurfaceComposerClient::createSurface实现原理

```c++
sp<SurfaceControl> SurfaceComposerClient::createSurface(const String8& name, uint32_t w, uint32_t h,
                                                        PixelFormat format, int32_t flags,
                                                        const sp<IBinder>& parentHandle,
                                                        LayerMetadata metadata,
                                                        uint32_t* outTransformHint) {
    sp<SurfaceControl> s;
    createSurfaceChecked(name, w, h, format, &s, flags, parentHandle, std::move(metadata),
                         outTransformHint);
    return s;
}
```


### SurfaceComposerClient::createSurfaceChecked


```c++
status_t SurfaceComposerClient::createSurfaceChecked(const String8& name, uint32_t w, uint32_t h,
                                                     PixelFormat format,
                                                     sp<SurfaceControl>* outSurface, int32_t flags,
                                                     const sp<IBinder>& parentHandle,
                                                     LayerMetadata metadata,
                                                     uint32_t* outTransformHint) {
    sp<SurfaceControl> sur;
    status_t err = mStatus;

    if (mStatus == NO_ERROR) {
        gui::CreateSurfaceResult result;
        // 1. 通过 binder 创建 Surface 对象
        binder::Status status = mClient->createSurface(std::string(name.c_str()), flags,
                                                       parentHandle, std::move(metadata), &result);
        err = statusTFromBinderStatus(status);
        if (outTransformHint) {
            *outTransformHint = result.transformHint;
        }
        ALOGE_IF(err, "SurfaceComposerClient::createSurface error %s", strerror(-err));
        // 2. 创建Surface 对象并返回
        if (err == NO_ERROR) {
            *outSurface = new SurfaceControl(this, result.handle, result.layerId,
                                             toString(result.layerName), w, h, format,
                                             result.transformHint, flags);
        }
    }
    return err;
}
```

- createSurface
```c++
binder::Status Client::createSurface(const std::string& name, int32_t flags,
                                     const sp<IBinder>& parent, const gui::LayerMetadata& metadata,
                                     gui::CreateSurfaceResult* outResult) {
    // We rely on createLayer to check permissions.
    sp<IBinder> handle;
    //  1. 准备createLayer的函数参数
    LayerCreationArgs args(mFlinger.get(), sp<Client>::fromExisting(this), name.c_str(),
                           static_cast<uint32_t>(flags), std::move(metadata));
    args.parentHandle = parent;
    // 2. 调用createLayer进程 Layer 的创建等操作
    const status_t status = mFlinger->createLayer(args, *outResult);
    return binderStatusFromStatusT(status);
}
```

```C++
status_t SurfaceFlinger::createLayer(LayerCreationArgs& args, gui::CreateSurfaceResult& outResult) {
    status_t result = NO_ERROR;

    sp<Layer> layer;

    switch (args.flags & ISurfaceComposerClient::eFXSurfaceMask) {
	    // 如果 flag 中包含如下标记则附带eNoColorFill标记。不 break 代码执行流程
        case ISurfaceComposerClient::eFXSurfaceBufferQueue:
        case ISurfaceComposerClient::eFXSurfaceContainer:
        case ISurfaceComposerClient::eFXSurfaceBufferState:
            args.flags |= ISurfaceComposerClient::eNoColorFill;
            [[fallthrough]];
        // flag 中包含eFXSurfaceEffect标记则创建 Layer
        case ISurfaceComposerClient::eFXSurfaceEffect: {
            // 创建 Layer 对象实例
            result = createBufferStateLayer(args, &outResult.handle, &layer);
            std::atomic<int32_t>* pendingBufferCounter = layer->getPendingBufferCounter();
            if (pendingBufferCounter) {
                std::string counterName = layer->getPendingBufferCounterName();
                mBufferCountTracker.add(outResult.handle->localBinder(), counterName,
                                        pendingBufferCounter);
            }
        } break;
        default:
            result = BAD_VALUE;
            break;
    }

    if (result != NO_ERROR) {
        return result;
    }

    args.addToRoot = args.addToRoot && callingThreadHasUnscopedSurfaceFlingerAccess();
    // We can safely promote the parent layer in binder thread because we have a strong reference
    // to the layer's handle inside this scope.
    sp<Layer> parent = LayerHandle::getLayer(args.parentHandle.promote());
    if (args.parentHandle != nullptr && parent == nullptr) {
        ALOGE("Invalid parent handle %p", args.parentHandle.promote().get());
        args.addToRoot = false;
    }

    uint32_t outTransformHint;
    // 将 layer 实例添加到 SurfaceFlinger 中
    result = addClientLayer(args, outResult.handle, layer, parent, &outTransformHint);
    if (result != NO_ERROR) {
        return result;
    }

    outResult.transformHint = static_cast<int32_t>(outTransformHint);
    outResult.layerId = layer->sequence;
    outResult.layerName = String16(layer->getDebugName());
    return result;
}
```


layer 创建逻辑，将 layer 添加到mCreatedLayers、mNewLayers、mNewLayerArgs中去，后续合成过程会调用SurfaceFlinger::updateLayerSnapshots将这三个参数用来构建 Layer 树。
```c++
status_t SurfaceFlinger::addClientLayer(LayerCreationArgs& args, const sp<IBinder>& handle,
                                        const sp<Layer>& layer, const wp<Layer>& parent,
                                        uint32_t* outTransformHint) {
    if (mNumLayers >= MAX_LAYERS) {
        // ......
    }

    layer->updateTransformHint(mActiveDisplayTransformHint);
    if (outTransformHint) {
        *outTransformHint = mActiveDisplayTransformHint;
    }
    args.parentId = LayerHandle::getLayerId(args.parentHandle.promote());
    args.layerIdToMirror = LayerHandle::getLayerId(args.mirrorLayerHandle.promote());
    {
	    // 将 layer 添加到mCreatedLayers、mNewLayers、mNewLayerArgs中去
        std::scoped_lock<std::mutex> lock(mCreatedLayersLock);
        mCreatedLayers.emplace_back(layer, parent, args.addToRoot);
        mNewLayers.emplace_back(std::make_unique<frontend::RequestedLayerState>(args));
        args.mirrorLayerHandle.clear();
        args.parentHandle.clear();
        mNewLayerArgs.emplace_back(std::move(args));
    }

    setTransactionFlags(eTransactionNeeded);
    return NO_ERROR;
}
```


## App发起事务操作


### 外部调用

//TODO 需要给出详细的调用处

```c++
// 构建事务对象并提交
SurfaceComposerClient::Transaction{}
		.setLayer(surfaceControl, std::numeric_limits<int32_t>::max())
		.show(surfaceControl)
		.setBackgroundColor(surfaceControl, half3{0, 0, 0}, 1.0f, ui::Dataspace::UNKNOWN) // black background
		.setAlpha(surfaceControl, 1.0f)
		.setLayerStack(surfaceControl, ui::LayerStack::fromValue(mLayerStack))
		.apply();
```

### Transaction::setXXX

Transaction::setLayer是一个案例，Transaction.setXXX的逻辑具有高度的相似性，逻辑如下：
1. 通过getLayerState创建/获取layer_state_t
2. 将函数参数字段设置到layer_state_t中。
```C++
SurfaceComposerClient::Transaction& SurfaceComposerClient::Transaction::setLayer(
        const sp<SurfaceControl>& sc, int32_t z) {
	// 1. 获取并创建layer_state_t
    layer_state_t* s = getLayerState(sc);
    if (!s) {
        mStatus = BAD_INDEX;
        return *this;
    }
    // 2. 设置layer_state_t中状态字段
    s->what |= layer_state_t::eLayerChanged;
    s->what &= ~layer_state_t::eRelativeLayerChanged;
    s->z = z;

    registerSurfaceControlForCallback(sc);
    return *this;
}

// 获取Transaction中的layer_state_t，执行流程如下：
// 1. 先从mComposerStates(map类型数据结构)中获取ComposerState(key 为SurfaceControl.getLayerId)，若无则创建ComposerState
// 2. 然后从ComposerState中获取layer_state_t并返回
layer_state_t* SurfaceComposerClient::Transaction::getLayerState(const sp<SurfaceControl>& sc) {
    auto handle = sc->getLayerStateHandle();

    if (mComposerStates.count(handle) == 0) {
        // we don't have it, add an initialized layer_state to our list
        ComposerState s;

        s.state.surface = handle;
        s.state.layerId = sc->getLayerId();

        mComposerStates[handle] = s;
    }

    return &(mComposerStates[handle].state);
}
```


### Transaction::apply

核心就是进行数据的准备操作（核心数据为ComposerState）最后通过远程调用到 SurfaceFlinger::setTransactionState 中
```c++
status_t SurfaceComposerClient::Transaction::apply(bool synchronous, bool oneWay) {
    if (mStatus != NO_ERROR) {
        return mStatus;
    }

    std::shared_ptr<SyncCallback> syncCallback = std::make_shared<SyncCallback>();
    if (synchronous) {
        syncCallback->init();
        addTransactionCommittedCallback(SyncCallback::getCallback(syncCallback),
                                        /*callbackContext=*/nullptr);
    }

	//  1. 同步SurfaceFlinger 需要的状态信息

    bool hasListenerCallbacks = !mListenerCallbacks.empty();
    std::vector<ListenerCallbacks> listenerCallbacks;
    // For every listener with registered callbacks
    // 遍历所有注册的 callbacks
    for (const auto& [listener, callbackInfo] : mListenerCallbacks) {
        auto& [callbackIds, surfaceControls] = callbackInfo;
        if (callbackIds.empty()) {
            continue;
        }
		// 全局callbacks
        if (surfaceControls.empty()) {
            listenerCallbacks.emplace_back(IInterface::asBinder(listener), std::move(callbackIds));
        } else {
            // If the listener has any SurfaceControls set on this Transaction update the surface
            // state
            // listener关联了 SurfaceControls
            for (const auto& surfaceControl : surfaceControls) {
                layer_state_t* s = getLayerState(surfaceControl);
                if (!s) {
                    ALOGE("failed to get layer state");
                    continue;
                }
                std::vector<CallbackId> callbacks(callbackIds.begin(), callbackIds.end());
                s->what |= layer_state_t::eHasListenerCallbacksChanged;
                s->listeners.emplace_back(IInterface::asBinder(listener), callbacks);
            }
        }
    }

    cacheBuffers();

    Vector<ComposerState> composerStates;
    Vector<DisplayState> displayStates;
    uint32_t flags = 0;

    for (auto const& kv : mComposerStates) {
        composerStates.add(kv.second);
    }

    displayStates = std::move(mDisplayStates);

    if (mAnimation) {
        flags |= ISurfaceComposer::eAnimation;
    }
    if (oneWay) {
        if (synchronous) {
            ALOGE("Transaction attempted to set synchronous and one way at the same time"
                  " this is an invalid request. Synchronous will win for safety");
        } else {
            flags |= ISurfaceComposer::eOneWay;
        }
    }

    // If both mEarlyWakeupStart and mEarlyWakeupEnd are set
    // it is equivalent for none
    if (mEarlyWakeupStart && !mEarlyWakeupEnd) {
        flags |= ISurfaceComposer::eEarlyWakeupStart;
    }
    if (mEarlyWakeupEnd && !mEarlyWakeupStart) {
        flags |= ISurfaceComposer::eEarlyWakeupEnd;
    }

    sp<IBinder> applyToken = mApplyToken ? mApplyToken : getDefaultApplyToken();

    sp<ISurfaceComposer> sf(ComposerService::getComposerService());

	// 2. 通过远程调用设置状态
    sf->setTransactionState(mFrameTimelineInfo, composerStates, displayStates, flags, applyToken,
                            mInputWindowCommands, mDesiredPresentTime, mIsAutoTimestamp,
                            mUncacheBuffers, hasListenerCallbacks, listenerCallbacks, mId,
                            mMergedTransactionIds);
    mId = generateId();

    // Clear the current states and flags
    clear();

    if (synchronous) {
        syncCallback->wait();
    }

    mStatus = NO_ERROR;
    return NO_ERROR;
}
```


###  SurfaceFlinger::setTransactionState

该方法首先会进行TransactionState字段的准备/创建，然后会通过mTransactionHandler.queueTransaction将TransactionState入队，最后通过setTransactionFlags方法触发入队事务的执行操作

```c++
status_t SurfaceFlinger::setTransactionState(
        const FrameTimelineInfo& frameTimelineInfo, Vector<ComposerState>& states,
        const Vector<DisplayState>& displays, uint32_t flags, const sp<IBinder>& applyToken,
        InputWindowCommands inputWindowCommands, int64_t desiredPresentTime, bool isAutoTimestamp,
        const std::vector<client_cache_t>& uncacheBuffers, bool hasListenerCallbacks,
        const std::vector<ListenerCallbacks>& listenerCallbacks, uint64_t transactionId,
        const std::vector<uint64_t>& mergedTransactionIds) {
    ATRACE_CALL();

    IPCThreadState* ipc = IPCThreadState::self();
    const int originPid = ipc->getCallingPid();
    const int originUid = ipc->getCallingUid();
    uint32_t permissions = LayerStatePermissions::getTransactionPermissions(originPid, originUid);
    for (auto& composerState : states) {
        composerState.state.sanitize(permissions);
    }

    for (DisplayState display : displays) {
        display.sanitize(permissions);
    }

    if (!inputWindowCommands.empty() &&
        (permissions & layer_state_t::Permission::ACCESS_SURFACE_FLINGER) == 0) {
        ALOGE("Only privileged callers are allowed to send input commands.");
        inputWindowCommands.clear();
    }

    if (flags & (eEarlyWakeupStart | eEarlyWakeupEnd)) {
        const bool hasPermission =
                (permissions & layer_state_t::Permission::ACCESS_SURFACE_FLINGER) ||
                callingThreadHasPermission(sWakeupSurfaceFlinger);
        if (!hasPermission) {
            ALOGE("Caller needs permission android.permission.WAKEUP_SURFACE_FLINGER to use "
                  "eEarlyWakeup[Start|End] flags");
            flags &= ~(eEarlyWakeupStart | eEarlyWakeupEnd);
        }
    }

	// TransactionState参数准备
	// 其中准备字段如下：

	// 提交时间
    const int64_t postTime = systemTime();
	// 未缓存的 BufferId
    std::vector<uint64_t> uncacheBufferIds;
    uncacheBufferIds.reserve(uncacheBuffers.size());
    for (const auto& uncacheBuffer : uncacheBuffers) {
        sp<GraphicBuffer> buffer = ClientCache::getInstance().erase(uncacheBuffer);
        if (buffer != nullptr) {
            uncacheBufferIds.push_back(buffer->getId());
        }
    }
	// 遍历states 创建/设置 resolvedStates 字段。
    std::vector<ResolvedComposerState> resolvedStates;
    resolvedStates.reserve(states.size());
    for (auto& state : states) {
        resolvedStates.emplace_back(std::move(state));
        auto& resolvedState = resolvedStates.back();
        if (resolvedState.state.hasBufferChanges() && resolvedState.state.hasValidBuffer() &&
            resolvedState.state.surface) {
            sp<Layer> layer = LayerHandle::getLayer(resolvedState.state.surface);
            std::string layerName = (layer) ?
                    layer->getDebugName() : std::to_string(resolvedState.state.layerId);
            resolvedState.externalTexture =
                    getExternalTextureFromBufferData(*resolvedState.state.bufferData,
                                                     layerName.c_str(), transactionId);
            if (resolvedState.externalTexture) {
                resolvedState.state.bufferData->buffer = resolvedState.externalTexture->getBuffer();
            }
            mBufferCountTracker.increment(resolvedState.state.surface->localBinder());
        }
        resolvedState.layerId = LayerHandle::getLayerId(resolvedState.state.surface);
        if (resolvedState.state.what & layer_state_t::eReparent) {
            resolvedState.parentId =
                    getLayerIdFromSurfaceControl(resolvedState.state.parentSurfaceControlForChild);
        }
        if (resolvedState.state.what & layer_state_t::eRelativeLayerChanged) {
            resolvedState.relativeParentId =
                    getLayerIdFromSurfaceControl(resolvedState.state.relativeLayerSurfaceControl);
        }
        if (resolvedState.state.what & layer_state_t::eInputInfoChanged) {
            wp<IBinder>& touchableRegionCropHandle =
                    resolvedState.state.windowInfoHandle->editInfo()->touchableRegionCropHandle;
            resolvedState.touchCropId =
                    LayerHandle::getLayerId(touchableRegionCropHandle.promote());
        }
    }

	// 构建最终的 TransactionState 字段
    TransactionState state{frameTimelineInfo,
                           resolvedStates,
                           displays,
                           flags,
                           applyToken,
                           std::move(inputWindowCommands),
                           desiredPresentTime,
                           isAutoTimestamp,
                           std::move(uncacheBufferIds),
                           postTime,
                           hasListenerCallbacks,
                           listenerCallbacks,
                           originPid,
                           originUid,
                           transactionId,
                           mergedTransactionIds};

    if (mTransactionTracing) {
        mTransactionTracing->addQueuedTransaction(state);
    }

    const auto schedule = [](uint32_t flags) {
        if (flags & eEarlyWakeupEnd) return TransactionSchedule::EarlyEnd;
        if (flags & eEarlyWakeupStart) return TransactionSchedule::EarlyStart;
        return TransactionSchedule::Late;
    }(state.flags);

    const auto frameHint = state.isFrameActive() ? FrameHint::kActive : FrameHint::kNone;
    // 入队提交TransactionState状态
    mTransactionHandler.queueTransaction(std::move(state));
    //  Vsync 同步通知：确保显示时序准确
    for (const auto& [displayId, data] : mNotifyExpectedPresentMap) {
        if (data.hintStatus.load() == NotifyExpectedPresentHintStatus::ScheduleOnTx) {
            scheduleNotifyExpectedPresentHint(displayId, VsyncId{frameTimelineInfo.vsyncId});
        }
    }
    // 唤醒SurfaceFlinger::handleTransaction，开始处理 mTransactionHandler队列中的事务。
    setTransactionFlags(eTransactionFlushNeeded, schedule, applyToken, frameHint);
    return NO_ERROR;
}
```


## App提交渲染Buffer


App 提交的渲染 Buffer 会经过如下流程被处理：
1. App 通过 queueBuffer 提交的所有的 buffer 最后会提交到 BLASTBufferQueue 中
2. BLASTBufferQueue 的消费者为SurfaceFlinger，其中消费者会通过调用acquireNextBufferLocked方法取出 BufferItem
3. SurfaceFlinger 创建 Transaction 将 BufferItem 的信息填充到 Transaction 中，最后通过 Transaction 提交到 SurfaceFlinger 中(同“App 发起事务操作”流程)

```c++
status_t BLASTBufferQueue::acquireNextBufferLocked(
        const std::optional<SurfaceComposerClient::Transaction*> transaction) {
    // Check if we have frames available and we have not acquired the maximum number of buffers.
    // Even with this check, the consumer can fail to acquire an additional buffer if the consumer
    // has already acquired (mMaxAcquiredBuffers + 1) and the new buffer is not droppable. In this
    // case mBufferItemConsumer->acquireBuffer will return with NO_BUFFER_AVAILABLE.
    if (mNumFrameAvailable == 0) {
        BQA_LOGV("Can't acquire next buffer. No available frames");
        return BufferQueue::NO_BUFFER_AVAILABLE;
    }

    if (mNumAcquired >= (mMaxAcquiredBuffers + 2)) {
        BQA_LOGV("Can't acquire next buffer. Already acquired max frames %d max:%d + 2",
                 mNumAcquired, mMaxAcquiredBuffers);
        return BufferQueue::NO_BUFFER_AVAILABLE;
    }

    if (mSurfaceControl == nullptr) {
        BQA_LOGE("ERROR : surface control is null");
        return NAME_NOT_FOUND;
    }

	// 创建 Transaction 对象
    SurfaceComposerClient::Transaction localTransaction;
    bool applyTransaction = true;
    SurfaceComposerClient::Transaction* t = &localTransaction;
    if (transaction) {
        t = *transaction;
        applyTransaction = false;
    }

    BufferItem bufferItem;
	// 通过 BufferQueue 的 acquireBuffer 接口获取 bufferItem 对象
    status_t status =
            mBufferItemConsumer->acquireBuffer(&bufferItem, 0 /* expectedPresent */, false);
    if (status == BufferQueue::NO_BUFFER_AVAILABLE) {
        BQA_LOGV("Failed to acquire a buffer, err=NO_BUFFER_AVAILABLE");
        return status;
    } else if (status != OK) {
        BQA_LOGE("Failed to acquire a buffer, err=%s", statusToString(status).c_str());
        return status;
    }

    auto buffer = bufferItem.mGraphicBuffer;
    mNumFrameAvailable--;
    BBQ_TRACE("frame=%" PRIu64, bufferItem.mFrameNumber);

    if (buffer == nullptr) {
        mBufferItemConsumer->releaseBuffer(bufferItem, Fence::NO_FENCE);
        BQA_LOGE("Buffer was empty");
        return BAD_VALUE;
    }

    if (rejectBuffer(bufferItem)) {
        BQA_LOGE("rejecting buffer:active_size=%dx%d, requested_size=%dx%d "
                 "buffer{size=%dx%d transform=%d}",
                 mSize.width, mSize.height, mRequestedSize.width, mRequestedSize.height,
                 buffer->getWidth(), buffer->getHeight(), bufferItem.mTransform);
        mBufferItemConsumer->releaseBuffer(bufferItem, Fence::NO_FENCE);
        return acquireNextBufferLocked(transaction);
    }

    mNumAcquired++;
    mLastAcquiredFrameNumber = bufferItem.mFrameNumber;
    ReleaseCallbackId releaseCallbackId(buffer->getId(), mLastAcquiredFrameNumber);
    mSubmitted[releaseCallbackId] = bufferItem;

    bool needsDisconnect = false;
    mBufferItemConsumer->getConnectionEvents(bufferItem.mFrameNumber, &needsDisconnect);

    // if producer disconnected before, notify SurfaceFlinger
    if (needsDisconnect) {
        t->notifyProducerDisconnect(mSurfaceControl);
    }

    // Ensure BLASTBufferQueue stays alive until we receive the transaction complete callback.
    incStrong((void*)transactionCallbackThunk);

    // Only update mSize for destination bounds if the incoming buffer matches the requested size.
    // Otherwise, it could cause stretching since the destination bounds will update before the
    // buffer with the new size is acquired.
    if (mRequestedSize == getBufferSize(bufferItem) ||
        bufferItem.mScalingMode != NATIVE_WINDOW_SCALING_MODE_FREEZE) {
        mSize = mRequestedSize;
    }
    Rect crop = computeCrop(bufferItem);
    mLastBufferInfo.update(true /* hasBuffer */, bufferItem.mGraphicBuffer->getWidth(),
                           bufferItem.mGraphicBuffer->getHeight(), bufferItem.mTransform,
                           bufferItem.mScalingMode, crop);

    auto releaseBufferCallback =
            std::bind(releaseBufferCallbackThunk, wp<BLASTBufferQueue>(this) /* callbackContext */,
                      std::placeholders::_1, std::placeholders::_2, std::placeholders::_3);
    // 填充Transaction 对象                  
    sp<Fence> fence = bufferItem.mFence ? new Fence(bufferItem.mFence->dup()) : Fence::NO_FENCE;
    t->setBuffer(mSurfaceControl, buffer, fence, bufferItem.mFrameNumber, mProducerId,
                 releaseBufferCallback);
    t->setDataspace(mSurfaceControl, static_cast<ui::Dataspace>(bufferItem.mDataSpace));
    t->setHdrMetadata(mSurfaceControl, bufferItem.mHdrMetadata);
    t->setSurfaceDamageRegion(mSurfaceControl, bufferItem.mSurfaceDamage);
    t->addTransactionCompletedCallback(transactionCallbackThunk, static_cast<void*>(this));

    mSurfaceControlsWithPendingCallback.push(mSurfaceControl);

    if (mUpdateDestinationFrame) {
        t->setDestinationFrame(mSurfaceControl, Rect(mSize));
    } else {
        const bool ignoreDestinationFrame =
                bufferItem.mScalingMode == NATIVE_WINDOW_SCALING_MODE_FREEZE;
        t->setFlags(mSurfaceControl,
                    ignoreDestinationFrame ? layer_state_t::eIgnoreDestinationFrame : 0,
                    layer_state_t::eIgnoreDestinationFrame);
    }
    t->setBufferCrop(mSurfaceControl, crop);
    t->setTransform(mSurfaceControl, bufferItem.mTransform);
    t->setTransformToDisplayInverse(mSurfaceControl, bufferItem.mTransformToDisplayInverse);
    t->setAutoRefresh(mSurfaceControl, bufferItem.mAutoRefresh);
    if (!bufferItem.mIsAutoTimestamp) {
        t->setDesiredPresentTime(bufferItem.mTimestamp);
    }

    // Drop stale frame timeline infos
    while (!mPendingFrameTimelines.empty() &&
           mPendingFrameTimelines.front().first < bufferItem.mFrameNumber) {
        ATRACE_FORMAT_INSTANT("dropping stale frameNumber: %" PRIu64 " vsyncId: %" PRId64,
                              mPendingFrameTimelines.front().first,
                              mPendingFrameTimelines.front().second.vsyncId);
        mPendingFrameTimelines.pop();
    }

    if (!mPendingFrameTimelines.empty() &&
        mPendingFrameTimelines.front().first == bufferItem.mFrameNumber) {
        ATRACE_FORMAT_INSTANT("Transaction::setFrameTimelineInfo frameNumber: %" PRIu64
                              " vsyncId: %" PRId64,
                              bufferItem.mFrameNumber,
                              mPendingFrameTimelines.front().second.vsyncId);
        t->setFrameTimelineInfo(mPendingFrameTimelines.front().second);
        mPendingFrameTimelines.pop();
    }

    {
        std::lock_guard _lock{mTimestampMutex};
        auto dequeueTime = mDequeueTimestamps.find(buffer->getId());
        if (dequeueTime != mDequeueTimestamps.end()) {
            Parcel p;
            p.writeInt64(dequeueTime->second);
            t->setMetadata(mSurfaceControl, gui::METADATA_DEQUEUE_TIME, p);
            mDequeueTimestamps.erase(dequeueTime);
        }
    }

    mergePendingTransactions(t, bufferItem.mFrameNumber);
    if (applyTransaction) {
		// 调用 Transaction.apply应用
        // All transactions on our apply token are one-way. See comment on mAppliedLastTransaction
        t->setApplyToken(mApplyToken).apply(false, true);
        mAppliedLastTransaction = true;
        mLastAppliedFrameNumber = bufferItem.mFrameNumber;
    } else {
        t->setBufferHasBarrier(mSurfaceControl, mLastAppliedFrameNumber);
        mAppliedLastTransaction = false;
    }

    BQA_LOGV("acquireNextBufferLocked size=%dx%d mFrameNumber=%" PRIu64
             " applyTransaction=%s mTimestamp=%" PRId64 "%s mPendingTransactions.size=%d"
             " graphicBufferId=%" PRIu64 "%s transform=%d",
             mSize.width, mSize.height, bufferItem.mFrameNumber, boolToString(applyTransaction),
             bufferItem.mTimestamp, bufferItem.mIsAutoTimestamp ? "(auto)" : "",
             static_cast<uint32_t>(mPendingTransactions.size()), bufferItem.mGraphicBuffer->getId(),
             bufferItem.mAutoRefresh ? " mAutoRefresh" : "", bufferItem.mTransform);
    return OK;
}
```






# SurfaceFlinger轮询渲染

SurfaceFlinger 是由Looper 驱动的因此定期会进行轮询执行 commit、composite 操作进行图像的合成
```c++
void Scheduler::onFrameSignal(ICompositor& compositor, VsyncId vsyncId,
                              TimePoint expectedVsyncTime) {
    const FrameTargeter::BeginFrameArgs beginFrameArgs =
            {.frameBeginTime = SchedulerClock::now(),
             .vsyncId = vsyncId,
             .expectedVsyncTime = expectedVsyncTime,
             .sfWorkDuration = mVsyncModulator->getVsyncConfig().sfWorkDuration,
             .hwcMinWorkDuration = mVsyncConfiguration->getCurrentConfigs().hwcMinWorkDuration};

    ftl::NonNull<const Display*> pacesetterPtr = pacesetterPtrLocked();
    pacesetterPtr->targeterPtr->beginFrame(beginFrameArgs, *pacesetterPtr->schedulePtr);

    {
        FrameTargets targets;
        targets.try_emplace(pacesetterPtr->displayId, &pacesetterPtr->targeterPtr->target());

        // TODO (b/256196556): Followers should use the next VSYNC after the frontrunner, not the
        // pacesetter.
        // Update expectedVsyncTime, which may have been adjusted by beginFrame.
        expectedVsyncTime = pacesetterPtr->targeterPtr->target().expectedPresentTime();

        for (const auto& [id, display] : mDisplays) {
            if (id == pacesetterPtr->displayId) continue;

            auto followerBeginFrameArgs = beginFrameArgs;
            followerBeginFrameArgs.expectedVsyncTime =
                    display.schedulePtr->vsyncDeadlineAfter(expectedVsyncTime);

            FrameTargeter& targeter = *display.targeterPtr;
            targeter.beginFrame(followerBeginFrameArgs, *display.schedulePtr);
            targets.try_emplace(id, &targeter.target());
        }

        if (!compositor.commit(pacesetterPtr->displayId, targets)) {
            if (FlagManager::getInstance().vrr_config()) {
                compositor.sendNotifyExpectedPresentHint(pacesetterPtr->displayId);
            }
            return;
        }
    }

    // The pacesetter may have changed or been registered anew during commit.
    pacesetterPtr = pacesetterPtrLocked();

    // TODO(b/256196556): Choose the frontrunner display.
    FrameTargeters targeters;
    targeters.try_emplace(pacesetterPtr->displayId, pacesetterPtr->targeterPtr.get());

    for (auto& [id, display] : mDisplays) {
        if (id == pacesetterPtr->displayId) continue;

        FrameTargeter& targeter = *display.targeterPtr;
        targeters.try_emplace(id, &targeter);
    }

    if (FlagManager::getInstance().vrr_config() &&
        CC_UNLIKELY(mPacesetterFrameDurationFractionToSkip > 0.f)) {
        const auto period = pacesetterPtr->targeterPtr->target().expectedFrameDuration();
        const auto skipDuration = Duration::fromNs(
                static_cast<nsecs_t>(period.ns() * mPacesetterFrameDurationFractionToSkip));
        ATRACE_FORMAT("Injecting jank for %f%% of the frame (%" PRId64 " ns)",
                      mPacesetterFrameDurationFractionToSkip * 100, skipDuration.ns());
        std::this_thread::sleep_for(skipDuration);
        mPacesetterFrameDurationFractionToSkip = 0.f;
    }

    if (FlagManager::getInstance().vrr_config()) {
        const auto minFramePeriod = pacesetterPtr->schedulePtr->minFramePeriod();
        const auto presentFenceForPastVsync =
                pacesetterPtr->targeterPtr->target().presentFenceForPastVsync(minFramePeriod);
        const auto lastConfirmedPresentTime = presentFenceForPastVsync->getSignalTime();
        if (lastConfirmedPresentTime != Fence::SIGNAL_TIME_PENDING &&
            lastConfirmedPresentTime != Fence::SIGNAL_TIME_INVALID) {
            pacesetterPtr->schedulePtr->getTracker()
                    .onFrameBegin(expectedVsyncTime, TimePoint::fromNs(lastConfirmedPresentTime));
        }
    }

    const auto resultsPerDisplay = compositor.composite(pacesetterPtr->displayId, targeters);
    if (FlagManager::getInstance().vrr_config()) {
        compositor.sendNotifyExpectedPresentHint(pacesetterPtr->displayId);
    }
    compositor.sample();

    for (const auto& [id, targeter] : targeters) {
        const auto resultOpt = resultsPerDisplay.get(id);
        LOG_ALWAYS_FATAL_IF(!resultOpt);
        targeter->endFrame(*resultOpt);
    }
}
```

## SurfaceFlinger::commit

在执行 commit 以前，SurfaceFlinger 渲染所需要的所有的数据已经通过IPC通信的方式保存到了SurfaceFlinger 中

- App 创建 Surface
```c++
class SurfaceFlinger : public BnSurfaceComposer,
                       public PriorityDumper,
                       private IBinder::DeathRecipient,
                       private HWC2::ComposerCallback,
                       private ICompositor,
                       private scheduler::ISchedulerCallback,
                       private compositionengine::ICEPowerCallback {
public:

	// ......
	
	// A temporay pool that store the created layers and will be added to current state in main
    // thread.
    std::vector<LayerCreatedState> mCreatedLayers GUARDED_BY(mCreatedLayersLock);

	// ......

	std::vector<std::unique_ptr<frontend::RequestedLayerState>> mNewLayers;

	// ......
	std::vector<LayerCreationArgs> mNewLayerArgs;

}
```

- App 事务/App 渲染 Buffer
```c++
class TransactionHandler {
public:

	// ......
	LocklessQueue<TransactionState> mLocklessTransactionQueue;

}
```

commit 的核心逻辑分为如下两个步骤
- flushLifecycleUpdates 将所有需要的数据进行整合合并到 Update 结构体中
- updateLayerSnapshots 传入所有数据即Update 结构体，更新 LayerSnapshot 的层级关系
\
```c++
bool SurfaceFlinger::commit(PhysicalDisplayId pacesetterId,
                            const scheduler::FrameTargets& frameTargets) {
    const scheduler::FrameTarget& pacesetterFrameTarget = *frameTargets.get(pacesetterId)->get();

    const VsyncId vsyncId = pacesetterFrameTarget.vsyncId();
    ATRACE_NAME(ftl::Concat(__func__, ' ', ftl::to_underlying(vsyncId)).c_str());

    if (pacesetterFrameTarget.didMissFrame()) {
        mTimeStats->incrementMissedFrames();
    }

    // If a mode set is pending and the fence hasn't fired yet, wait for the next commit.
    if (std::any_of(frameTargets.begin(), frameTargets.end(),
                    [this](const auto& pair) FTL_FAKE_GUARD(mStateLock)
                            FTL_FAKE_GUARD(kMainThreadContext) {
                                if (!pair.second->isFramePending()) return false;

                                if (const auto display = getDisplayDeviceLocked(pair.first)) {
                                    return display->isModeSetPending();
                                }

                                return false;
                            })) {
        mScheduler->scheduleFrame();
        return false;
    }

    {
        Mutex::Autolock lock(mStateLock);

        for (const auto [id, target] : frameTargets) {
            // TODO(b/241285876): This is `nullptr` when the DisplayDevice is about to be removed in
            // this commit, since the PhysicalDisplay has already been removed. Rather than checking
            // for `nullptr` below, change Scheduler::onFrameSignal to filter out the FrameTarget of
            // the removed display.
            const auto display = getDisplayDeviceLocked(id);

            if (display && display->isModeSetPending()) {
                finalizeDisplayModeChange(*display);
            }
        }
    }

    if (pacesetterFrameTarget.isFramePending()) {
        if (mBackpressureGpuComposition || pacesetterFrameTarget.didMissHwcFrame()) {
            if (FlagManager::getInstance().vrr_config()) {
                mScheduler->getVsyncSchedule()->getTracker().onFrameMissed(
                        pacesetterFrameTarget.expectedPresentTime());
            }
            scheduleCommit(FrameHint::kNone);
            return false;
        }
    }

    const Period vsyncPeriod = mScheduler->getVsyncSchedule()->period();

    // Save this once per commit + composite to ensure consistency
    // TODO (b/240619471): consider removing active display check once AOD is fixed
    const auto activeDisplay = FTL_FAKE_GUARD(mStateLock, getDisplayDeviceLocked(mActiveDisplayId));
    mPowerHintSessionEnabled = mPowerAdvisor->usePowerHintSession() && activeDisplay &&
            activeDisplay->getPowerMode() == hal::PowerMode::ON;
    if (mPowerHintSessionEnabled) {
        mPowerAdvisor->setCommitStart(pacesetterFrameTarget.frameBeginTime());
        mPowerAdvisor->setExpectedPresentTime(pacesetterFrameTarget.expectedPresentTime());

        // Frame delay is how long we should have minus how long we actually have.
        const Duration idealSfWorkDuration =
                mScheduler->vsyncModulator().getVsyncConfig().sfWorkDuration;
        const Duration frameDelay =
                idealSfWorkDuration - pacesetterFrameTarget.expectedFrameDuration();

        mPowerAdvisor->setFrameDelay(frameDelay);
        mPowerAdvisor->setTotalFrameTargetWorkDuration(idealSfWorkDuration);

        const auto& display = FTL_FAKE_GUARD(mStateLock, getDefaultDisplayDeviceLocked()).get();
        const Period idealVsyncPeriod = display->getActiveMode().fps.getPeriod();
        mPowerAdvisor->updateTargetWorkDuration(idealVsyncPeriod);
    }

    if (mRefreshRateOverlaySpinner || mHdrSdrRatioOverlay) {
        Mutex::Autolock lock(mStateLock);
        if (const auto display = getDefaultDisplayDeviceLocked()) {
            display->animateOverlay();
        }
    }

    // Composite if transactions were committed, or if requested by HWC.
    bool mustComposite = mMustComposite.exchange(false);
    {
        mFrameTimeline->setSfWakeUp(ftl::to_underlying(vsyncId),
                                    pacesetterFrameTarget.frameBeginTime().ns(),
                                    Fps::fromPeriodNsecs(vsyncPeriod.ns()),
                                    mScheduler->getPacesetterRefreshRate());

        const bool flushTransactions = clearTransactionFlags(eTransactionFlushNeeded);
        bool transactionsAreEmpty;
        // 默认值为 false，具体可见 SurfaceFlinger 构造函数
        if (mLegacyFrontEndEnabled) {
            mustComposite |=
                    updateLayerSnapshotsLegacy(vsyncId, pacesetterFrameTarget.frameBeginTime().ns(),
                                               flushTransactions, transactionsAreEmpty);
        }
        // 默认值为 true，具体可见 SurfaceFlinger 构造函数
        if (mLayerLifecycleManagerEnabled) {
            mustComposite |=
                    updateLayerSnapshots(vsyncId, pacesetterFrameTarget.frameBeginTime().ns(),
                                         flushTransactions, transactionsAreEmpty);
        }

        if (transactionFlushNeeded()) {
            setTransactionFlags(eTransactionFlushNeeded);
        }

        // This has to be called after latchBuffers because we want to include the layers that have
        // been latched in the commit callback
        if (transactionsAreEmpty) {
            // Invoke empty transaction callbacks early.
            mTransactionCallbackInvoker.sendCallbacks(false /* onCommitOnly */);
        } else {
            // Invoke OnCommit callbacks.
            mTransactionCallbackInvoker.sendCallbacks(true /* onCommitOnly */);
        }
    }

    // Layers need to get updated (in the previous line) before we can use them for
    // choosing the refresh rate.
    // Hold mStateLock as chooseRefreshRateForContent promotes wp<Layer> to sp<Layer>
    // and may eventually call to ~Layer() if it holds the last reference
    {
        bool updateAttachedChoreographer = mUpdateAttachedChoreographer;
        mUpdateAttachedChoreographer = false;

        Mutex::Autolock lock(mStateLock);
        mScheduler->chooseRefreshRateForContent(mLayerLifecycleManagerEnabled
                                                        ? &mLayerHierarchyBuilder.getHierarchy()
                                                        : nullptr,
                                                updateAttachedChoreographer);
        initiateDisplayModeChanges();
    }

    updateCursorAsync();
    if (!mustComposite) {
        updateInputFlinger(vsyncId, pacesetterFrameTarget.frameBeginTime());
    }
    doActiveLayersTracingIfNeeded(false, mVisibleRegionsDirty,
                                  pacesetterFrameTarget.frameBeginTime(), vsyncId);

    mLastCommittedVsyncId = vsyncId;

    persistDisplayBrightness(mustComposite);

    return mustComposite && CC_LIKELY(mBootStage != BootStage::BOOTLOADER);
}
```

#### SurfaceFlinger::updateLayerSnapshots

> 这部分代码比较长，主要有如下几个核心流程

1. 采集 TransactionHandler 中的所有状态字段，保存到 Update 结构体中，为后续做准备

```c++
bool SurfaceFlinger::updateLayerSnapshots(VsyncId vsyncId, nsecs_t frameTimeNs,
                                          bool flushTransactions, bool& outTransactionsAreEmpty) {
    using Changes = frontend::RequestedLayerState::Changes;
    ATRACE_CALL();
    // 采集transactionHandler 中的所有状态字段。保存到 Update 结构体中
    frontend::Update update;
    if (flushTransactions) {
        ATRACE_NAME("TransactionHandler:flushTransactions");
        // Locking:
        // 1. to prevent onHandleDestroyed from being called while the state lock is held,
        // we must keep a copy of the transactions (specifically the composer
        // states) around outside the scope of the lock.
        // 2. Transactions and created layers do not share a lock. To prevent applying
        // transactions with layers still in the createdLayer queue, collect the transactions
        // before committing the created layers.
        // 3. Transactions can only be flushed after adding layers, since the layer can be a newly
        // created one
        mTransactionHandler.collectTransactions();
        {
            // TODO(b/238781169) lockless queue this and keep order.
            std::scoped_lock<std::mutex> lock(mCreatedLayersLock);
            update.layerCreatedStates = std::move(mCreatedLayers);
            mCreatedLayers.clear();
            update.newLayers = std::move(mNewLayers);
            mNewLayers.clear();
            update.layerCreationArgs = std::move(mNewLayerArgs);
            mNewLayerArgs.clear();
            update.destroyedHandles = std::move(mDestroyedHandles);
            mDestroyedHandles.clear();
        }

        mLayerLifecycleManager.addLayers(std::move(update.newLayers));
        update.transactions = mTransactionHandler.flushTransactions();
        if (mTransactionTracing) {
            mTransactionTracing->addCommittedTransactions(ftl::to_underlying(vsyncId), frameTimeNs,
                                                          update, mFrontEndDisplayInfos,
                                                          mFrontEndDisplayInfosChanged);
        }
        mLayerLifecycleManager.applyTransactions(update.transactions);
        mLayerLifecycleManager.onHandlesDestroyed(update.destroyedHandles);
        for (auto& legacyLayer : update.layerCreatedStates) {
            sp<Layer> layer = legacyLayer.layer.promote();
            if (layer) {
                mLegacyLayers[layer->sequence] = layer;
            }
        }
        mLayerHierarchyBuilder.update(mLayerLifecycleManager);
    }

    bool mustComposite = false;
    mustComposite |= applyAndCommitDisplayTransactionStates(update.transactions);

    {
        ATRACE_NAME("LayerSnapshotBuilder:update");
        frontend::LayerSnapshotBuilder::Args
                args{.root = mLayerHierarchyBuilder.getHierarchy(),
                     .layerLifecycleManager = mLayerLifecycleManager,
                     .displays = mFrontEndDisplayInfos,
                     .displayChanges = mFrontEndDisplayInfosChanged,
                     .globalShadowSettings = mDrawingState.globalShadowSettings,
                     .supportsBlur = mSupportsBlur,
                     .forceFullDamage = mForceFullDamage,
                     .supportedLayerGenericMetadata =
                             getHwComposer().getSupportedLayerGenericMetadata(),
                     .genericLayerMetadataKeyMap = getGenericLayerMetadataKeyMap(),
                     .skipRoundCornersWhenProtected =
                             !getRenderEngine().supportsProtectedContent()};
        mLayerSnapshotBuilder.update(args);
    }

    if (mLayerLifecycleManager.getGlobalChanges().any(Changes::Geometry | Changes::Input |
                                                      Changes::Hierarchy | Changes::Visibility)) {
        mUpdateInputInfo = true;
    }
    if (mLayerLifecycleManager.getGlobalChanges().any(Changes::VisibleRegion | Changes::Hierarchy |
                                                      Changes::Visibility | Changes::Geometry)) {
        mVisibleRegionsDirty = true;
    }
    if (mLayerLifecycleManager.getGlobalChanges().any(Changes::Hierarchy | Changes::FrameRate)) {
        // The frame rate of attached choreographers can only change as a result of a
        // FrameRate change (including when Hierarchy changes).
        mUpdateAttachedChoreographer = true;
    }
    outTransactionsAreEmpty = mLayerLifecycleManager.getGlobalChanges().get() == 0;
    mustComposite |= mLayerLifecycleManager.getGlobalChanges().get() != 0;

    bool newDataLatched = false;
    if (!mLegacyFrontEndEnabled) {
        ATRACE_NAME("DisplayCallbackAndStatsUpdates");
        mustComposite |= applyTransactions(update.transactions, vsyncId);
        traverseLegacyLayers([&](Layer* layer) { layer->commitTransaction(); });
        const nsecs_t latchTime = systemTime();
        bool unused = false;

        for (auto& layer : mLayerLifecycleManager.getLayers()) {
            if (layer->changes.test(frontend::RequestedLayerState::Changes::Created) &&
                layer->bgColorLayer) {
                sp<Layer> bgColorLayer = getFactory().createEffectLayer(
                        LayerCreationArgs(this, nullptr, layer->name,
                                          ISurfaceComposerClient::eFXSurfaceEffect, LayerMetadata(),
                                          std::make_optional(layer->id), true));
                mLegacyLayers[bgColorLayer->sequence] = bgColorLayer;
            }
            const bool willReleaseBufferOnLatch = layer->willReleaseBufferOnLatch();

            auto it = mLegacyLayers.find(layer->id);
            if (it == mLegacyLayers.end() &&
                layer->changes.test(frontend::RequestedLayerState::Changes::Destroyed)) {
                // Layer handle was created and immediately destroyed. It was destroyed before it
                // was added to the map.
                continue;
            }

            LLOG_ALWAYS_FATAL_WITH_TRACE_IF(it == mLegacyLayers.end(),
                                            "Couldnt find layer object for %s",
                                            layer->getDebugString().c_str());
            if (!layer->hasReadyFrame() && !willReleaseBufferOnLatch) {
                if (!it->second->hasBuffer()) {
                    // The last latch time is used to classify a missed frame as buffer stuffing
                    // instead of a missed frame. This is used to identify scenarios where we
                    // could not latch a buffer or apply a transaction due to backpressure.
                    // We only update the latch time for buffer less layers here, the latch time
                    // is updated for buffer layers when the buffer is latched.
                    it->second->updateLastLatchTime(latchTime);
                }
                continue;
            }

            const bool bgColorOnly =
                    !layer->externalTexture && (layer->bgColorLayerId != UNASSIGNED_LAYER_ID);
            if (willReleaseBufferOnLatch) {
                mLayersWithBuffersRemoved.emplace(it->second);
            }
            it->second->latchBufferImpl(unused, latchTime, bgColorOnly);
            newDataLatched = true;

            mLayersWithQueuedFrames.emplace(it->second);
            mLayersIdsWithQueuedFrames.emplace(it->second->sequence);
        }

        updateLayerHistory(latchTime);
        mLayerSnapshotBuilder.forEachVisibleSnapshot([&](const frontend::LayerSnapshot& snapshot) {
            if (mLayersIdsWithQueuedFrames.find(snapshot.path.id) ==
                mLayersIdsWithQueuedFrames.end())
                return;
            Region visibleReg;
            visibleReg.set(snapshot.transformedBoundsWithoutTransparentRegion);
            invalidateLayerStack(snapshot.outputFilter, visibleReg);
        });

        for (auto& destroyedLayer : mLayerLifecycleManager.getDestroyedLayers()) {
            mLegacyLayers.erase(destroyedLayer->id);
        }

        {
            ATRACE_NAME("LLM:commitChanges");
            mLayerLifecycleManager.commitChanges();
        }

        // enter boot animation on first buffer latch
        if (CC_UNLIKELY(mBootStage == BootStage::BOOTLOADER && newDataLatched)) {
            ALOGI("Enter boot animation");
            mBootStage = BootStage::BOOTANIMATION;
        }
    }
    mustComposite |= (getTransactionFlags() & ~eTransactionFlushNeeded) || newDataLatched;
    if (mustComposite && !mLegacyFrontEndEnabled) {
        commitTransactions();
    }

    return mustComposite;
}
```


#### SurfaceFlinger::commitCreatedLayers

处理添加 layer 的请求
```c++
bool SurfaceFlinger::commitCreatedLayers(VsyncId vsyncId,
                                         std::vector<LayerCreatedState>& createdLayers) {
    if (createdLayers.size() == 0) {
        return false;
    }

    Mutex::Autolock _l(mStateLock);
    for (const auto& createdLayer : createdLayers) {
        handleLayerCreatedLocked(createdLayer, vsyncId);
    }
    mLayersAdded = true;
    return mLayersAdded;
}
```

handleLayerCreatedLocked 中，会根据 LayerCreatedState 的参数，将 layer 添加到 layer 树上。如果是根节点，就添加到 mCurrentState.layersSortedByZ，否则就找到对应的父节点，然后通过 addChild 添加。最后会根据 layerStack 获取对应的 display，根据屏幕的旋转状态通过 updateTransformHint 给 layer 设置旋转状态。
```c++
void SurfaceFlinger::handleLayerCreatedLocked(const LayerCreatedState& state, VsyncId vsyncId) {
    sp<Layer> layer = state.layer.promote();
    if (!layer) {
        ALOGD("Layer was destroyed soon after creation %p", state.layer.unsafe_get());
        return;
    }
    MUTEX_ALIAS(mStateLock, layer->mFlinger->mStateLock);

    sp<Layer> parent;
    bool addToRoot = state.addToRoot;
    if (state.initialParent != nullptr) {
        parent = state.initialParent.promote();
        if (parent == nullptr) {
            ALOGD("Parent was destroyed soon after creation %p", state.initialParent.unsafe_get());
            addToRoot = false;
        }
    }
	// 如果当前节点是根节点，将 layer 添加到mCurrentState.layersSortedByZ中
    if (parent == nullptr && addToRoot) {
        layer->setIsAtRoot(true);
        mCurrentState.layersSortedByZ.add(layer);
    } else if (parent == nullptr) {
        layer->onRemovedFromCurrentState();
    } else if (parent->isRemovedFromCurrentState()) {
        parent->addChild(layer);
        layer->onRemovedFromCurrentState();
    } else {
	    // 非根节点天机到 parent 中的 child 节点中去
        parent->addChild(layer);
    }

    ui::LayerStack layerStack = layer->getLayerStack(LayerVector::StateSet::Current);
    sp<const DisplayDevice> hintDisplay;
    // Find the display that includes the layer.
    for (const auto& [token, display] : mDisplays) {
        if (display->getLayerStack() == layerStack) {
            hintDisplay = display;
            break;
        }
    }

    if (hintDisplay) {
        layer->updateTransformHint(hintDisplay->getTransformHint());
    }
}
```

#### SurfaceFlinger::applyTransactions

applyTransactions 主要适用于将 ResolvedComposerState 进行逐一遍历执行 updateLayerCallbacksAndStats 方法，以将ResolvedComposerState.state状态同步到ResolvedComposerState.state.surface->mLayer对象中。
```c++
bool SurfaceFlinger::applyTransactionState(const FrameTimelineInfo& frameTimelineInfo,
                                           std::vector<ResolvedComposerState>& states,
                                           Vector<DisplayState>& displays, uint32_t flags,
                                           const InputWindowCommands& inputWindowCommands,
                                           const int64_t desiredPresentTime, bool isAutoTimestamp,
                                           const std::vector<uint64_t>& uncacheBufferIds,
                                           const int64_t postTime, bool hasListenerCallbacks,
                                           const std::vector<ListenerCallbacks>& listenerCallbacks,
                                           int originPid, int originUid, uint64_t transactionId) {
    uint32_t transactionFlags = 0;
    if (!mLayerLifecycleManagerEnabled) {
        for (DisplayState& display : displays) {
            transactionFlags |= setDisplayStateLocked(display);
        }
    }

    // start and end registration for listeners w/ no surface so they can get their callback.  Note
    // that listeners with SurfaceControls will start registration during setClientStateLocked
    // below.
    for (const auto& listener : listenerCallbacks) {
        mTransactionCallbackInvoker.addEmptyTransaction(listener);
    }
    nsecs_t now = systemTime();
    uint32_t clientStateFlags = 0;
    for (auto& resolvedState : states) {
        if (mLegacyFrontEndEnabled) {
            clientStateFlags |=
                    setClientStateLocked(frameTimelineInfo, resolvedState, desiredPresentTime,
                                         isAutoTimestamp, postTime, transactionId);

        } else /*mLayerLifecycleManagerEnabled*/ {
            clientStateFlags |= updateLayerCallbacksAndStats(frameTimelineInfo, resolvedState,
                                                             desiredPresentTime, isAutoTimestamp,
                                                             postTime, transactionId);
        }
        if ((flags & eAnimation) && resolvedState.state.surface) {
            if (const auto layer = LayerHandle::getLayer(resolvedState.state.surface)) {
                const auto layerProps = scheduler::LayerProps{
                        .visible = layer->isVisible(),
                        .bounds = layer->getBounds(),
                        .transform = layer->getTransform(),
                        .setFrameRateVote = layer->getFrameRateForLayerTree(),
                        .frameRateSelectionPriority = layer->getFrameRateSelectionPriority(),
                        .isFrontBuffered = layer->isFrontBuffered(),
                };
                layer->recordLayerHistoryAnimationTx(layerProps, now);
            }
        }
    }

    transactionFlags |= clientStateFlags;
    transactionFlags |= addInputWindowCommands(inputWindowCommands);

    for (uint64_t uncacheBufferId : uncacheBufferIds) {
        mBufferIdsToUncache.push_back(uncacheBufferId);
    }

    // If a synchronous transaction is explicitly requested without any changes, force a transaction
    // anyway. This can be used as a flush mechanism for previous async transactions.
    // Empty animation transaction can be used to simulate back-pressure, so also force a
    // transaction for empty animation transactions.
    if (transactionFlags == 0 && (flags & eAnimation)) {
        transactionFlags = eTransactionNeeded;
    }

    bool needsTraversal = false;
    if (transactionFlags) {
        // We are on the main thread, we are about to perform a traversal. Clear the traversal bit
        // so we don't have to wake up again next frame to perform an unnecessary traversal.
        if (transactionFlags & eTraversalNeeded) {
            transactionFlags = transactionFlags & (~eTraversalNeeded);
            needsTraversal = true;
        }
        if (transactionFlags) {
            setTransactionFlags(transactionFlags);
        }
    }

    return needsTraversal;
}
```

主要用于将 layer_state_t 结构体字段同步到 android::Layer 字段中。
```c++
uint32_t SurfaceFlinger::updateLayerCallbacksAndStats(const FrameTimelineInfo& frameTimelineInfo,
                                                      ResolvedComposerState& composerState,
                                                      int64_t desiredPresentTime,
                                                      bool isAutoTimestamp, int64_t postTime,
                                                      uint64_t transactionId) {
    layer_state_t& s = composerState.state;

    std::vector<ListenerCallbacks> filteredListeners;
    for (auto& listener : s.listeners) {
        // Starts a registration but separates the callback ids according to callback type. This
        // allows the callback invoker to send on latch callbacks earlier.
        // note that startRegistration will not re-register if the listener has
        // already be registered for a prior surface control

        ListenerCallbacks onCommitCallbacks = listener.filter(CallbackId::Type::ON_COMMIT);
        if (!onCommitCallbacks.callbackIds.empty()) {
            filteredListeners.push_back(onCommitCallbacks);
        }

        ListenerCallbacks onCompleteCallbacks = listener.filter(CallbackId::Type::ON_COMPLETE);
        if (!onCompleteCallbacks.callbackIds.empty()) {
            filteredListeners.push_back(onCompleteCallbacks);
        }
    }

    const uint64_t what = s.what;
    uint32_t flags = 0;
    sp<Layer> layer = nullptr;
    if (s.surface) {
	    // 获取layer_state_t.surface 对象并将其转为LayerHandle获取mLayer对象
	    // 及获取s.surface->mLayer对象。
        layer = LayerHandle::getLayer(s.surface);
    } else {
        // The client may provide us a null handle. Treat it as if the layer was removed.
        ALOGW("Attempt to set client state with a null layer handle");
    }
    if (layer == nullptr) {
        for (auto& [listener, callbackIds] : s.listeners) {
            mTransactionCallbackInvoker.addCallbackHandle(sp<CallbackHandle>::make(listener,
                                                                                   callbackIds,
                                                                                   s.surface),
                                                          std::vector<JankData>());
        }
        return 0;
    }
    if (what & layer_state_t::eProducerDisconnect) {
        layer->onDisconnect();
    }
    // 将 layer_state_t 同步到 android::Layer 中
    std::optional<nsecs_t> dequeueBufferTimestamp;
    if (what & layer_state_t::eMetadataChanged) {
        dequeueBufferTimestamp = s.metadata.getInt64(gui::METADATA_DEQUEUE_TIME);
    }

    std::vector<sp<CallbackHandle>> callbackHandles;
    if ((what & layer_state_t::eHasListenerCallbacksChanged) && (!filteredListeners.empty())) {
        for (auto& [listener, callbackIds] : filteredListeners) {
            callbackHandles.emplace_back(
                    sp<CallbackHandle>::make(listener, callbackIds, s.surface));
        }
    }
    // TODO(b/238781169) remove after screenshot refactor, currently screenshots
    // requires to read drawing state from binder thread. So we need to fix that
    // before removing this.
    if (what & layer_state_t::eBufferTransformChanged) {
        if (layer->setTransform(s.bufferTransform)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eTransformToDisplayInverseChanged) {
        if (layer->setTransformToDisplayInverse(s.transformToDisplayInverse))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eCropChanged) {
        if (layer->setCrop(s.crop)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eSidebandStreamChanged) {
        if (layer->setSidebandStream(s.sidebandStream, frameTimelineInfo, postTime))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eDataspaceChanged) {
        if (layer->setDataspace(s.dataspace)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eExtendedRangeBrightnessChanged) {
        if (layer->setExtendedRangeBrightness(s.currentHdrSdrRatio, s.desiredHdrSdrRatio)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eDesiredHdrHeadroomChanged) {
        if (layer->setDesiredHdrHeadroom(s.desiredHdrSdrRatio)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eBufferChanged) {
        std::optional<ui::Transform::RotationFlags> transformHint = std::nullopt;
        frontend::LayerSnapshot* snapshot = mLayerSnapshotBuilder.getSnapshot(layer->sequence);
        if (snapshot) {
            transformHint = snapshot->transformHint;
        }
        layer->setTransformHint(transformHint);
        if (layer->setBuffer(composerState.externalTexture, *s.bufferData, postTime,
                             desiredPresentTime, isAutoTimestamp, dequeueBufferTimestamp,
                             frameTimelineInfo)) {
            flags |= eTraversalNeeded;
        }
        mLayersWithQueuedFrames.emplace(layer);
    } else if (frameTimelineInfo.vsyncId != FrameTimelineInfo::INVALID_VSYNC_ID) {
        layer->setFrameTimelineVsyncForBufferlessTransaction(frameTimelineInfo, postTime);
    }

    if ((what & layer_state_t::eBufferChanged) == 0) {
        layer->setDesiredPresentTime(desiredPresentTime, isAutoTimestamp);
    }

    if (what & layer_state_t::eTrustedPresentationInfoChanged) {
        if (layer->setTrustedPresentationInfo(s.trustedPresentationThresholds,
                                              s.trustedPresentationListener)) {
            flags |= eTraversalNeeded;
        }
    }

    const auto& requestedLayerState = mLayerLifecycleManager.getLayerFromId(layer->getSequence());
    bool willPresentCurrentTransaction = requestedLayerState &&
            (requestedLayerState->hasReadyFrame() ||
             requestedLayerState->willReleaseBufferOnLatch());
    if (layer->setTransactionCompletedListeners(callbackHandles, willPresentCurrentTransaction))
        flags |= eTraversalNeeded;

    return flags;
}
```

#### SurfaceFlinger::commitTransactions

```c++
void SurfaceFlinger::commitTransactions() {
    ATRACE_CALL();

    // Keep a copy of the drawing state (that is going to be overwritten
    // by commitTransactionsLocked) outside of mStateLock so that the side
    // effects of the State assignment don't happen with mStateLock held,
    // which can cause deadlocks.
    State drawingState(mDrawingState);

    Mutex::Autolock lock(mStateLock);
    mDebugInTransaction = systemTime();

    // Here we're guaranteed that some transaction flags are set
    // so we can call commitTransactionsLocked unconditionally.
    // We clear the flags with mStateLock held to guarantee that
    // mCurrentState won't change until the transaction is committed.
    mScheduler->modulateVsync({}, &VsyncModulator::onTransactionCommit);
    commitTransactionsLocked(clearTransactionFlags(eTransactionMask));

    mDebugInTransaction = 0;
}
```

1.commitTransactionsLocked 先更新了屏幕相关的配置，然后遍历 layer，利用 layer 里面的 LayerStack 找到 layer 对应的屏，根据屏幕的旋转更新 layer 的旋转

如果有增加或者删除 layer，标记 mVisibleRegionsDirty 和 mUpdateInputInfo 为 true,遍历要删除的所有 layer，将他们的可见区域重叠，这些区域是需要刷新的。



2.doCommitTransactions 先处理需要删除的 layer，如果是 root layer，就将它从 layersSortedByZ 删除，没有父节点的就加到 mOffscreenLayers，防止后面需要遍历的时候我们拿不到他们了。

currentState/currentChildren/currentParent 都相当于中间变量，我们的改变都先配置在他们上，当要去使用它，让它生效时，就会赋值到 DrawingXXX 上。

commitChildList 会递归所有 child，然后将每个 layer 的将 mDrawingChildren 和 mDrawingParent 更新成 mCurrentChildren，mCurrentParent。最后同步更新 mirrorlayer 里面的属性


#### SurfaceFlinger::latchBuffers


将 drawState 同步到BufferInfo 中

## SurfaceFlinger::composite

SurfaceFlinger::commit 主要是做数据的准备，commit 执行完成后SurfaceFlinger 所需要的数据也就准备好了，composite 会做图层的合成操作。


```c++
CompositeResultsPerDisplay SurfaceFlinger::composite(
        PhysicalDisplayId pacesetterId, const scheduler::FrameTargeters& frameTargeters) {
    const scheduler::FrameTarget& pacesetterTarget =
            frameTargeters.get(pacesetterId)->get()->target();

    const VsyncId vsyncId = pacesetterTarget.vsyncId();
    ATRACE_NAME(ftl::Concat(__func__, ' ', ftl::to_underlying(vsyncId)).c_str());

    compositionengine::CompositionRefreshArgs refreshArgs;
    refreshArgs.powerCallback = this;
    const auto& displays = FTL_FAKE_GUARD(mStateLock, mDisplays);
    // 1. 添加准备refreshArgs.outputs
    refreshArgs.outputs.reserve(displays.size());

    // Add outputs for physical displays.
    // 1.1 添加物理的 display
    for (const auto& [id, targeter] : frameTargeters) { // 遍历所有的 frameTageter 
        ftl::FakeGuard guard(mStateLock);
		// 将 id 转为display 添加到refreshArgs.outputs中去
        if (const auto display = getCompositionDisplayLocked(id)) {
            refreshArgs.outputs.push_back(display);
        }
		// 添加 refreshArgs.frameTargets
        refreshArgs.frameTargets.try_emplace(id, &targeter->target());
    }
	// 1.2 添加虚拟的 display
    std::vector<DisplayId> displayIds;
    for (const auto& [_, display] : displays) {
        displayIds.push_back(display->getId());
        display->tracePowerMode();

        // Add outputs for virtual displays.
        if (display->isVirtual()) {
            const Fps refreshRate = display->getAdjustedRefreshRate();

            if (!refreshRate.isValid() ||
                mScheduler->isVsyncInPhase(pacesetterTarget.frameBeginTime(), refreshRate)) {
                refreshArgs.outputs.push_back(display->getCompositionDisplay());
            }
        }
    }
    mPowerAdvisor->setDisplays(displayIds);

    const bool updateTaskMetadata = mCompositionEngine->getFeatureFlags().test(
            compositionengine::Feature::kSnapshotLayerMetadata);
    // update Task Metadata
    if (updateTaskMetadata && (mVisibleRegionsDirty || mLayerMetadataSnapshotNeeded)) {
        updateLayerMetadataSnapshot();
        mLayerMetadataSnapshotNeeded = false;
    }

	// 填充refreshArgs.borderInfoList
    if (DOES_CONTAIN_BORDER) {
        refreshArgs.borderInfoList.clear();
        mDrawingState.traverse([&refreshArgs](Layer* layer) {
            if (layer->isBorderEnabled()) {
                compositionengine::BorderRenderInfo info;
                info.width = layer->getBorderWidth();
                info.color = layer->getBorderColor();
                layer->traverse(LayerVector::StateSet::Drawing, [&info](Layer* ilayer) {
                    info.layerIds.push_back(ilayer->getSequence());
                });
                refreshArgs.borderInfoList.emplace_back(std::move(info));
            }
        });
    }
	// 复制 bufferIdsToUncache
    refreshArgs.bufferIdsToUncache = std::move(mBufferIdsToUncache);
	// 2. 将 layersWithQueuedFrames 转为 LayerFE
    refreshArgs.layersWithQueuedFrames.reserve(mLayersWithQueuedFrames.size());
    for (auto layer : mLayersWithQueuedFrames) {
        if (auto layerFE = layer->getCompositionEngineLayerFE())
            refreshArgs.layersWithQueuedFrames.push_back(layerFE);
    }
	// 填充refreshArgs 参数
    refreshArgs.outputColorSetting = mDisplayColorSetting;
    refreshArgs.forceOutputColorMode = mForceColorMode;

    refreshArgs.updatingOutputGeometryThisFrame = mVisibleRegionsDirty;
    refreshArgs.updatingGeometryThisFrame = mGeometryDirty.exchange(false) || mVisibleRegionsDirty;
    refreshArgs.internalDisplayRotationFlags = getActiveDisplayRotationFlags();

    if (CC_UNLIKELY(mDrawingState.colorMatrixChanged)) {
        refreshArgs.colorTransformMatrix = mDrawingState.colorMatrix;
        mDrawingState.colorMatrixChanged = false;
    }

    refreshArgs.devOptForceClientComposition = mDebugDisableHWC;

    if (mDebugFlashDelay != 0) {
        refreshArgs.devOptForceClientComposition = true;
        refreshArgs.devOptFlashDirtyRegionsDelay = std::chrono::milliseconds(mDebugFlashDelay);
    }

    // TODO(b/255601557) Update frameInterval per display
    refreshArgs.frameInterval =
            mScheduler->getNextFrameInterval(pacesetterId, pacesetterTarget.expectedPresentTime());
    const auto scheduledFrameResultOpt = mScheduler->getScheduledFrameResult();
    const auto scheduledFrameTimeOpt = scheduledFrameResultOpt
            ? std::optional{scheduledFrameResultOpt->callbackTime}
            : std::nullopt;
    refreshArgs.scheduledFrameTime = scheduledFrameTimeOpt;
    refreshArgs.hasTrustedPresentationListener = mNumTrustedPresentationListeners > 0;
    // Store the present time just before calling to the composition engine so we could notify
    // the scheduler.
    const auto presentTime = systemTime();

    constexpr bool kCursorOnly = false;
    // 3. 调用 moveSnapshotsToCompositionArgs 来更新所有的 layerFE 的 mSnapshot
    const auto layers = moveSnapshotsToCompositionArgs(refreshArgs, kCursorOnly);

    if (mLayerLifecycleManagerEnabled && !mVisibleRegionsDirty) {
        for (const auto& [token, display] : FTL_FAKE_GUARD(mStateLock, mDisplays)) {
            auto compositionDisplay = display->getCompositionDisplay();
            if (!compositionDisplay->getState().isEnabled) continue;
            for (auto outputLayer : compositionDisplay->getOutputLayersOrderedByZ()) {
                if (outputLayer->getLayerFE().getCompositionState() == nullptr) {
                    // This is unexpected but instead of crashing, capture traces to disk
                    // and recover gracefully by forcing CE to rebuild layer stack.
                    ALOGE("Output layer %s for display %s %" PRIu64 " has a null "
                          "snapshot. Forcing mVisibleRegionsDirty",
                          outputLayer->getLayerFE().getDebugName(),
                          compositionDisplay->getName().c_str(), compositionDisplay->getId().value);

                    TransactionTraceWriter::getInstance().invoke(__func__, /* overwrite= */ false);
                    mVisibleRegionsDirty = true;
                    refreshArgs.updatingOutputGeometryThisFrame = mVisibleRegionsDirty;
                    refreshArgs.updatingGeometryThisFrame = mVisibleRegionsDirty;
                }
            }
        }
    }
	// 4. 调用 renderEngine 进行图层的合成操作
    mCompositionEngine->present(refreshArgs);
    moveSnapshotsFromCompositionArgs(refreshArgs, layers);

    for (auto [layer, layerFE] : layers) {
        CompositionResult compositionResult{layerFE->stealCompositionResult()};
        layer->onPreComposition(compositionResult.refreshStartTime);
        for (auto& [releaseFence, layerStack] : compositionResult.releaseFences) {
            Layer* clonedFrom = layer->getClonedFrom().get();
            auto owningLayer = clonedFrom ? clonedFrom : layer;
            owningLayer->onLayerDisplayed(std::move(releaseFence), layerStack);
        }
        if (compositionResult.lastClientCompositionFence) {
            layer->setWasClientComposed(compositionResult.lastClientCompositionFence);
        }
    }

    mTimeStats->recordFrameDuration(pacesetterTarget.frameBeginTime().ns(), systemTime());

    // Send a power hint after presentation is finished.
    if (mPowerHintSessionEnabled) {
        // Now that the current frame has been presented above, PowerAdvisor needs the present time
        // of the previous frame (whose fence is signaled by now) to determine how long the HWC had
        // waited on that fence to retire before presenting.
        const auto& previousPresentFence = pacesetterTarget.presentFenceForPreviousFrame();

        mPowerAdvisor->setSfPresentTiming(TimePoint::fromNs(previousPresentFence->getSignalTime()),
                                          TimePoint::now());
        mPowerAdvisor->reportActualWorkDuration();
    }

    if (mScheduler->onCompositionPresented(presentTime)) {
        scheduleComposite(FrameHint::kNone);
    }

    mNotifyExpectedPresentMap[pacesetterId].hintStatus = NotifyExpectedPresentHintStatus::Start;
    onCompositionPresented(pacesetterId, frameTargeters, presentTime);

    const bool hadGpuComposited =
            multiDisplayUnion(mCompositionCoverage).test(CompositionCoverage::Gpu);
    mCompositionCoverage.clear();

    TimeStats::ClientCompositionRecord clientCompositionRecord;

    for (const auto& [_, display] : displays) {
        const auto& state = display->getCompositionDisplay()->getState();
        CompositionCoverageFlags& flags =
                mCompositionCoverage.try_emplace(display->getId()).first->second;

        if (state.usesDeviceComposition) {
            flags |= CompositionCoverage::Hwc;
        }

        if (state.reusedClientComposition) {
            flags |= CompositionCoverage::GpuReuse;
        } else if (state.usesClientComposition) {
            flags |= CompositionCoverage::Gpu;
        }

        clientCompositionRecord.predicted |=
                (state.strategyPrediction != CompositionStrategyPredictionState::DISABLED);
        clientCompositionRecord.predictionSucceeded |=
                (state.strategyPrediction == CompositionStrategyPredictionState::SUCCESS);
    }

    const auto coverage = multiDisplayUnion(mCompositionCoverage);
    const bool hasGpuComposited = coverage.test(CompositionCoverage::Gpu);

    clientCompositionRecord.hadClientComposition = hasGpuComposited;
    clientCompositionRecord.reused = coverage.test(CompositionCoverage::GpuReuse);
    clientCompositionRecord.changed = hadGpuComposited != hasGpuComposited;

    mTimeStats->pushCompositionStrategyState(clientCompositionRecord);

    using namespace ftl::flag_operators;

    // TODO(b/160583065): Enable skip validation when SF caches all client composition layers.
    const bool hasGpuUseOrReuse =
            coverage.any(CompositionCoverage::Gpu | CompositionCoverage::GpuReuse);
    mScheduler->modulateVsync({}, &VsyncModulator::onDisplayRefresh, hasGpuUseOrReuse);

    mLayersWithQueuedFrames.clear();
    mLayersIdsWithQueuedFrames.clear();
    doActiveLayersTracingIfNeeded(true, mVisibleRegionsDirty, pacesetterTarget.frameBeginTime(),
                                  vsyncId);

    updateInputFlinger(vsyncId, pacesetterTarget.frameBeginTime());

    if (mVisibleRegionsDirty) mHdrLayerInfoChanged = true;
    mVisibleRegionsDirty = false;

    if (mCompositionEngine->needsAnotherUpdate()) {
        scheduleCommit(FrameHint::kNone);
    }

    if (mPowerHintSessionEnabled) {
        mPowerAdvisor->setCompositeEnd(TimePoint::now());
    }

    CompositeResultsPerDisplay resultsPerDisplay;

    // Filter out virtual displays.
    for (const auto& [id, coverage] : mCompositionCoverage) {
        if (const auto idOpt = PhysicalDisplayId::tryCast(id)) {
            resultsPerDisplay.try_emplace(*idOpt, CompositeResult{coverage});
        }
    }

    return resultsPerDisplay;
}
```

### output准备

将物理 display & virtual display 添加到 refreshArgs.outputs中
```c++
// 1. 添加准备refreshArgs.outputs
refreshArgs.outputs.reserve(displays.size());

// Add outputs for physical displays.
// 1.1 添加物理的 display
for (const auto& [id, targeter] : frameTargeters) { // 遍历所有的 frameTageter 
	ftl::FakeGuard guard(mStateLock);
	// 将 id 转为display 添加到refreshArgs.outputs中去
	if (const auto display = getCompositionDisplayLocked(id)) {
		refreshArgs.outputs.push_back(display);
	}
	// 添加 refreshArgs.frameTargets
	refreshArgs.frameTargets.try_emplace(id, &targeter->target());
}
// 1.2 添加虚拟的 display
std::vector<DisplayId> displayIds;
for (const auto& [_, display] : displays) {
	displayIds.push_back(display->getId());
	display->tracePowerMode();

	// Add outputs for virtual displays.
	if (display->isVirtual()) {
		const Fps refreshRate = display->getAdjustedRefreshRate();

		if (!refreshRate.isValid() ||
			mScheduler->isVsyncInPhase(pacesetterTarget.frameBeginTime(), refreshRate)) {
			refreshArgs.outputs.push_back(display->getCompositionDisplay());
		}
	}
}
mPowerAdvisor->setDisplays(displayIds);
```



### 准备LayerFE

通过逐一调用 mLayersWithQueuedFrames调用其getCompositeEngineLayerFE 方法获取 LayerFE 方法。

```c++
// 2. 将 layersWithQueuedFrames 转为 LayerFE
refreshArgs.layersWithQueuedFrames.reserve(mLayersWithQueuedFrames.size());
for (auto layer : mLayersWithQueuedFrames) {
	if (auto layerFE = layer->getCompositionEngineLayerFE())
		refreshArgs.layersWithQueuedFrames.push_back(layerFE);
}
```

layersWithQueuedFrames源于mLayersWithQueuedFrames.getCompositionEngineLayerFE()，那么问题来了mLayersWithQueuedFrames从哪来的
```c++
bool SurfaceFlinger::updateLayerSnapshots(VsyncId vsyncId, nsecs_t frameTimeNs,
                                          bool flushTransactions, bool& outTransactionsAreEmpty) {
                                  
	// ......

	for (auto& layer : mLayerLifecycleManager.getLayers()) {
            // .....
            it->second->latchBufferImpl(unused, latchTime, bgColorOnly);
			// ......
            mLayersWithQueuedFrames.emplace(it->second);
            mLayersIdsWithQueuedFrames.emplace(it->second->sequence);
        }

	// ......
}
```

因此简单归纳下LayerFE 的创建流程
mLayerLifecycleManager.getLayers() 
	-> mLayersWithQueuedFrames 
	-> mLayersWithQueuedFrames.getCompositionEngineLayerFE() 
	-> layersWithQueuedFrames
	-> LayerFE

### 同步状态到 LayerFE

moveSnapshotsToCompositionArgs
该方法主要作用是将 Snapshots 的内容添加到CompositionRefreshArgs中去即refreshArgs.layers中去。
```c++
std::vector<std::pair<Layer*, LayerFE*>> SurfaceFlinger::moveSnapshotsToCompositionArgs(
        compositionengine::CompositionRefreshArgs& refreshArgs, bool cursorOnly) {
    std::vector<std::pair<Layer*, LayerFE*>> layers;
    // 本机mLayerLifecycleManagerEnabled为 true
    if (mLayerLifecycleManagerEnabled) {
        nsecs_t currentTime = systemTime();
        mLayerSnapshotBuilder.forEachVisibleSnapshot(
                [&](std::unique_ptr<frontend::LayerSnapshot>& snapshot) {
                    if (cursorOnly &&
                        snapshot->compositionType !=
                                aidl::android::hardware::graphics::composer3::Composition::CURSOR) {
                        return;
                    }
					// 如果 snapshot 没有任何东西需要渲染直接 return
                    if (!snapshot->hasSomethingToDraw()) {
                        return;
                    }
					
                    auto it = mLegacyLayers.find(snapshot->sequence);
                    LLOG_ALWAYS_FATAL_WITH_TRACE_IF(it == mLegacyLayers.end(),
                                                    "Couldnt find layer object for %s",
                                                    snapshot->getDebugString().c_str());
                    auto& legacyLayer = it->second;
                    sp<LayerFE> layerFE = legacyLayer->getCompositionEngineLayerFE(snapshot->path);
                    snapshot->fps = getLayerFramerate(currentTime, snapshot->sequence);
                    layerFE->mSnapshot = std::move(snapshot);
                    // 将 layerFE 添加到refreshArgs.layers中
                    refreshArgs.layers.push_back(layerFE);
                    // 将 layer & layerFE 添加到 layers 变量中
                    layers.emplace_back(legacyLayer.get(), layerFE.get());
                });
    }
	// mLegacyFrontEndEnabled 默认为 false
    if (mLegacyFrontEndEnabled && !mLayerLifecycleManagerEnabled) {
        // ......
    }

	// 返回 layers
    return layers;
}
```


数据流向：
App() -> Layer -> LayerFE

###  CompositionEngine::present


```c++
void CompositionEngine::present(CompositionRefreshArgs& args) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);

    preComposition(args);

	// 1. 构建 Output 对象
    {
        // latchedLayers is used to track the set of front-end layer state that
        // has been latched across all outputs for the prepare step, and is not
        // needed for anything else.
        LayerFESet latchedLayers;

        for (const auto& output : args.outputs) {
            output->prepare(args, latchedLayers);
        }
    }

    // Offloading the HWC call for `present` allows us to simultaneously call it
    // on multiple displays. This is desirable because these calls block and can
    // be slow.
    offloadOutputs(args.outputs);

    ui::DisplayVector<ftl::Future<std::monostate>> presentFutures;
    // 2. 逐一合成图层
    for (const auto& output : args.outputs) {
        presentFutures.push_back(output->present(args));
    }

    {
        ATRACE_NAME("Waiting on HWC");
        for (auto& future : presentFutures) {
            // TODO(b/185536303): Call ftl::Future::wait() once it exists, since
            // we do not need the return value of get().
            future.get();
        }
    }
}
```


#### Output::prepare

调用 rebuildLayerStacks，最后的数据准备:
```c++
void Output::prepare(const compositionengine::CompositionRefreshArgs& refreshArgs,
                     LayerFESet& geomSnapshots) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);

    rebuildLayerStacks(refreshArgs, geomSnapshots);
    uncacheBuffers(refreshArgs.bufferIdsToUncache);
}
```

##### rebuildLayerStacks

Output 代表了一个显示设备，OutputLayer 代表一个图层

这里会构建 OutputLayer，计算各个区域大小。

OutputLayer，这个是一个合成的中间产物，相比于LayerFE，他记录了更多在合成过程中计算出来的信息，比如一些区域信息，比如脏区等。

调用 collectVisibleLayers：
```c++
void Output::rebuildLayerStacks(const compositionengine::CompositionRefreshArgs& refreshArgs,
                                LayerFESet& layerFESet) {
    auto& outputState = editState();

    // Do nothing if this output is not enabled or there is no need to perform this update
    if (!outputState.isEnabled || CC_LIKELY(!refreshArgs.updatingOutputGeometryThisFrame)) {
        return;
    }
    ATRACE_CALL();
    ALOGV(__FUNCTION__);

    // Process the layers to determine visibility and coverage
    compositionengine::Output::CoverageState coverage{layerFESet};
    coverage.aboveCoveredLayersExcludingOverlays = refreshArgs.hasTrustedPresentationListener
            ? std::make_optional<Region>()
            : std::nullopt;
    collectVisibleLayers(refreshArgs, coverage);

    // Compute the resulting coverage for this output, and store it for later
    const ui::Transform& tr = outputState.transform;
    Region undefinedRegion{outputState.displaySpace.getBoundsAsRect()};
    undefinedRegion.subtractSelf(tr.transform(coverage.aboveOpaqueLayers));

    outputState.undefinedRegion = undefinedRegion;
    outputState.dirtyRegion.orSelf(coverage.dirtyRegion);
}
```

遍历所有的 layerFE，调用ensureOutputLayerIfVisible构建 OuputLayer，然后将 OutputLayer 暂存到mPendingOutputLayersOrderedByZ里

```c++
void Output::collectVisibleLayers(const compositionengine::CompositionRefreshArgs& refreshArgs,
                                  compositionengine::Output::CoverageState& coverage) {
    // Evaluate the layers from front to back to determine what is visible. This
    // also incrementally calculates the coverage information for each layer as
    // well as the entire output.
    for (auto layer : reversed(refreshArgs.layers)) {
	    // 遍历所有 layerFE，调用 ensureOutputLayerIfVisible,构建 OutputLayer，这些 OutputLayer 会暂时存在 mPendingOutputLayersOrderedByZ 里
        // Incrementally process the coverage for each layer
        ensureOutputLayerIfVisible(layer, coverage);

        // TODO(b/121291683): Stop early if the output is completely covered and
        // no more layers could even be visible underneath the ones on top.
    }

    setReleasedLayers(refreshArgs);

    finalizePendingOutputLayers();
}
```

- ensureOutputLayerIfVisible
传入layerFE用于创建 OutputLayer 对象
```c++
void Output::ensureOutputLayerIfVisible(sp<compositionengine::LayerFE>& layerFE,
                                        compositionengine::Output::CoverageState& coverage) {
    // Ensure we have a snapshot of the basic geometry layer state. Limit the
    // snapshots to once per frame for each candidate layer, as layers may
    // appear on multiple outputs.
    if (!coverage.latchedLayers.count(layerFE)) {
        coverage.latchedLayers.insert(layerFE);
    }

    // Only consider the layers on this output
    if (!includesLayer(layerFE)) {
        return;
    }

    // Obtain a read-only pointer to the front-end layer state
    const auto* layerFEState = layerFE->getCompositionState();
    if (CC_UNLIKELY(!layerFEState)) {
        return;
    }

    // handle hidden surfaces by setting the visible region to empty
    if (CC_UNLIKELY(!layerFEState->isVisible)) {
        return;
    }

    bool computeAboveCoveredExcludingOverlays = coverage.aboveCoveredLayersExcludingOverlays &&
            !layerFEState->outputFilter.toInternalDisplay;

    /*
     * opaqueRegion: area of a surface that is fully opaque.
     */
    Region opaqueRegion;

    /*
     * visibleRegion: area of a surface that is visible on screen and not fully
     * transparent. This is essentially the layer's footprint minus the opaque
     * regions above it. Areas covered by a translucent surface are considered
     * visible.
     */
    Region visibleRegion;

    /*
     * coveredRegion: area of a surface that is covered by all visible regions
     * above it (which includes the translucent areas).
     */
    Region coveredRegion;

    /*
     * transparentRegion: area of a surface that is hinted to be completely
     * transparent.
     * This is used to tell when the layer has no visible non-transparent
     * regions and can be removed from the layer list. It does not affect the
     * visibleRegion of this layer or any layers beneath it. The hint may not
     * be correct if apps don't respect the SurfaceView restrictions (which,
     * sadly, some don't).
     *
     * In addition, it is used on DISPLAY_DECORATION layers to specify the
     * blockingRegion, allowing the DPU to skip it to save power. Once we have
     * hardware that supports a blockingRegion on frames with AFBC, it may be
     * useful to use this for other layers, too, so long as we can prevent
     * regressions on b/7179570.
     */
    Region transparentRegion;

    /*
     * shadowRegion: Region cast by the layer's shadow.
     */
    Region shadowRegion;

    /**
     * covered region above excluding internal display overlay layers
     */
    std::optional<Region> coveredRegionExcludingDisplayOverlays = std::nullopt;

    const ui::Transform& tr = layerFEState->geomLayerTransform;

    // Get the visible region
    // TODO(b/121291683): Is it worth creating helper methods on LayerFEState
    // for computations like this?
    const Rect visibleRect(tr.transform(layerFEState->geomLayerBounds));
    visibleRegion.set(visibleRect);

    if (layerFEState->shadowSettings.length > 0.0f) {
        // if the layer casts a shadow, offset the layers visible region and
        // calculate the shadow region.
        const auto inset = static_cast<int32_t>(ceilf(layerFEState->shadowSettings.length) * -1.0f);
        Rect visibleRectWithShadows(visibleRect);
        visibleRectWithShadows.inset(inset, inset, inset, inset);
        visibleRegion.set(visibleRectWithShadows);
        shadowRegion = visibleRegion.subtract(visibleRect);
    }

    if (visibleRegion.isEmpty()) {
        return;
    }

    // Remove the transparent area from the visible region
    if (!layerFEState->isOpaque) {
        if (tr.preserveRects()) {
            // Clip the transparent region to geomLayerBounds first
            // The transparent region may be influenced by applications, for
            // instance, by overriding ViewGroup#gatherTransparentRegion with a
            // custom view. Once the layer stack -> display mapping is known, we
            // must guard against very wrong inputs to prevent underflow or
            // overflow errors. We do this here by constraining the transparent
            // region to be within the pre-transform layer bounds, since the
            // layer bounds are expected to play nicely with the full
            // transform.
            const Region clippedTransparentRegionHint =
                    layerFEState->transparentRegionHint.intersect(
                            Rect(layerFEState->geomLayerBounds));

            if (clippedTransparentRegionHint.isEmpty()) {
                if (!layerFEState->transparentRegionHint.isEmpty()) {
                    ALOGD("Layer: %s had an out of bounds transparent region",
                          layerFE->getDebugName());
                    layerFEState->transparentRegionHint.dump("transparentRegionHint");
                }
                transparentRegion.clear();
            } else {
                transparentRegion = tr.transform(clippedTransparentRegionHint);
            }
        } else {
            // transformation too complex, can't do the
            // transparent region optimization.
            transparentRegion.clear();
        }
    }

    // compute the opaque region
    const auto layerOrientation = tr.getOrientation();
    if (layerFEState->isOpaque && ((layerOrientation & ui::Transform::ROT_INVALID) == 0)) {
        // If we one of the simple category of transforms (0/90/180/270 rotation
        // + any flip), then the opaque region is the layer's footprint.
        // Otherwise we don't try and compute the opaque region since there may
        // be errors at the edges, and we treat the entire layer as
        // translucent.
        opaqueRegion.set(visibleRect);
    }

    // Clip the covered region to the visible region
    coveredRegion = coverage.aboveCoveredLayers.intersect(visibleRegion);

    // Update accumAboveCoveredLayers for next (lower) layer
    coverage.aboveCoveredLayers.orSelf(visibleRegion);

    if (CC_UNLIKELY(computeAboveCoveredExcludingOverlays)) {
        coveredRegionExcludingDisplayOverlays =
                coverage.aboveCoveredLayersExcludingOverlays->intersect(visibleRegion);
        coverage.aboveCoveredLayersExcludingOverlays->orSelf(visibleRegion);
    }

    // subtract the opaque region covered by the layers above us
    visibleRegion.subtractSelf(coverage.aboveOpaqueLayers);

    if (visibleRegion.isEmpty()) {
        return;
    }

    // Get coverage information for the layer as previously displayed,
    // also taking over ownership from mOutputLayersorderedByZ.
    auto prevOutputLayerIndex = findCurrentOutputLayerForLayer(layerFE);
    auto prevOutputLayer =
            prevOutputLayerIndex ? getOutputLayerOrderedByZByIndex(*prevOutputLayerIndex) : nullptr;

    //  Get coverage information for the layer as previously displayed
    // TODO(b/121291683): Define kEmptyRegion as a constant in Region.h
    const Region kEmptyRegion;
    const Region& oldVisibleRegion =
            prevOutputLayer ? prevOutputLayer->getState().visibleRegion : kEmptyRegion;
    const Region& oldCoveredRegion =
            prevOutputLayer ? prevOutputLayer->getState().coveredRegion : kEmptyRegion;

    // compute this layer's dirty region
    Region dirty;
    if (layerFEState->contentDirty) {
        // we need to invalidate the whole region
        dirty = visibleRegion;
        // as well, as the old visible region
        dirty.orSelf(oldVisibleRegion);
    } else {
        /* compute the exposed region:
         *   the exposed region consists of two components:
         *   1) what's VISIBLE now and was COVERED before
         *   2) what's EXPOSED now less what was EXPOSED before
         *
         * note that (1) is conservative, we start with the whole visible region
         * but only keep what used to be covered by something -- which mean it
         * may have been exposed.
         *
         * (2) handles areas that were not covered by anything but got exposed
         * because of a resize.
         *
         */
        const Region newExposed = visibleRegion - coveredRegion;
        const Region oldExposed = oldVisibleRegion - oldCoveredRegion;
        dirty = (visibleRegion & oldCoveredRegion) | (newExposed - oldExposed);
    }
    dirty.subtractSelf(coverage.aboveOpaqueLayers);

    // accumulate to the screen dirty region
    coverage.dirtyRegion.orSelf(dirty);

    // Update accumAboveOpaqueLayers for next (lower) layer
    coverage.aboveOpaqueLayers.orSelf(opaqueRegion);

    // Compute the visible non-transparent region
    Region visibleNonTransparentRegion = visibleRegion.subtract(transparentRegion);

    // Perform the final check to see if this layer is visible on this output
    // TODO(b/121291683): Why does this not use visibleRegion? (see outputSpaceVisibleRegion below)
    const auto& outputState = getState();
    Region drawRegion(outputState.transform.transform(visibleNonTransparentRegion));
    drawRegion.andSelf(outputState.displaySpace.getBoundsAsRect());
    if (drawRegion.isEmpty()) {
        return;
    }

    Region visibleNonShadowRegion = visibleRegion.subtract(shadowRegion);

    // The layer is visible. Either reuse the existing outputLayer if we have
    // one, or create a new one if we do not.
    // 传入 layerFE 用于创建 OutputLayer
    auto result = ensureOutputLayer(prevOutputLayerIndex, layerFE);

    // Store the layer coverage information into the layer state as some of it
    // is useful later.
    auto& outputLayerState = result->editState();
    outputLayerState.visibleRegion = visibleRegion;
    outputLayerState.visibleNonTransparentRegion = visibleNonTransparentRegion;
    outputLayerState.coveredRegion = coveredRegion;
    outputLayerState.outputSpaceVisibleRegion = outputState.transform.transform(
            visibleNonShadowRegion.intersect(outputState.layerStackSpace.getContent()));
    outputLayerState.shadowRegion = shadowRegion;
    outputLayerState.outputSpaceBlockingRegionHint =
            layerFEState->compositionType == Composition::DISPLAY_DECORATION
            ? outputState.transform.transform(
                      transparentRegion.intersect(outputState.layerStackSpace.getContent()))
            : Region();
    if (CC_UNLIKELY(computeAboveCoveredExcludingOverlays)) {
        outputLayerState.coveredRegionExcludingDisplayOverlays =
                std::move(coveredRegionExcludingDisplayOverlays);
    }
}
```


#### Output::present


```c++
ftl::Future<std::monostate> Output::present(
        const compositionengine::CompositionRefreshArgs& refreshArgs) {
    const auto stringifyExpectedPresentTime = [this, &refreshArgs]() -> std::string {
        return ftl::Optional(getDisplayId())
                .and_then(PhysicalDisplayId::tryCast)
                .and_then([&refreshArgs](PhysicalDisplayId id) {
                    return refreshArgs.frameTargets.get(id);
                })
                .transform([](const auto& frameTargetPtr) {
                    return frameTargetPtr.get()->expectedPresentTime();
                })
                .transform([](TimePoint expectedPresentTime) {
                    return base::StringPrintf(" vsyncIn %.2fms",
                                              ticks<std::milli, float>(expectedPresentTime -
                                                                       TimePoint::now()));
                })
                .or_else([] {
                    // There is no vsync for this output.
                    return std::make_optional(std::string());
                })
                .value();
    };
    ATRACE_FORMAT("%s for %s%s", __func__, mNamePlusId.c_str(),
                  stringifyExpectedPresentTime().c_str());
    ALOGV(__FUNCTION__);
	// 根据 refresh 里面的参数更新颜色空间等（像素点表示颜色的编码方式），存储在 OutputState
    updateColorProfile(refreshArgs);
    // 关注点1 调用所有 OutputLayer 的 updateCompositionState
    updateCompositionState(refreshArgs);
    planComposition();
    // 关注点2 将一部分合成状态写到 hwc，这里通过 HWCLayer，将状态同步到HWC侧，也是通过aidl的binder
    writeCompositionState(refreshArgs);
    // 更新颜色转换矩阵，类似于layer的Transform，只是这里是针对于颜色的。同时会更新state里面的脏区
    setColorTransform(refreshArgs);
    // 这里会调用RenderSurface的beginFrame，后会调用DisplaySurface的beginFrame，如果是虚拟屏则会有一些刷新hwc的buffer相关的处理，如果不是虚拟盘什么也没做
    beginFrame();

    GpuCompositionResult result;
    // 判断是否同步进行
    const bool predictCompositionStrategy = canPredictCompositionStrategy(refreshArgs);

	// 无论是否同步最后都会走到 finishPrepareFrame，然后也是调用 renderSurface 的 prepareFrame，再调用 DisplaySurface 的 prepareFrame。如果是主屏这里也什么都没做
    // 如果是 prepareFrameAsync，会调用 dequeueRenderBuffer 和 composeSurfaces 进行合成操作，如果是 prepareFrame，合成操作在 finishFrame
    // 合成操作就在 finishFrame 里再介绍

    if (predictCompositionStrategy) {
        result = prepareFrameAsync();
    } else {
        prepareFrame();
    }

    devOptRepaintFlash(refreshArgs);
    // 关注点4
    // 如果之前走的是 prepareFrame，会在这里进行合成操作，调用 dequeueRenderBuffer 和 composeSurfaces
    finishFrame(std::move(result));
    ftl::Future<std::monostate> future;
    // 关注点5
    // 主要调用了 presentAndGetFrameFences 将剩余的 layer 提交给 HWC，让 HWC 进行硬件合成。除此之外会处理一些 buffer 的释放相关工作
    if (mOffloadPresent) {
        future = presentFrameAndReleaseLayersAsync();

        // Only offload for this frame. The next frame will determine whether it
        // needs to be offloaded. Leave the HwcAsyncWorker in place. For one thing,
        // it is currently presenting. Further, it may be needed next frame, and
        // we don't want to churn.
        mOffloadPresent = false;
    } else {
        presentFrameAndReleaseLayers();
        future = ftl::yield<std::monostate>({});
    }
    renderCachedSets(refreshArgs);
    return future;
}
```


- 关注点1，调用所有 OutputLayer 的 updateCompositionState，更新 OutputLayer 的 state 的部分成员
- 关注点2，将一部分合成状态写到 writer 中
- 关注点3，prepareFrame 完成部分 OutputLayer device 合成的同时从 HWC HAL 中获取到 OutputLayer 们的合成方式
- 关注点4，finishFrame，如果需要，完成 OutputLayer 的 client 合成
- 关注点5，postFramebuffer 完成后续所有 OutputLayer 的 device 合成


##### updateCompositionState

逐一遍历所有的 layers 的 updateCompositeState 方法

```c++
void Output::updateCompositionState(const compositionengine::CompositionRefreshArgs& refreshArgs) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);

    if (!getState().isEnabled) {
        return;
    }

    mLayerRequestingBackgroundBlur = findLayerRequestingBackgroundComposition();
    bool forceClientComposition = mLayerRequestingBackgroundBlur != nullptr;

    for (auto* layer : getOutputLayersOrderedByZ()) {
        layer->updateCompositionState(refreshArgs.updatingGeometryThisFrame,
                                      refreshArgs.devOptForceClientComposition ||
                                              forceClientComposition,
                                      refreshArgs.internalDisplayRotationFlags);

        if (mLayerRequestingBackgroundBlur == layer) {
            forceClientComposition = false;
        }
    }

    updateCompositionStateForBorder(refreshArgs);
}
```

##### writeCompositionState

将一部分合成状态写到 writer 中
```c++
void Output::writeCompositionState(const compositionengine::CompositionRefreshArgs& refreshArgs) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);

    if (!getState().isEnabled) {
        return;
    }

    if (auto frameTargetPtrOpt = ftl::Optional(getDisplayId())
                                         .and_then(PhysicalDisplayId::tryCast)
                                         .and_then([&refreshArgs](PhysicalDisplayId id) {
                                             return refreshArgs.frameTargets.get(id);
                                         })) {
        editState().earliestPresentTime = frameTargetPtrOpt->get()->earliestPresentTime();
        editState().expectedPresentTime = frameTargetPtrOpt->get()->expectedPresentTime().ns();
    }
    editState().frameInterval = refreshArgs.frameInterval;
    editState().powerCallback = refreshArgs.powerCallback;

    compositionengine::OutputLayer* peekThroughLayer = nullptr;
    sp<GraphicBuffer> previousOverride = nullptr;
    bool includeGeometry = refreshArgs.updatingGeometryThisFrame;
    uint32_t z = 0;
    bool overrideZ = false;
    uint64_t outputLayerHash = 0;
    // 逐一遍历所有的 layers并通过writeStateToHWC写入到 hwc 中。
    for (auto* layer : getOutputLayersOrderedByZ()) {
        if (layer == peekThroughLayer) {
            // No longer needed, although it should not show up again, so
            // resetting it is not truly needed either.
            peekThroughLayer = nullptr;

            // peekThroughLayer was already drawn ahead of its z order.
            continue;
        }
        bool skipLayer = false;
        const auto& overrideInfo = layer->getState().overrideInfo;
        if (overrideInfo.buffer != nullptr) {
            if (previousOverride && overrideInfo.buffer->getBuffer() == previousOverride) {
                ALOGV("Skipping redundant buffer");
                skipLayer = true;
            } else {
                // First layer with the override buffer.
                if (overrideInfo.peekThroughLayer) {
                    peekThroughLayer = overrideInfo.peekThroughLayer;

                    // Draw peekThroughLayer first.
                    overrideZ = true;
                    includeGeometry = true;
                    constexpr bool isPeekingThrough = true;
                    peekThroughLayer->writeStateToHWC(includeGeometry, false, z++, overrideZ,
                                                      isPeekingThrough);
                    outputLayerHash ^= android::hashCombine(
                            reinterpret_cast<uint64_t>(&peekThroughLayer->getLayerFE()),
                            z, includeGeometry, overrideZ, isPeekingThrough,
                            peekThroughLayer->requiresClientComposition());
                }

                previousOverride = overrideInfo.buffer->getBuffer();
            }
        }

        constexpr bool isPeekingThrough = false;
        layer->writeStateToHWC(includeGeometry, skipLayer, z++, overrideZ, isPeekingThrough);
        if (!skipLayer) {
            outputLayerHash ^= android::hashCombine(
                    reinterpret_cast<uint64_t>(&layer->getLayerFE()),
                    z, includeGeometry, overrideZ, isPeekingThrough,
                    layer->requiresClientComposition());
        }
    }
    editState().outputLayerHash = outputLayerHash;
}
```

##### prepareFrame

`Output::prepareFrame` 是合成流程中的**“决策中心”**。它的核心任务是完成 **SurfaceFlinger 与 HWC 硬件之间的“谈判”**，确定每一层到底由谁来渲染。

```c++
void Output::prepareFrame() {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);

    auto& outputState = editState();
    // 如果输出设备（屏幕）未启用，直接退出。
    if (!outputState.isEnabled) {
        return;
    }

    std::optional<android::HWComposer::DeviceRequestedChanges> changes;
    // 它会把当前所有的图层信息发送给 HWC 硬件。HWC 会返回一个 `changes`（DeviceRequestedChanges）。这个 `changes` 包含了硬件的反馈
    bool success = chooseCompositionStrategy(&changes);
    // 状态重置
    resetCompositionStrategy();
    outputState.strategyPrediction = CompositionStrategyPredictionState::DISABLED;
    outputState.previousDeviceRequestedChanges = changes;
    outputState.previousDeviceRequestedSuccess = success;
    // 应用策略
    if (success) {
        applyCompositionStrategy(changes);
    }
    // 这一步会进行最后的清理和计算，例如确定哪些区域是“脏”的（Dirty Region），需要重新绘制 cs.android.com。
    finishPrepareFrame();
}
```


##### finishFrame


该方法主要是用于将非 HWC (Client) 图层进行渲染提交到 HWC 中


其实不难发现主要的执行流程有几步：
- dequeueRenderBuffer： 它是从 `RenderSurface` 的缓冲队列（BufferQueue）中申请的一块 **Gralloc 内存**。当 HWC 无法处理所有图层，需要 GPU 介入时，GPU 需要一个地方来绘制那些标记为 `Client` 的层.
- composeSurfaces：它执行的是 **“局部拼图”**，将所有 **非 HWC (Client)** 图层画到上面那个空白 Buffer 里。
- queueBuffer：它入队的是 **“已经画好 Client 内容的成品图”**。

```c++
void Output::finishFrame(GpuCompositionResult&& result) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
    const auto& outputState = getState();
    // 前置条件过滤
    if (!outputState.isEnabled) {
        return;
    }

    std::optional<base::unique_fd> optReadyFence;
    std::shared_ptr<renderengine::ExternalTexture> buffer;
    base::unique_fd bufferFence;
    // 根据策略执行操作
    if (outputState.strategyPrediction == CompositionStrategyPredictionState::SUCCESS) {
        optReadyFence = std::move(result.fence);
    } else {
	    // 大概率会走到如下分支。
	    // 设置变量
        if (result.bufferAvailable()) {
            buffer = std::move(result.buffer);
            bufferFence = std::move(result.fence);
        } else {
            updateProtectedContentState();
            //  1.取Buffer
            if (!dequeueRenderBuffer(&bufferFence, &buffer)) {
                return;
            }
        }
        // Repaint the framebuffer (if needed), getting the optional fence for when
        // the composition completes.
		// 2.进行图像的合成（GPU 合成的图像）
        optReadyFence = composeSurfaces(Region::INVALID_REGION, buffer, bufferFence);
    }
    if (!optReadyFence) {
        return;
    }

    if (isPowerHintSessionEnabled()) {
        // get fence end time to know when gpu is complete in display
        setHintSessionGpuFence(
                std::make_unique<FenceTime>(sp<Fence>::make(dup(optReadyFence->get()))));
    }
    // swap buffers (presentation)
    // 3.入队 Buffer
    mRenderSurface->queueBuffer(std::move(*optReadyFence), getHdrSdrRatio(buffer));
}
```

- Output::dequeueRenderBuffer 
```c++
// 内部通过调用mRenderSurface->dequeueBuffer获取ExternalTexture，并将其设置到入参 tex 中。
bool Output::dequeueRenderBuffer(base::unique_fd* bufferFence,
                                 std::shared_ptr<renderengine::ExternalTexture>* tex) {
    const auto& outputState = getState();

    // If we aren't doing client composition on this output, but do have a
    // flipClientTarget request for this frame on this output, we still need to
    // dequeue a buffer.
    if (outputState.usesClientComposition || outputState.flipClientTarget) {
		// 调用dequeueBuffer获取 Buffer
        *tex = mRenderSurface->dequeueBuffer(bufferFence);
        if (*tex == nullptr) {
            ALOGW("Dequeuing buffer for display [%s] failed, bailing out of "
                  "client composition for this frame",
                  mName.c_str());
            return false;
        }
    }
    return true;
}


// 调用mNativeWindow->dequeueBuffer获取 Buffer，将其转为renderengine::impl::ExternalTexture返回。
std::shared_ptr<renderengine::ExternalTexture> RenderSurface::dequeueBuffer(
        base::unique_fd* bufferFence) {
    ATRACE_CALL();
    int fd = -1;
    ANativeWindowBuffer* buffer = nullptr;

    status_t result = mNativeWindow->dequeueBuffer(mNativeWindow.get(), &buffer, &fd);
	// 如果出现异常，直接返回
    if (result != NO_ERROR) {
        ALOGE("ANativeWindow::dequeueBuffer failed for display [%s] with error: %d",
              mDisplay.getName().c_str(), result);
        // Return fast here as we can't do much more - any rendering we do
        // now will just be wrong.
        return mTexture;
    }
	// 将 GraphicBuffer 将其转为renderengine::impl::ExternalTexture并返回
    ALOGW_IF(mTexture != nullptr, "Clobbering a non-null pointer to a buffer [%p].",
             mTexture->getBuffer()->getNativeBuffer()->handle);

    sp<GraphicBuffer> newBuffer = GraphicBuffer::from(buffer);

    std::shared_ptr<renderengine::ExternalTexture> texture;

    for (auto it = mTextureCache.begin(); it != mTextureCache.end(); it++) {
        const auto& cachedTexture = *it;
        if (cachedTexture->getBuffer()->getId() == newBuffer->getId()) {
            texture = cachedTexture;
            mTextureCache.erase(it);
            break;
        }
    }

    if (texture) {
        mTexture = texture;
    } else {
	    // 创建ExternalTexture
        mTexture = std::make_shared<
                renderengine::impl::ExternalTexture>(GraphicBuffer::from(buffer),
                                                     mCompositionEngine.getRenderEngine(),
                                                     renderengine::impl::ExternalTexture::Usage::
                                                             WRITEABLE);
    }
    mTextureCache.push_back(mTexture);
    if (mTextureCache.size() > mMaxTextureCacheSize) {
        mTextureCache.erase(mTextureCache.begin());
    }

    *bufferFence = base::unique_fd(fd);

    return mTexture;
}
```

- Output::composeSurfaces

主要流程可分为两个步骤：
1.调用generateClientCompositionRequests生成合成所需要的信息
2.调用drawLayers进行图层的合成最终将其渲染到 Buffer(tex 纹理)中
```c++
std::optional<base::unique_fd> Output::composeSurfaces(
        const Region& debugRegion, std::shared_ptr<renderengine::ExternalTexture> tex,
        base::unique_fd& fd) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);

    const auto& outputState = getState();
    const TracedOrdinal<bool> hasClientComposition = {
        base::StringPrintf("hasClientComposition %s", mNamePlusId.c_str()),
        outputState.usesClientComposition};
    if (!hasClientComposition) {
        setExpensiveRenderingExpected(false);
        return base::unique_fd();
    }

    if (tex == nullptr) {
        ALOGW("Buffer not valid for display [%s], bailing out of "
              "client composition for this frame",
              mName.c_str());
        return {};
    }

    ALOGV("hasClientComposition");

    renderengine::DisplaySettings clientCompositionDisplay =
            generateClientCompositionDisplaySettings(tex);

    // Generate the client composition requests for the layers on this output.
    //  1.将 output layers 做遍历，最后生产std::vector<LayerFE::LayerSettings>用于后续渲染
    auto& renderEngine = getCompositionEngine().getRenderEngine();
    const bool supportsProtectedContent = renderEngine.supportsProtectedContent();
    std::vector<LayerFE*> clientCompositionLayersFE;
    std::vector<LayerFE::LayerSettings> clientCompositionLayers =
            generateClientCompositionRequests(supportsProtectedContent,
                                              clientCompositionDisplay.outputDataspace,
                                              clientCompositionLayersFE);
    appendRegionFlashRequests(debugRegion, clientCompositionLayers);

    OutputCompositionState& outputCompositionState = editState();
    // Check if the client composition requests were rendered into the provided graphic buffer. If
    // so, we can reuse the buffer and avoid client composition.
    if (mClientCompositionRequestCache) {
        if (mClientCompositionRequestCache->exists(tex->getBuffer()->getId(),
                                                   clientCompositionDisplay,
                                                   clientCompositionLayers)) {
            ATRACE_NAME("ClientCompositionCacheHit");
            outputCompositionState.reusedClientComposition = true;
            setExpensiveRenderingExpected(false);
            // b/239944175 pass the fence associated with the buffer.
            return base::unique_fd(std::move(fd));
        }
        ATRACE_NAME("ClientCompositionCacheMiss");
        mClientCompositionRequestCache->add(tex->getBuffer()->getId(), clientCompositionDisplay,
                                            clientCompositionLayers);
    }

    // We boost GPU frequency here because there will be color spaces conversion
    // or complex GPU shaders and it's expensive. We boost the GPU frequency so that
    // GPU composition can finish in time. We must reset GPU frequency afterwards,
    // because high frequency consumes extra battery.
    const bool expensiveRenderingExpected =
            std::any_of(clientCompositionLayers.begin(), clientCompositionLayers.end(),
                        [outputDataspace =
                                 clientCompositionDisplay.outputDataspace](const auto& layer) {
                            return layer.sourceDataspace != outputDataspace;
                        });
    if (expensiveRenderingExpected) {
        setExpensiveRenderingExpected(true);
    }

    std::vector<renderengine::LayerSettings> clientRenderEngineLayers;
    clientRenderEngineLayers.reserve(clientCompositionLayers.size());
    std::transform(clientCompositionLayers.begin(), clientCompositionLayers.end(),
                   std::back_inserter(clientRenderEngineLayers),
                   [](LayerFE::LayerSettings& settings) -> renderengine::LayerSettings {
                       return settings;
                   });

    const nsecs_t renderEngineStart = systemTime();
	// 2.绘制 layers 到 tex 中
    auto fenceResult = renderEngine
                               .drawLayers(clientCompositionDisplay, clientRenderEngineLayers, tex,
                                           std::move(fd))
                               .get();

    if (mClientCompositionRequestCache && fenceStatus(fenceResult) != NO_ERROR) {
        // If rendering was not successful, remove the request from the cache.
        mClientCompositionRequestCache->remove(tex->getBuffer()->getId());
    }

    const auto fence = std::move(fenceResult).value_or(Fence::NO_FENCE);

    if (auto timeStats = getCompositionEngine().getTimeStats()) {
        if (fence->isValid()) {
            timeStats->recordRenderEngineDuration(renderEngineStart,
                                                  std::make_shared<FenceTime>(fence));
        } else {
            timeStats->recordRenderEngineDuration(renderEngineStart, systemTime());
        }
    }

    for (auto* clientComposedLayer : clientCompositionLayersFE) {
        clientComposedLayer->setWasClientComposed(fence);
    }

    return base::unique_fd(fence->dup());
}
```

- RenderSurface::queueBuffer
将入队到Buffer对应的 **FramebufferSurface** 的 `BufferQueue` 中。
物理流向：CompositionEngine (生产者)  ->RenderSurface -> BufferQueue -> HWC / Display HAL (消费者)。

```c++
void RenderSurface::queueBuffer(base::unique_fd readyFence, float hdrSdrRatio) {
    auto& state = mDisplay.getState();

    if (state.usesClientComposition || state.flipClientTarget) {
        // hasFlipClientTargetRequest could return true even if we haven't
        // dequeued a buffer before. Try dequeueing one if we don't have a
        // buffer ready.
        if (mTexture == nullptr) {
            ALOGI("Attempting to queue a client composited buffer without one "
                  "previously dequeued for display [%s]. Attempting to dequeue "
                  "a scratch buffer now",
                  mDisplay.getName().c_str());
            // We shouldn't deadlock here, since mTexture == nullptr only
            // after a successful call to queueBuffer, or if dequeueBuffer has
            // never been called.
            base::unique_fd unused;
            dequeueBuffer(&unused);
        }

        if (mTexture == nullptr) {
            ALOGE("No buffer is ready for display [%s]", mDisplay.getName().c_str());
        } else {
	        // 通过mNativeWindow将 Buffer 入队。
            status_t result = mNativeWindow->queueBuffer(mNativeWindow.get(),
                                                         mTexture->getBuffer()->getNativeBuffer(),
                                                         dup(readyFence));
            if (result != NO_ERROR) {
                ALOGE("Error when queueing buffer for display [%s]: %d", mDisplay.getName().c_str(),
                      result);
                // We risk blocking on dequeueBuffer if the primary display failed
                // to queue up its buffer, so crash here.
                if (!mDisplay.isVirtual()) {
                    LOG_ALWAYS_FATAL("ANativeWindow::queueBuffer failed with error: %d", result);
                } else {
                    mNativeWindow->cancelBuffer(mNativeWindow.get(),
                                                mTexture->getBuffer()->getNativeBuffer(),
                                                dup(readyFence));
                }
            }

            mTexture = nullptr;
        }
    }

    status_t result = mDisplaySurface->advanceFrame(hdrSdrRatio);
    if (result != NO_ERROR) {
        ALOGE("[%s] failed pushing new frame to HWC: %d", mDisplay.getName().c_str(), result);
    }
}
```

##### Output::presentFrameAndReleaseLayers

该流程通过 Hidl 执行所有 HWC layers（包含前一步 finishFrame 中的输出）的合成操作。

- Output::presentFrameAndReleaseLayers
```c++
void Output::presentFrameAndReleaseLayers() {
    ATRACE_FORMAT("%s for %s", __func__, mNamePlusId.c_str());
    ALOGV(__FUNCTION__);

    if (!getState().isEnabled) {
        return;
    }

    auto& outputState = editState();
    outputState.dirtyRegion.clear();

    auto frame = presentFrame();

    mRenderSurface->onPresentDisplayCompleted();

    for (auto* layer : getOutputLayersOrderedByZ()) {
        // The layer buffer from the previous frame (if any) is released
        // by HWC only when the release fence from this frame (if any) is
        // signaled.  Always get the release fence from HWC first.
        sp<Fence> releaseFence = Fence::NO_FENCE;

        if (auto hwcLayer = layer->getHwcLayer()) {
            if (auto f = frame.layerFences.find(hwcLayer); f != frame.layerFences.end()) {
                releaseFence = f->second;
            }
        }

        // If the layer was client composited in the previous frame, we
        // need to merge with the previous client target acquire fence.
        // Since we do not track that, always merge with the current
        // client target acquire fence when it is available, even though
        // this is suboptimal.
        // TODO(b/121291683): Track previous frame client target acquire fence.
        if (outputState.usesClientComposition) {
            releaseFence =
                    Fence::merge("LayerRelease", releaseFence, frame.clientTargetAcquireFence);
        }
        layer->getLayerFE()
                .onLayerDisplayed(ftl::yield<FenceResult>(std::move(releaseFence)).share(),
                                  outputState.layerFilter.layerStack);
    }

    // We've got a list of layers needing fences, that are disjoint with
    // OutputLayersOrderedByZ.  The best we can do is to
    // supply them with the present fence.
    for (auto& weakLayer : mReleasedLayers) {
        if (const auto layer = weakLayer.promote()) {
            layer->onLayerDisplayed(ftl::yield<FenceResult>(frame.presentFence).share(),
                                    outputState.layerFilter.layerStack);
        }
    }

    // Clear out the released layers now that we're done with them.
    mReleasedLayers.clear();
}
```

- Display::presentFrame
```c++
compositionengine::Output::FrameFences Display::presentFrame() {
    auto fences = impl::Output::presentFrame();

    const auto halDisplayIdOpt = HalDisplayId::tryCast(mId);
    if (mIsDisconnected || !halDisplayIdOpt) {
        return fences;
    }

    auto& hwc = getCompositionEngine().getHwComposer();

    const TimePoint startTime = TimePoint::now();

    if (isPowerHintSessionEnabled() && getState().earliestPresentTime) {
        mPowerAdvisor->setHwcPresentDelayedTime(mId, *getState().earliestPresentTime);
    }

	// 执行 HWC layers 的合成
    hwc.presentAndGetReleaseFences(*halDisplayIdOpt, getState().earliestPresentTime);

    if (isPowerHintSessionEnabled()) {
        mPowerAdvisor->setHwcPresentTiming(mId, startTime, TimePoint::now());
    }

    fences.presentFence = hwc.getPresentFence(*halDisplayIdOpt);

    // TODO(b/121291683): Change HWComposer call to return entire map
    for (const auto* layer : getOutputLayersOrderedByZ()) {
        auto hwcLayer = layer->getHwcLayer();
        if (!hwcLayer) {
            continue;
        }

        fences.layerFences.emplace(hwcLayer, hwc.getLayerReleaseFence(*halDisplayIdOpt, hwcLayer));
    }

    hwc.clearReleaseFences(*halDisplayIdOpt);

    return fences;
}
```
- HWComposer::presentAndGetReleaseFences
```c++
status_t HWComposer::presentAndGetReleaseFences(
        HalDisplayId displayId,
        std::optional<std::chrono::steady_clock::time_point> earliestPresentTime) {
    ATRACE_CALL();

    RETURN_IF_INVALID_DISPLAY(displayId, BAD_INDEX);

    auto& displayData = mDisplayData[displayId];
    auto& hwcDisplay = displayData.hwcDisplay;

    if (displayData.validateWasSkipped) {
        // explicitly flush all pending commands
        auto error = static_cast<hal::Error>(mComposer->executeCommands(hwcDisplay->getId()));
        RETURN_IF_HWC_ERROR_FOR("executeCommands", error, displayId, UNKNOWN_ERROR);
        RETURN_IF_HWC_ERROR_FOR("present", displayData.presentError, displayId, UNKNOWN_ERROR);
        return NO_ERROR;
    }

    if (earliestPresentTime) {
        ATRACE_NAME("wait for earliest present time");
        std::this_thread::sleep_until(*earliestPresentTime);
    }
	// 进一步调用 hwDisplay 执行hwc layers 的合成
    auto error = hwcDisplay->present(&displayData.lastPresentFence);
    RETURN_IF_HWC_ERROR_FOR("present", error, displayId, UNKNOWN_ERROR);

    std::unordered_map<HWC2::Layer*, sp<Fence>> releaseFences;
    error = hwcDisplay->getReleaseFences(&releaseFences);
    RETURN_IF_HWC_ERROR_FOR("getReleaseFences", error, displayId, UNKNOWN_ERROR);

    displayData.releaseFences = std::move(releaseFences);

    return NO_ERROR;
}
```
- Display::present
```c++
Error Display::present(sp<Fence>* outPresentFence)
{
    int32_t presentFenceFd = -1;
    auto intError = mComposer.presentDisplay(mId, &presentFenceFd);
    auto error = static_cast<Error>(intError);
    if (error != Error::NONE) {
        return error;
    }

    *outPresentFence = sp<Fence>::make(presentFenceFd);
    return Error::NONE;
}
```
- HidlComposer
通过 Hidl 执行HWC layer 的渲染和合成
```c++
Error HidlComposer::presentDisplay(Display display, int* outPresentFence) {
    ATRACE_NAME("HwcPresentDisplay");
    mWriter.selectDisplay(display);
    mWriter.presentDisplay();

    Error error = execute();
    if (error != Error::NONE) {
        return error;
    }

    mReader.takePresentFence(display, outPresentFence);

    return Error::NONE;
}
```

# 总结




# Refs

[Android14 显示系统剖 10 ———— SurfaceFlinger 图层合成过程分析上](https://juejin.cn/post/7410657936714432547?searchId=202602121027186FBDEB3F8D19A7E16462)
[Android14 显示系统剖 10 ———— SurfaceFlinger 图层合成过程分析中](https://juejin.cn/post/7410670277308678159?searchId=202602121027186FBDEB3F8D19A7E16462)
[Android14 显示系统剖 10 ———— SurfaceFlinger 图层合成过程分析下](https://juejin.cn/post/7410616997006639144?searchId=20260212102845D7BAD652187929F2A7F6)
[Android14 显示系统剖析5 ———— BLASTBufferQueue 初始化](https://juejin.cn/post/7397424623959080998)
