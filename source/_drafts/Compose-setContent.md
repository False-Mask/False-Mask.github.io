---
title: Compose SetContent原理解析
tags:
- Compose
cover:
---

 

# Compose SetContent



> 具体的步骤为
>
> 1. 确保ContentView为ComposeView（如果不是就创建ComposeView并设置）
> 2. 设置owners
> 3. 通过setContentView覆盖顶层父类

```kotlin
public fun ComponentActivity.setContent(
    parent: CompositionContext? = null,
    content: @Composable () -> Unit
) {
    val existingComposeView = window.decorView
        .findViewById<ViewGroup>(android.R.id.content)
        .getChildAt(0) as? ComposeView

    if (existingComposeView != null) with(existingComposeView) {
        setParentCompositionContext(parent)
        setContent(content)
    } else ComposeView(this).apply {
        // Set content and parent **before** setContentView
        // to have ComposeView create the composition on attach
        setParentCompositionContext(parent)
        setContent(content)
        // Set the view tree owners before setting the content view so that the inflation process
        // and attach listeners will see them already present
        setOwners()
        setContentView(this, DefaultActivityContentLayoutParams)
    }
}
```





# ComposeView



> 在父类的基础上做了一层包装，实现了设置、执行Composable函数的功能。

```kotlin
class ComposeView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : AbstractComposeView(context, attrs, defStyleAttr) {

    private val content = mutableStateOf<(@Composable () -> Unit)?>(null)

    @Suppress("RedundantVisibilityModifier")
    protected override var shouldCreateCompositionOnAttachedToWindow: Boolean = false
        private set

    @Composable
    override fun Content() {
        content.value?.invoke()
    }

    override fun getAccessibilityClassName(): CharSequence {
        return javaClass.name
    }

    /**
     * Set the Jetpack Compose UI content for this view.
     * Initial composition will occur when the view becomes attached to a window or when
     * [createComposition] is called, whichever comes first.
     */
    fun setContent(content: @Composable () -> Unit) {
        shouldCreateCompositionOnAttachedToWindow = true
        this.content.value = content
        if (isAttachedToWindow) {
            createComposition()
        }
    }
}
```





# AbstractComposeView



> 这个类实际上没做什么事情，只是在onAttach的时候创建了AndroidComposeView子View，并将onMeasure、onLayout方法委托给AndroidComposeView。





> 单单从类型申明上看就是一个简单的ViewGroup。
>
> 当然ViewGroup的特殊之处肯定在于重写了measure等绘制相关的方法。

```kotlin
abstract class AbstractComposeView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : ViewGroup(context, attrs, defStyleAttr) {
```



## onAttachedToWindow



> 1. 保存windowToken
> 2. 在必要时确保Composite创建

```kotlin
override fun onAttachedToWindow() {
        super.onAttachedToWindow()

        previousAttachedWindowToken = windowToken

        if (shouldCreateCompositionOnAttachedToWindow) {
            ensureCompositionCreated()
        }
}
```



> 确保Composite创建
>
> 将AndroidComposeView添加进ComposeView内

```kotlin
private fun ensureCompositionCreated() {
        if (composition == null) {
            try {
                creatingComposition = true
                composition = setContent(resolveParentCompositionContext()) {
                    Content()
                }
            } finally {
                creatingComposition = false
            }
        }
}

internal fun AbstractComposeView.setContent(
    parent: CompositionContext,
    content: @Composable () -> Unit
): Composition {
    GlobalSnapshotManager.ensureStarted()
    val composeView =
        if (childCount > 0) {
            getChildAt(0) as? AndroidComposeView
        } else {
            removeAllViews(); null
        } ?: AndroidComposeView(context, parent.effectCoroutineContext).also {
            addView(it.view, DefaultLayoutParams)
        }
    return doSetContent(composeView, parent, content)
}
```







## onMeasure



> 用于测量ViewGroup，But是调用的子View的测量方法。

```kotlin
final override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        ensureCompositionCreated()
    	// measure
        internalOnMeasure(widthMeasureSpec, heightMeasureSpec)
}

// 调用子view的measure
internal open fun internalOnMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val child = getChildAt(0)
        if (child == null) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec)
            return
        }

        val width = maxOf(0, MeasureSpec.getSize(widthMeasureSpec) - paddingLeft - paddingRight)
        val height = maxOf(0, MeasureSpec.getSize(heightMeasureSpec) - paddingTop - paddingBottom)
        child.measure(
            MeasureSpec.makeMeasureSpec(width, MeasureSpec.getMode(widthMeasureSpec)),
            MeasureSpec.makeMeasureSpec(height, MeasureSpec.getMode(heightMeasureSpec)),
        )
        setMeasuredDimension(
            child.measuredWidth + paddingLeft + paddingRight,
            child.measuredHeight + paddingTop + paddingBottom
        )
    }
```







## onLayout



> 同理对直接子View进行布局

```kotlin

final override fun onLayout(changed: Boolean, left: Int, top: Int, right: Int, bottom: Int) =
        internalOnLayout(changed, left, top, right, bottom)

    internal open fun internalOnLayout(
        changed: Boolean,
        left: Int,
        top: Int,
        right: Int,
        bottom: Int
    ) {
        getChildAt(0)?.layout(
            paddingLeft,
            paddingTop,
            right - left - paddingRight,
            bottom - top - paddingBottom
        )
    }
```







## onRtlPropertiesChanged



> 也没有什么好说的，也就是设置了layoutDirection

```kotlin
override fun onRtlPropertiesChanged(layoutDirection: Int) {
    // Force the single child for our composition to have the same LayoutDirection
    // that we do. We will get onRtlPropertiesChanged eagerly as the value changes,
    // but the composition child view won't until it measures. This can be too late
    // to catch the composition pass for that frame, so propagate it eagerly.
    getChildAt(0)?.layoutDirection = layoutDirection
}
```





## isTransitionGroup



> 

```kotlin
override fun isTransitionGroup(): Boolean = !isTransitionGroupSet || super.isTransitionGroup()
```





## addView

> 其实就是抛异常，Compose内不允许addView

```kotlin
 override fun addView(child: View?) {
        checkAddView()
        super.addView(child)
}

private fun checkAddView() {
        if (!creatingComposition) {
            throw UnsupportedOperationException(
                "Cannot add views to " +
                    "${javaClass.simpleName}; only Compose content is supported"
            )
        }
    }
```





## addViewInLayout



> 同上



## addViewInLayout

> 同上



## shouldDelayChildPressedState



> 为了兼容性覆写。

```kotlin
 override fun shouldDelayChildPressedState(): Boolean = false
```





# AndroidComposeView



## onAttachedToWindow



> 添加各种observer

```kotlin
override fun onAttachedToWindow() {
        super.onAttachedToWindow()
        invalidateLayoutNodeMeasurement(root)
        invalidateLayers(root)
        snapshotObserver.startObserving()
        ifDebug {
            if (autofillSupported()) {
                _autofill?.let { AutofillCallback.register(it) }
            }
        }

        val lifecycleOwner = findViewTreeLifecycleOwner()
        val savedStateRegistryOwner = findViewTreeSavedStateRegistryOwner()

        val oldViewTreeOwners = viewTreeOwners
        // We need to change the ViewTreeOwner if there isn't one yet (null)
        // or if either the lifecycleOwner or savedStateRegistryOwner has changed.
        val resetViewTreeOwner = oldViewTreeOwners == null ||
            (
                (lifecycleOwner != null && savedStateRegistryOwner != null) &&
                    (
                        lifecycleOwner !== oldViewTreeOwners.lifecycleOwner ||
                            savedStateRegistryOwner !== oldViewTreeOwners.lifecycleOwner
                        )
                )
        if (resetViewTreeOwner) {
            if (lifecycleOwner == null) {
                throw IllegalStateException(
                    "Composed into the View which doesn't propagate ViewTreeLifecycleOwner!"
                )
            }
            if (savedStateRegistryOwner == null) {
                throw IllegalStateException(
                    "Composed into the View which doesn't propagate" +
                        "ViewTreeSavedStateRegistryOwner!"
                )
            }
            oldViewTreeOwners?.lifecycleOwner?.lifecycle?.removeObserver(this)
            lifecycleOwner.lifecycle.addObserver(this)
            val viewTreeOwners = ViewTreeOwners(
                lifecycleOwner = lifecycleOwner,
                savedStateRegistryOwner = savedStateRegistryOwner
            )
            _viewTreeOwners = viewTreeOwners
            onViewTreeOwnersAvailable?.invoke(viewTreeOwners)
            onViewTreeOwnersAvailable = null
        }

        _inputModeManager.inputMode = if (isInTouchMode) Touch else Keyboard

        viewTreeOwners!!.lifecycleOwner.lifecycle.addObserver(this)
        viewTreeObserver.addOnGlobalLayoutListener(globalLayoutListener)
        viewTreeObserver.addOnScrollChangedListener(scrollChangedListener)
        viewTreeObserver.addOnTouchModeChangeListener(touchModeChangeListener)
    }
```









## onMeasure





## onLayout





## onDraw



















































