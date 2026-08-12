---
title: 用 Android 圆形揭露动画（Circular Reveal）打造炫酷的页面特效
tags:
  - Android/Animation
---

在 Material Design 设计规范中，**圆形揭露动画（Circular Reveal Animation）** 是一种非常经典且具有极强视觉张力的动画效果。它能够以某个指定的坐标为圆心，将一个视图像波纹扩散一样逐渐剪裁揭露出来，或者反过来向内收缩隐藏。

本篇文章将带你全面掌握 `ViewAnimationUtils.createCircularReveal()` 的 API 使用技巧，并教你如何优雅地实现各种常见的揭露动效。

---

## 1. 什么是圆形揭露动画？

圆形揭露动画在 Android 5.0 (API 21) 及以上版本引入，主要通过 `ViewAnimationUtils.createCircularReveal()` 静态方法创建。

它的核心作用是：**为视图剪裁一个不断变化的圆形显示区域。**
- **显示过程**：圆形的半径从 0（或者很小的值）逐渐扩大，直到覆盖整个视图，让视图自然浮现。
- **隐藏过程**：圆形的半径从最大逐渐收缩到 0，最后将视图设为不可见（`GONE` 或 `INVISIBLE`）。

### 常见应用场景：
- **切换搜索框**：点击搜索按钮后，搜索栏从按钮处向两端圆形扩散展开。
- **悬浮按钮（FAB）扩展**：点击悬浮按钮，展开成一个全屏的菜单或新页面。
- **页面揭露转场**：从点击的坐标点开始，新页面逐渐扩散遮盖旧页面。

---

## 2. API 参数深度剖析

`ViewAnimationUtils.createCircularReveal()` 会返回一个标准并且操作简单的 `Animator`（实际上是一个 `ValueAnimator` 的子类 `RevealAnimator`），你可以像配置其他属性动画一样去配置它的时长、插值器及监听器。

它的签名如下：
```kotlin
fun createCircularReveal(
    view: View,         // 目标视图（要剪裁的 View）
    centerX: Int,       // 裁剪圆心的 X 坐标（相对于目标视图本身）
    centerY: Int,       // 裁剪圆心的 Y 坐标（相对于目标视图本身）
    startRadius: Float, // 动画开始时的圆形裁剪半径
    endRadius: Float    // 动画结束时的圆形裁剪半径
): Animator
```

### 关键点：如何计算最大半径（`endRadius`）？
如果我们的目标是让整个矩形 View 完全显示，那么最终的圆必须能够覆盖矩形的四个角。
因此，圆心到矩形最远顶点的距离，就是最完美的 `endRadius`。
数学上，这可以通过勾股定理（`hypot`）轻松算出：
```kotlin
// 圆心在 View 的中心点时，计算覆盖整个 View 的半径
val endRadius = kotlin.math.hypot(width / 2.0, height / 2.0).toFloat()
```

---

## 3. 完整代码示例：显隐切换

下面我们来看一个常见的需求：点击按钮，让一张图片从中心“圆形揭露”展开；再次点击，向内收缩隐藏。

![[circle_animation.gif|200]]

### 3.1 编写 Kotlin 实现代码
```kotlin
import android.animation.Animator
import android.animation.AnimatorListenerAdapter
import android.view.View
import android.view.ViewAnimationUtils
import android.widget.ImageView
import kotlin.math.hypot

class RevealAnimationHelper {

    /**
     * 以 View 的正中心为圆心，播放揭露显示动画
     */
    fun showWithReveal(view: View, durationMs: Long = 500L) {
        // 确保在主线程且 View 已完成测量
        view.post {
            val cx = view.width / 2
            val cy = view.height / 2

            // 计算对角线长度作为最终的剪裁半径
            val finalRadius = hypot(cx.toDouble(), cy.toDouble()).toFloat()

            // 将 View 设为可见，但初始状态半径为 0
            view.visibility = View.VISIBLE

            ViewAnimationUtils.createCircularReveal(view, cx, cy, 0F, finalRadius).apply {
                duration = durationMs
                start()
            }
        }
    }

    /**
     * 以 View 的正中心为圆心，播放收缩隐藏动画
     */
    fun hideWithReveal(view: View, durationMs: Long = 500L) {
        val cx = view.width / 2
        val cy = view.height / 2

        // 计算初始半径（即当前能覆盖整个 View 的半径）
        val initialRadius = hypot(cx.toDouble(), cy.toDouble()).toFloat()

        ViewAnimationUtils.createCircularReveal(view, cx, cy, initialRadius, 0F).apply {
            duration = durationMs
            // 必须在动画结束的回调中隐藏 View，避免画面闪烁
            addListener(object : AnimatorListenerAdapter() {
                override fun onAnimationEnd(animation: Animator) {
                    view.visibility = View.GONE
                }
            })
            start()
        }
    }
}
```

---

## 4. 进阶技巧与踩坑指南

1. **硬件加速**：圆形揭露动画在硬件加速下才能顺畅运行，如果发现动画异常或卡顿，请确认应用程序或 Activity 是否开启了硬件加速（Android 系统默认开启）。
2. **View 必须完成测量（Layout）**：如果你在 `onCreate` 中直接调用此动画，由于 View 的 `width` 和 `height` 还未被测量出来（均为 0），动画将无法正常播放。请务必使用 `view.post { ... }` 确保 View 已经就绪。
3. **圆心坐标的相对性**：`centerX` 和 `centerY` 是**相对于该 View 自身的左上角**的坐标，而不是相对于屏幕的坐标。如果是从别的按钮触发揭露动画，需要将按钮相对于屏幕的坐标转换为目标 View 的相对坐标。