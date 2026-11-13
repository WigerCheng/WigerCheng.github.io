---
title: Android Jetpack LiveData
tags:
  - LiveData
  - Android/Jetpack
draft: true
---

>[!info] 什么是LiveData
>
>LiveData 是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 activity、fragment 或 service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。


- livedata 内部维护了两个变量
	- `mData`：值
	- `mVersion`：版本
- livedata的方法除了`postValue`其他都要在主线程调用



## Observer和ObserverWrapper

LiveData的Observer接口，实现它来接收从LiveData的数据。

```kotlin
/**
 * A simple callback that can receive from [LiveData].
 *
 * @see LiveData LiveData - for a usage description.
 */
public fun interface Observer<T> {

    /** Called when the data is changed to [value]. */
    public fun onChanged(value: T)
}

```

LiveData内部ObserverWrapper，是一个Observer容器的抽象类。

| 变量/方法                                      | 作用             |
| ------------------------------------------ | -------------- |
| boolean mActive                            | LiveData是否还在活跃 |
| int mLastVersion                           | LiveData的数据版本号 |
| abstract boolean shouldBeActive()          | Observer时候应该活跃 |
| boolean isAttachedTo(LifecycleOwner owner) | 是否已经附在owner    |
| detachObserver()                           | 分离（移除）监听       |
|                                            |                |

### LiveData两种监听
### 1. observe（和生命周期绑定）

```java
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }
```

1. 先判断是否在主线程，否则抛出异常
2. 判断owner的生命周期是否已经Destroyed，是就停止。


