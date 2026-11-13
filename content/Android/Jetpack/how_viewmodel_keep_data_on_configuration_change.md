---
title: ViewModel 是如何在配置变化时保持数据不丢失的？
tags:
  - ViewModel
  - Android/Jetpack
---

## ViewModel描述

ViewModel 是一个用于为 Activity 和 Fragment 提供数据准备与管理的类。同时，它也负责处理 Activity/Fragment 与应用其他部分之间的通信，例如调用业务逻辑等。

当一个 ViewModel 被创建时，它通常会与一个 Scope（如 Activity 或 Fragment）关联；只要这个 Scope 仍然存活，ViewModel 就会继续保留其数据。例如，如果它关联的是一个 Activity，那么它会一直存在，直到该 Activity 结束。同时这意味着，若其 Owner 因为配置发生改变（如旋转屏幕）而被销毁的时候，ViewModel不会随之被销毁。新的 Owner 实例会重新连接到现有的ViewModel。

## ComponentActivity实现方式

### 1.实现ViewModelStoreOwner接口

ComponentActivity实现了[[viewmodelstore#ViewModelStoreOwner|ViewModelStoreOwner]]接口实现了`viewModelStore`属性。

首先通过`getApplication`方法来判断Activity是否已经成功附加到Application实例（即已经执行完onCreate而不是一个普通的类实例）。随后调用`ensureViewModelStore`来确保`_viewModelStore`初始化。最后返回该`_viewModelStore`。

这里是延迟初始化，如果没有调用`getViewModelStore`的话，`_viewModelStore`一直为null。

```kotlin

    init {
        lifecycle.addObserver(
            object : LifecycleEventObserver {
                override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
                    ensureViewModelStore()
                    lifecycle.removeObserver(this)
                }
            }
        )
    }

    override val viewModelStore: ViewModelStore
        get() {
            checkNotNull(application) {
                ("Your activity is not yet attached to the " +
                    "Application instance. You can't request ViewModel before onCreate call.")
            }
            ensureViewModelStore()
            return _viewModelStore!!
        }

    private fun ensureViewModelStore() {
        if (_viewModelStore == null) {
            val nc = lastNonConfigurationInstance as NonConfigurationInstances?
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                _viewModelStore = nc.viewModelStore
            }
            if (_viewModelStore == null) {
                _viewModelStore = ViewModelStore()
            }
        }
    }    
```

上面`ensureViewModelStore`方法中，最重要的部分就是通过`getLastNonConfigurationInstance()`方法获取NonConfigurationInstances，然后从中取得先前的ViewModelStore来实现新的owner重用之前的实例。

### 2. 实现重用ViewModelStore实例

在`getLastNonConfigurationInstance`的方法描述中，我们可以知道，它允许取回`onRetainNonConfigurationInstance()`方法返回的非配置实例数据。

而 `onRetainNonConfigurationInstance()` 的方法说明则表明：当 Activity 因配置变化（如屏幕旋转）而被销毁时，系统会在 onStop() 到 onDestroy() 这段期间调用这个方法。我们可以通过它返回任意希望保留下来的对象实例；在配置变化后，系统会根据新的配置立即创建新的 Activity 实例，稍后我们就可以通过 `getLastNonConfigurationInstance() `在新 Activity 中使用这些保留下来的数据。

```kotlin
internal class NonConfigurationInstances {  
    var custom: Any? = null  
    var viewModelStore: ViewModelStore? = null  
}

    /**
     * Retain all appropriate non-config state. You can NOT override this yourself! Use a
     * [androidx.lifecycle.ViewModel] if you want to retain your own non config state.
     */
    @Suppress("deprecation")
    final override fun onRetainNonConfigurationInstance(): Any? {
        // Maintain backward compatibility.
        val custom = onRetainCustomNonConfigurationInstance()
        var viewModelStore = _viewModelStore
        if (viewModelStore == null) {
            // No one called getViewModelStore(), so see if there was an existing
            // ViewModelStore from our last NonConfigurationInstance
            val nc = lastNonConfigurationInstance as NonConfigurationInstances?
            if (nc != null) {
                viewModelStore = nc.viewModelStore
            }
        }
        if (viewModelStore == null && custom == null) {
            return null
        }
        val nci = NonConfigurationInstances()
        nci.custom = custom
        nci.viewModelStore = viewModelStore
        return nci
    }
```

从 ComponentActivity#onRetainNonConfigurationInstance() 的实现看，ComponentActivity 会先检查 _viewModelStore 是否已存在；若不存在，则尝试从 getLastNonConfigurationInstance() 恢复之前保存的 ViewModelStore。如果两种方式都没有结果，就返回 null；否则返回一个包含 viewModelStore 的 NonConfigurationInstances 实例。

### 3. 实现当Activity销毁时执行clear

通过注册生命周期变化监听，当Activity处于ON_DESTROY并且不是发生配置变化则执行`viewModelStore.clear()`方法

```kotlin
lifecycle.addObserver(  
    LifecycleEventObserver { _, event ->  
        if (event == Lifecycle.Event.ON_DESTROY) {  
            // Clear out the available context  
            contextAwareHelper.clearAvailableContext()  
            // And clear the ViewModelStore  
            if (!isChangingConfigurations) {  
                viewModelStore.clear()  
            }  
            reportFullyDrawnExecutor.activityDestroyed()  
        }  
    }  
)
```

## 总结

综上，ViewModel 能够在配置变化时继续保留数据，核心在于它并不是直接绑定在某个 Activity 实例上，而是通过 ViewModelStore 与 Activity 的生命周期进行关联。在配置变化发生时，系统会保留对应的 ViewModelStore，并将其重新绑定给新的 Activity 实例，从而实现数据的持续保持。
