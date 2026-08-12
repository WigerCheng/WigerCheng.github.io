---
title: Android 实战：用淡入淡出动效（Crossfade）优化页面加载体验
tags:
  - Android/Animation
---

在移动应用开发中，“等待加载”是无法避免的用户体验环节。相比于生硬的界面直接切换，**淡入淡出（Crossfade）** 动画能够优雅地在“加载状态（骨架屏/Loading/占位图）”与“真实内容”之间做无缝过渡。

本文将带你通过一个实战案例，讲解如何在 Android 中实现完美的淡入淡出（Crossfade）切换动画，避免各种因视图重叠导致的交互“大坑”。

---

## 1. 什么是 Crossfade 动效？

淡入淡出（Crossfade）通常包含两个同时进行的视图透明度动画：
1. **淡出（Fade Out）**：将当前处于顶层的加载占位视图（如骨架屏、进度条或占位图）透明度从 1 逐渐减小到 0，并将其隐藏。
2. **淡入（Fade In）**：将处于底层的真实内容视图透明度从 0 逐渐增大到 1，让内容优雅地呈现在用户眼前。

这种双向过渡的动画可以产生“消隐交融”的视觉美感，使用户感受到的加载等待时间比实际要短。

![[fade_animate_gif.gif]]

---

## 2. 第一步：合理的 XML 布局设计

实现 Crossfade 的首要前提是**重叠布局**。我们需要通过 `FrameLayout` 或 `ConstraintLayout`，将占位图和内容图重叠叠放在同一个位置上。

注意：内容视图的默认能见度应当设为 `gone`，以免在数据未返回时占位显示冲突。

```xml
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <!-- 1. 真实内容视图（默认隐藏且透明度设为 0F） -->
    <ImageView
        android:id="@+id/v_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:scaleType="centerCrop"
        android:src="@drawable/profile_picture"
        android:visibility="gone" />

    <!-- 2. 加载占位视图（默认显示在最上层） -->
    <ImageView
        android:id="@+id/v_loading"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:scaleType="centerInside"
        android:src="@mipmap/ic_launcher" />
</FrameLayout>
```

---

## 3. 第二步：编写 Kotlin 动画逻辑

我们将使用 Android 现代且极其高效的 `ViewPropertyAnimator` 接口（即 `view.animate()`）来驱动动画。它能将多个属性的变化合并为单次重绘，拥有非常优秀的渲染性能。

```kotlin
import android.animation.Animator
import android.animation.AnimatorListenerAdapter
import android.os.Bundle
import android.view.View
import android.widget.ImageView
import androidx.appcompat.app.AppCompatActivity

class CrossfadeActivity : AppCompatActivity() {

    private lateinit var contentView: ImageView
    private lateinit var loadingView: ImageView
    private var shortAnimationDuration: Long = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_crossfade)

        contentView = findViewById(R.id.v_content)
        loadingView = findViewById(R.id.v_loading)

        // 读取系统默认的动画时长（config_longAnimTime，通常为 400ms 左右）
        shortAnimationDuration = resources.getInteger(android.R.integer.config_longAnimTime).toLong()
        
        // 模拟数据加载成功后触发切换
        contentView.postDelayed({
            executeCrossfade()
        }, 2000)
    }

    private fun executeCrossfade() {
        // 1. 内容视图淡入
        contentView.apply {
            // 将 visibility 设置为 VISIBLE，但保持 alpha 为 0，使其依然“隐形”
            visibility = View.VISIBLE
            alpha = 0F
            // 开始执行渐显动画
            animate()
                .alpha(1F)
                .setDuration(shortAnimationDuration)
                .setListener(null) // 必须清除历史 listener 干扰
        }

        // 2. 占位视图淡出
        loadingView.animate()
            .alpha(0F)
            .setDuration(shortAnimationDuration)
            .setListener(object : AnimatorListenerAdapter() {
                override fun onAnimationEnd(animation: Animator) {
                    // 动画结束时，务必将占位视图设为 GONE！
                    loadingView.visibility = View.GONE
                }
            })
    }
}
```

---

## 4. 核心避坑指南（关键技巧）

在实际开发中，如果不注意以下细节，淡入淡出动画往往会带来奇奇怪怪的 Bug：

### 🚨 避坑一：淡出后必须设置为 `GONE`，不能只设为 `INVISIBLE` 或仅为 `alpha = 0`
哪怕 View 的透明度是 0（或者 `INVISIBLE`），它在布局结构中仍然**占据空间且会拦截触摸事件**。如果淡出完成后不设置 `loadingView.visibility = View.GONE`，这个“透明”的 View 将继续盖在真实内容上面，导致底层的真实内容（如按钮、列表）**完全无法被点击**。

### 🚨 避坑二：清除 Animator 的 Listener 干扰
在对同一个 View 进行多次动画时，`animate().setListener(...)` 会将 Listener 缓存在 `ViewPropertyAnimator` 内部。所以我们在进行新的淡入淡出前，最好通过 `.setListener(null)` 清理一下可能残留的旧动画监听。
