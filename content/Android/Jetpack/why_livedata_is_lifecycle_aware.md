---
tags:
  - LiveData
  - Lifecycle
  - Android/Jetpack
title: LiveData 是如何实现生命周期感知的？
---

## LiveData 简介

LiveData 是一种可观察的数据容器。和普通的观察者模式不同，LiveData 具备生命周期感知能力：它会感知 Activity、Fragment、Service 等组件的生命周期，并且只向处于活跃状态的观察者分发数据。

当观察者对应的 `LifecycleOwner` 处于 `STARTED` 或 `RESUMED` 状态时，LiveData 会认为这个观察者是活跃的；如果生命周期低于 `STARTED`，则认为它是非活跃的。LiveData 只会把数据变化通知给活跃观察者，非活跃观察者不会收到更新。

同时，当 `LifecycleOwner` 进入 `DESTROYED` 状态时，LiveData 会自动移除对应的观察者。因此在 Activity 或 Fragment 中使用 `observe(owner, observer)` 时，一般不需要手动取消监听，也不容易因为页面销毁后继续持有 observer 而造成内存泄漏。

## 怎么实现生命周期感知

LiveData 的生命周期感知能力，本质上不是 `Observer` 自己具备生命周期能力，而是 LiveData 在内部用 `ObserverWrapper` 包装了外部传入的 `Observer`。

当我们使用 `observe(owner, observer)` 注册观察者时，LiveData 会创建一个 `LifecycleBoundObserver`。它既是 `ObserverWrapper`，也是生命周期监听者。这样，LiveData 就可以在生命周期变化时重新判断 observer 是否应该处于活跃状态，并只向活跃 observer 分发数据。

### 1. ObserverWrapper

`ObserverWrapper` 是 LiveData 内部对 Observer 的包装。无论是和生命周期绑定的 `LifecycleBoundObserver`，还是通过 `observeForever` 注册的 `AlwaysActiveObserver`，它们都继承自 `ObserverWrapper`。

```java
private abstract class ObserverWrapper {
    final Observer<? super T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;

    ObserverWrapper(Observer<? super T> observer) {
        mObserver = observer;
    }

    abstract boolean shouldBeActive();

    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }

    void detachObserver() {
    }

    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            return;
        }
        mActive = newActive;
        changeActiveCounter(mActive ? 1 : -1);
        if (mActive) {
            dispatchingValue(this);
        }
    }
}
```

`ObserverWrapper` 中有几个关键字段/方法：

| 字段/方法              | 作用                                                                             |
| ---------------------- | -------------------------------------------------------------------------------- |
| `mObserver`            | 外部真正传入的 Observer                                                          |
| `mActive`              | 当前 observer 是否处于活跃状态                                                   |
| `mLastVersion`         | observer 已经接收过的数据版本                                                    |
| `shouldBeActive()`     | 判断当前 observer 是否应该处于活跃状态                                           |
| `activeStateChanged()` | 当活跃状态变化时，更新 LiveData 的活跃 observer 数量，并在变为活跃时尝试分发数据 |

当 Observer 的活跃状态发生变化时，会调用 `activeStateChanged()`：

这段代码做了三件事：

1. 如果活跃状态没有变化，直接返回。
2. 如果状态变化了，就更新当前 wrapper 的 `mActive`，并通过 `changeActiveCounter()` 修改 LiveData 内部的活跃 observer 数量。
3. 如果 observer 从非活跃变成活跃，就调用 `dispatchingValue(this)` 尝试把最新的数据分发给它。

其中 `changeActiveCounter()` 会维护 LiveData 内部的 `mActiveCount`。当活跃 observer 数量从 0 变成 1 时，会回调 `onActive()`；当活跃 observer 数量从 1 变成 0 时，会回调 `onInactive()`。

```java
@MainThread
protected void onActive() {
}

@MainThread
protected void onInactive() {
}
```

这也是自定义 LiveData 时常用的扩展点。例如，当第一个界面开始观察数据时，在 `onActive()` 中注册系统监听、开启定位、连接数据源；当没有任何活跃 observer 时，在 `onInactive()` 中释放资源。

需要注意的是，`onInactive()` 表示当前没有活跃 observer，并不一定表示没有 observer。比如一个 Activity 进入后台后，它的 observer 仍然注册在 LiveData 中，只是由于生命周期低于 `STARTED`，所以暂时不活跃。

### 2. observe 和 LifecycleBoundObserver

我们通过调用 `LiveData#observe` 传入 `LifecycleOwner` 和 `Observer`，开始监听 LiveData 数据的变化。

```java
    @MainThread
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

这段代码的关键在于：LiveData 并不是直接把外部传入的 `observer` 注册到 Lifecycle 中，而是先创建 `LifecycleBoundObserver`，然后将它注册给 `owner.getLifecycle()`。

`observe()` 的整体流程如下：

1. 检查当前是否在主线程调用。
2. 如果 owner 当前已经是 `DESTROYED` 状态，直接忽略本次注册。
3. 创建 `LifecycleBoundObserver`，把 `owner` 和 `observer` 包装起来。
4. 通过 `mObservers.putIfAbsent(observer, wrapper)` 记录 observer 和 wrapper 的映射关系。
5. 如果同一个 observer 已经绑定过其他 `LifecycleOwner`，则抛出异常。
6. 最后调用 `owner.getLifecycle().addObserver(wrapper)`，让 wrapper 监听生命周期变化。

`LifecycleBoundObserver` 继承 `ObserverWrapper`，并实现 `LifecycleEventObserver#onStateChanged`，专门用于处理 `LifecycleOwner` 的生命周期变化。

```java
    class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {

        final @NonNull LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
            super(observer);
            mOwner = owner;
        }

        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                Lifecycle.@NonNull Event event) {
            Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
            if (currentState == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            Lifecycle.State prevState = null;
            while (prevState != currentState) {
                prevState = currentState;
                activeStateChanged(shouldBeActive());
                currentState = mOwner.getLifecycle().getCurrentState();
            }
        }

        @Override
        boolean isAttachedTo(LifecycleOwner owner) {
            return mOwner == owner;
        }

        @Override
        void detachObserver() {
            mOwner.getLifecycle().removeObserver(this);
        }

    }

```

```java
//LiveData.java

    /**
     * Removes the given observer from the observers list.
     *
     * @param observer The Observer to receive events.
     */
    @MainThread
    public void removeObserver(final @NonNull Observer<? super T> observer) {
        assertMainThread("removeObserver");
        ObserverWrapper removed = mObservers.remove(observer);
        if (removed == null) {
            return;
        }
        removed.detachObserver();
        removed.activeStateChanged(false);
    }
```

1. 如果 owner 当前的生命周期是 `DESTROYED`，则执行 `LiveData#removeObserver`，从 `mObservers` 中移除当前 observer，并通过 `detachObserver()` 从 Lifecycle 中移除当前生命周期监听。
2. 如果 owner 当前的生命周期不是 `DESTROYED`，则通过 while 循环反复读取当前生命周期状态，并调用 `activeStateChanged(shouldBeActive())` 更新 observer 的活跃状态。
3. `shouldBeActive()` 内部通过 `mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED)` 判断当前生命周期是否至少处于 `STARTED` 状态。也就是说，只有 `STARTED` 和 `RESUMED` 会被视为活跃状态。

这里的 while 循环看起来有些特殊，它的作用是处理生命周期状态在回调过程中再次发生变化的情况。每轮循环都会重新读取一次 `currentState`，直到前后两次读取到的状态一致，才认为这次状态同步完成。

当 observer 从非活跃变成活跃时，`activeStateChanged(true)` 会进一步调用 `dispatchingValue(this)`。也就是说，页面从后台回到前台时，如果 LiveData 中已经有了新数据，observer 会立刻收到最新值。

### 3. 数据为什么只分发给活跃 observer

LiveData 在真正通知 observer 前，会通过 `considerNotify()` 再做一次判断：

```java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    observer.mObserver.onChanged((T) mData);
}
```

这段逻辑是 LiveData 能做到“只通知活跃观察者”的最后一道判断：

1. 如果 observer 当前不是活跃状态，直接返回。
2. 如果 observer 标记为活跃，但根据最新生命周期判断已经不应该活跃，则先切换为非活跃，再停止通知。
3. 如果 observer 已经接收过当前版本的数据，也不会重复通知。
4. 只有通过上面这些检查后，才会调用外部 observer 的 `onChanged()`。

因此，LiveData 不是在 `setValue()` 后无差别通知所有 observer，而是在分发前结合 `mActive`、`shouldBeActive()` 和 `mLastVersion` 做了多重判断。

### 4. observeForever 和 AlwaysActiveObserver

除了 `observe(owner, observer)`，LiveData 还提供了 `observeForever(observer)`。

`observeForever` 不需要传入 `LifecycleOwner`，因此它不会感知 Activity 或 Fragment 的生命周期。通过这种方式注册的 observer 会被 LiveData 认为是永远活跃的。只要 LiveData 有新数据，它就会收到通知。

```java
@MainThread
public void observeForever(@NonNull Observer<? super T> observer) {
    assertMainThread("observeForever");
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing instanceof LiveData.LifecycleBoundObserver) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    wrapper.activeStateChanged(true);
}
```

`observeForever` 的流程和 `observe` 很像，但有几个关键差异：

1. 它创建的是 `AlwaysActiveObserver`，不是 `LifecycleBoundObserver`。
2. 它不会调用 `owner.getLifecycle().addObserver(wrapper)`，因为根本没有 LifecycleOwner。
3. 它会直接调用 `wrapper.activeStateChanged(true)`，把 observer 设置成活跃状态。
4. 如果同一个 observer 已经通过 `observe(owner, observer)` 注册过，再调用 `observeForever(observer)` 会抛出异常。

其中 `AlwaysActiveObserver` 的实现非常简单：

```java
private class AlwaysActiveObserver extends ObserverWrapper {

    AlwaysActiveObserver(Observer<? super T> observer) {
        super(observer);
    }

    @Override
    boolean shouldBeActive() {
        return true;
    }
}
```

它的核心就是 `shouldBeActive()` 永远返回 `true`。因此在 `considerNotify()` 中，只要这个 observer 没有被移除，并且当前数据版本比它上一次接收的版本新，就会收到 `onChanged()` 回调。

这也是为什么 `observeForever` 注册的 observer 必须手动移除：

```java
liveData.observeForever(observer);

// 不再需要观察时，必须主动移除
liveData.removeObserver(observer);
```

如果不调用 `removeObserver(observer)`，LiveData 会一直持有这个 observer。**只要 observer 间接引用了 Activity、Fragment、View 或其他短生命周期对象，就可能造成内存泄漏。**

## observe 和 observeForever 的区别

| 方法                       | 是否绑定 LifecycleOwner | 是否自动移除                      | 是否一直活跃                        | 适合场景                                                         |
| -------------------------- | ----------------------- | --------------------------------- | ----------------------------------- | ---------------------------------------------------------------- |
| `observe(owner, observer)` | 是                      | owner 进入 `DESTROYED` 时自动移除 | 否，只有 `STARTED`/`RESUMED` 时活跃 | Activity、Fragment 观察 UI 数据                                  |
| `observeForever(observer)` | 否                      | 否，需要手动 `removeObserver`     | 是                                  | Repository、全局监听、桥接其他数据源等需要明确管理生命周期的场景 |

一般情况下，在 Activity 或 Fragment 中应该优先使用 `observe(owner, observer)`。只有当观察者本身不依赖界面生命周期，并且能明确控制移除时，才适合使用 `observeForever`。

## 总结

LiveData 实现生命周期感知的核心，并不是让数据本身拥有生命周期，而是把外部传入的 `Observer` 包装成内部的 `ObserverWrapper`。

当使用 `observe(owner, observer)` 时，LiveData 会创建 `LifecycleBoundObserver`，并把它注册到 owner 的 Lifecycle 中。每次生命周期变化时，`LifecycleBoundObserver` 都会根据当前状态是否至少为 `STARTED` 来决定 observer 是否活跃。当 owner 进入 `DESTROYED` 时，LiveData 会自动移除这个 observer，从而避免 Activity 或 Fragment 销毁后继续收到数据，也减少内存泄漏风险。

当 observer 从非活跃变成活跃时，LiveData 会调用 `dispatchingValue()` 尝试分发最新数据；真正分发前，又会通过 `considerNotify()` 再次检查 observer 是否活跃、生命周期是否仍然满足条件、数据版本是否已经接收过。因此 LiveData 能保证只有活跃 observer 才会收到更新。

而 `observeForever` 走的是另一条路径。它使用 `AlwaysActiveObserver` 包装 observer，`shouldBeActive()` 永远返回 `true`，所以它不感知任何 LifecycleOwner，也不会被自动移除。使用它时必须手动调用 `removeObserver()`，否则 LiveData 会一直持有 observer。

因此，LiveData 的生命周期感知能力可以概括为三点：生命周期绑定负责自动管理 observer 的注册与移除，活跃状态负责控制数据是否分发，版本号负责避免重复通知。
