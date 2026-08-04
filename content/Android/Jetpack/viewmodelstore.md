---
title: ViewModelStore分析
tags:
  - Android/Jetpack
---
## ViewModelStore

ViewModelStore，顾名思义就是存储ViewModel的类，也可以称它为ViewModel容器。ViewModelStore 通过内部维护一个 Map 集合，用于管理 ViewModel 实例。

维护一个ViewModelStore的实例必须处理两种情况： ^90194a

1. 当一个Owner（通常是ViewModelStoreOwner）在配置发生变化的时候触发了销毁和重建的时候，新的owner必须重用之前的ViewModelStore实例。
2. 当Owner将被永久销毁不会重建的时候，它应该调用[[#clear|clear()]]方法去通知容器内的所有viewModel实例执行`clear`方法。

> [!warning]
>
> 虽然ViewModelStore这个类可见性是open，但是是不应该被继承的。

### put

>[!important]
>
> 值得注意的是，执行put添加viewModel实例的时候，ViewModelStore会同时根据key去取对应的ViewModel实例，若取到的旧实例不为null，将执行它的clear方法。

```kotlin
    /**
     * Stores [viewModel] under [key], replacing any existing entry.
     *
     * If a [ViewModel] is already stored for [key], it is removed and immediately cleared.
     */
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public fun put(key: Any?, viewModel: ViewModel) {
        val oldViewModel = map.put(key, viewModel)
        oldViewModel?.clear()
    }
```

### clear

清空容器并通知所有持有的ViewModel实例执行clear方法。

```kotlin
    /**
     * Clears this store and notifies all stored [ViewModel] instances that they are no longer used.
     *
     * After this call, the store is empty.
     *
     * @see ViewModel.onCleared
     */
    public fun clear() {
        val snapshot = map.toMap()
        map.clear()
        for (viewModel in snapshot.values) {
            viewModel.clear()
        }
    }
```

## ViewModelStoreOwner

ViewModelStoreOwner，顾名思义就是ViewModelStore的持有者。它的主要功能是持有并管理 ViewModelStore，并要处理好上述的[[#^90194a|两种情况]]。

它提供了一个属性`viewModelStore`供Activity、Fragment访问ViewModelStore。

```kotlin
/**
 * A scope that owns [ViewModelStore].
 *
 * A responsibility of an implementation of this interface is to retain owned ViewModelStore during
 * the configuration changes and call [ViewModelStore.clear], when this scope is going to be
 * destroyed.
 */
public interface ViewModelStoreOwner {

    /** The owned [ViewModelStore] */
    public val viewModelStore: ViewModelStore
}

```