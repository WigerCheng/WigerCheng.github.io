---
title: Android 自定义 View 进阶：实现 View 滑动的五种核心方案与原理解析
date: 2026-08-12T12:10:00
tags:
  - Android/View
---

# Android 自定义 View 进阶：实现 View 滑动的五种核心方案与原理解析

在 Android 交互设计中，滑动是一个随处可见的效果。无论是下拉刷新、滑动卡片，还是侧滑菜单，其核心都是 **View 的滑动**。

要实现 View 的滑动，其基本原理是相似的：**当手指触摸屏幕时，记录触摸点坐标；在手指移动时，计算偏移量，并通过修改 View 的坐标或内容实现位置更新。** 

本文将为你深度拆解实现 View 滑动的五种主流方案，探讨它们的实现细节与底层原理。

---

## 方案一：使用 `layout()` 方法直接重新布局

View 在绘制时会经历 `measure`、`layout` 和 `draw` 流程。在 `layout` 阶段，系统会确定 View 的四个顶点。因此，我们可以通过直接修改 View 的 `left`、`top`、`right`、`bottom` 四个边界值，来改变它的位置。

### 1. 代码实现
我们可以自定义一个 View，在 `onTouchEvent` 中获取每次移动的偏移量，然后调用 `layout()` 方法：

```kotlin
class CustomView @JvmOverloads constructor(  
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0  
) : View(context, attrs, defStyleAttr) {  
  
    private var lastX: Int = 0  
    private var lastY: Int = 0  
  
    override fun onTouchEvent(event: MotionEvent): Boolean {  
        // 获取当前触摸点相对于控件自身的坐标
        val x = event.x.toInt()  
        val y = event.y.toInt()  
  
        when (event.action) {  
            MotionEvent.ACTION_DOWN -> {  
                lastX = x  
                lastY = y  
            }  
  
            MotionEvent.ACTION_MOVE -> {  
                // 计算偏移量
                val offsetX = x - lastX  
                val offsetY = y - lastY  
                // 重新设定 View 在父容器中的边界
                layout(  
                    left + offsetX,  
                    top + offsetY,  
                    right + offsetX,  
                    bottom + offsetY  
                )  
            }  
        }  
        return true  
    }  
}
```

---

## 方案二：使用快捷方法 `offsetLeftAndRight()` 与 `offsetTopAndBottom()`

如果你觉得每次都要算四个边界值并调用 `layout()` 稍微有点繁琐，Android 提供了一对高度封装的辅助方法：`offsetLeftAndRight(dx)` 和 `offsetTopAndBottom(dy)`。

其原理与直接调用 `layout()` 类似，都是直接修改 View 的位置参数。

### 1. 代码实现
```kotlin
class CustomView @JvmOverloads constructor(  
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0  
) : View(context, attrs, defStyleAttr) {  
  
    private var lastX: Int = 0  
    private var lastY: Int = 0  
  
    override fun onTouchEvent(event: MotionEvent): Boolean {  
        val x = event.x.toInt()  
        val y = event.y.toInt()  
  
        when (event.action) {  
            MotionEvent.ACTION_DOWN -> {  
                lastX = x  
                lastY = y  
            }  
  
            MotionEvent.ACTION_MOVE -> {  
                val offsetX = x - lastX  
                val offsetY = y - lastY  
                // 快捷偏移 View 的物理边界
                offsetLeftAndRight(offsetX)  
                offsetTopAndBottom(offsetY)  
            }  
        }  
        return true  
    }  
}
```

---

## 方案三：通过 View 提供的 `scrollTo` 与 `scrollBy`

这是 Android 官方为 View 内置的滑动 API，也是最常用的滑动机制之一。

- **`scrollTo(x, y)`**：将 View **内容**滑向指定的绝对坐标（属于**绝对滑动**）。
- **`scrollBy(dx, dy)`**：基于当前位置滑动指定的位移量（属于**相对滑动**）。其源码实现也是通过调用 `scrollTo`：
  ```java
  public void scrollBy(int x, int y) {  
      scrollTo(mScrollX + x, mScrollY + y);  
  }
  ```

> [!WARNING]
> **`scrollTo` 和 `scrollBy` 移动的是 View 中的内容（即子元素或绘制图层），而不是 View 自身。**
> 例如，如果对一个 Button 调用 `scrollBy`，你会发现 Button 依然在原位，但它上面的文字跑偏了；如果对一个 LinearLayout 调用，则会移动其所有的子视图。

### 1. 坐标方向的“反直觉”原理
当你调用 `scrollBy(100, 0)` 时，你会惊奇地发现 View 的内容**向左**移动了，而不是向右。
- **原理**：`mScrollX` 和 `mScrollY` 代表的是 **View 的边缘**与 **View 内容的边缘**在坐标轴上的距离差值。
- 从左往右滑（手指右滑，内容右移）：`mScrollX` 为**负值**。
- 从上往下滑（手指下滑，内容下移）：`mScrollY` 为**负值**。

![Scroll 坐标移动图解](view2.png)

### 2. 实战：跟着手势滑动 `window.decorView`
以下是一个经典的手势滑动页面示例，通过改变根视图（DecorView）的 scroll 来实现内容跟手：

```kotlin
override fun onTouchEvent(event: MotionEvent): Boolean {  
    val x = event.x.roundToInt()  
    val y = event.y.roundToInt()  
    val decorView = window.decorView  
  
    when (event.action) {  
        MotionEvent.ACTION_DOWN -> {  
            mLastX = x  
            mLastY = y  
        }  
        MotionEvent.ACTION_MOVE -> {  
            // 倒减是因为我们要让手指滑动的方向与内容移动方向一致
            val dx = mLastX - x  
            val dy = mLastY - y  
            val oldScrollX = decorView.scrollX  
            val oldScrollY = decorView.scrollY  
            decorView.scrollTo(oldScrollX + dx, oldScrollY + dy)  
            mLastX = x  
            mLastY = y  
        }  
    }  
    return true  
}
```

---

## 方案四：通过平移动画（Animation / Animator）

如果你想实现平滑的过度，或者非常复杂的曲线滑动，动画是最佳选择。

### 1. 传统补间动画 (TranslateAnimation)
通过动画让 View 发生位移。但传统的补间动画**只会改变 View 的渲染画面，不会真正改变 View 的物理点击位置**。
```kotlin
val anim = TranslateAnimation(0f, 100f, 0f, 100f).apply {
    duration = 500
    fillAfter = true // 动画结束后保持状态
}
view.startAnimation(anim)
```

### 2. 属性动画 (ObjectAnimator / ViewPropertyAnimator)
属性动画是真正修改 View 的属性值的。通过改变 `translationX` 和 `translationY`，不仅能实现平滑滑动，还能确保 View 的**点击区域随之移动**。
```kotlin
view.animate()
    .translationX(100f)
    .translationY(100f)
    .setDuration(500)
    .start()
```

---

## 方案五：修改布局参数 LayoutParams

View 的大小和排放位置都遵循父容器提供的 `LayoutParams` 规则。如果我们动态修改 View 的 `LayoutParams`（例如修改 Margin 间距），也可以强迫 View 重新执行布局流程，从而发生滑动。

### 1. 代码实现
例如，通过给一个 View 的左侧边距（leftMargin）累加 100 像素，将其向右平移：

```kotlin
binding.button1.setOnClickListener {  
    binding.view1.updateLayoutParams<MarginLayoutParams> {  
        leftMargin += 100  
    }  
}
```

---

## 总结：如何挑选最佳滑动方案？

| 方案 | 移动目标 | 物理点击区域随之改变？ | 适用场景 |
| :--- | :--- | :--- | :--- |
| **`layout() / offset...`** | View 自身 | 是 | 适合即时跟手的拖拽型自定义 View（无需过度动画） |
| **`scrollTo / scrollBy`** | View 内容 | 是 (仅限内容本身) | 适合 ViewGroup 内容滚动（如 ScrollView、ViewPager、下拉刷新容器） |
| **属性动画 (Animator)** | View 自身 | 是 | 适合需要精美过渡动画的 UI 交互（如展开收起、弹窗飞入） |
| **修改 LayoutParams** | View 自身 | 是 | 适合有交互且依赖父容器排版的 View（如位置被强绑定的普通控件） |

### 💡 附：实现一个全屏跟手移动的自定义 View 最简方案
在自定义控件时，我们也可以结合属性动画中的 `translationX / translationY` 来快速实现一个完美支持点击的跟手拖拽 View：

```kotlin
class CustomMoveView @JvmOverloads constructor(  
    context: Context,  
    attrs: AttributeSet? = null,  
    defStyleAttr: Int = 0,  
) : View(context, attrs, defStyleAttr) {  
  
    private var mLastX = 0  
    private var mLastY = 0  
  
    override fun onTouchEvent(event: MotionEvent): Boolean {  
        // 使用绝对坐标，避免移动时造成坐标基准抖动
        val x = event.rawX.roundToInt()  
        val y = event.rawY.roundToInt()  
  
        when (event.action) {  
            MotionEvent.ACTION_MOVE -> {  
                val deltaX = x - mLastX  
                val deltaY = y - mLastY  
                // 直接改变 translation 属性，物理点击位置会自动跟随
                translationX += deltaX  
                translationY += deltaY  
            }  
        }  
        mLastX = x  
        mLastY = y  
        return true  
    }  
}
```