---
title: Android Canvas 绘图艺术（三）：文字测量与精细化排版指南
date: 2026-08-12T12:10:00
tags:
  - Android/View
---

# Android Canvas 绘图艺术（三）：文字测量与精细化排版指南

在 Android 自定义 View 开发中，绘制基础图形和路径相对直观。然而，当你需要在屏幕上绘制文本，并且要求文本**精准对齐、水平居中或垂直居中**时，事情就变得复杂起来了。

如果你直接使用 `canvas.drawText("Text", x, y, paint)`，你会发现文字的 `y` 轴并不是文字的中心，甚至不是文字的底部，而是文字的 **Baseline（基线）**。 

为了实现精细化的文字排版，我们必须深入理解 Android 的文字测量体系——**`FontMetrics`**。

---

## 一、 认识文字测量的基石：FontMetrics

在绘制文字时，Android 系统会根据当前设置的字体（Typeface）和字号（TextSize）为文字分配一组虚拟的参考线。这组参考线可以通过 `Paint.getFontMetrics()` 获得，它们被统称为 **`FontMetrics`**。

![文字测量线图解](text_measure.png)

如上图所示，文字从上到下共有五条关键线：

1. **Baseline（基线）**：文字绘制的物理基准线。`drawText()` 方法中传入的 `y` 坐标就是 Baseline 所在的 `y` 轴位置。
2. **Ascent（上坡度线）**：单个字符（不带声调符号等特殊延伸）所能达到的推荐最高高度。它是相对于 Baseline 的**相对距离**（由于在 Baseline 上方，所以其值通常为**负数**）。
3. **Descent（下坡度线）**：字符在 Baseline 下方推荐延伸的最低深度（如字母 `g`、`y` 的尾部）。它是相对于 Baseline 的距离（由于在 Baseline 下方，所以值通常为**正数**）。
4. **Top（最大高度线）**：当前字号下，字符在系统内部所能达到的物理最大上限（通常为包含音标、声调符号后的高度）。相对值，通常为**极小的负数**。
5. **Bottom（最大深度线）**：当前字号下，字符在系统内部所能延伸到的物理最大下限。相对值，通常为**正数**。

### FontMetrics 在代码中的获取方式
```kotlin
val fontMetrics = paint.fontMetrics
val top = fontMetrics.top        // 负数
val ascent = fontMetrics.ascent  // 负数
val descent = fontMetrics.descent// 正数
val bottom = fontMetrics.bottom  // 正数
val leading = fontMetrics.leading// 行间距
```

---

## 二、 文本尺寸的精确获取

在排版时，我们常常需要知道一段文字的真实宽度和高度。Android 提供了两种方式来计算。

### 1. 获取文本宽度
- **方式一（最常用）**：`paint.measureText(text)`
  直接返回文本在屏幕上渲染时所占用的宽度值。
  ```kotlin
  val textWidth = paint.measureText("Hello Android")
  ```

### 2. 获取文本高度与真实边界：`paint.getTextBounds`
当我们需要计算文字真实的视觉物理边界时，使用 `getTextBounds`。
- **方法签名**：
  ```kotlin
  paint.getTextBounds(text, 0, text.length, rect)
  ```
  该方法会把测量结果填充到传入的 `Rect` 矩形对象中。
  - `rect.width()`：文字的实际视觉宽度（通常比 `measureText` 略窄，因为去掉了左右两侧的字距留白）。
  - `rect.height()`：文字的实际视觉高度（即文字最高像素到最低像素的绝对距离）。

---

## 三、 实战：如何在矩形区域内完美居中文字？

在 UI 开发中，最经典的需求莫过于：**给定一个矩形的中心点坐标，让文字在这个中心点实现视觉上的绝对居中。**

### 1. 水平居中
水平居中非常简单，我们只需要将画笔的对齐方式设置为 `CENTER` 即可：
```kotlin
paint.textAlign = Paint.Align.CENTER
```
这样，在绘制时，我们传入矩形的中心 `centerX`，文字就会以自身中轴线对齐。

### 2. 垂直居中（推导公式）
因为 `drawText()` 的 `y` 坐标必须传入 Baseline 的高度，所以我们需要推导出：**当文字居中时，Baseline 的 y 坐标与矩形中心点 `centerY` 的关系。**

#### 视觉完美居中法（基于 Ascent / Descent）
如果我们要让字符主体部分（即不考虑过于极端的声调，以 Ascent 到 Descent 区域为基准）在 `centerY` 垂直居中：
- 字符主体区域的绝对高度：$$H = \text{descent} - \text{ascent}$$ *(因为 ascent 是负数，所以是减法)*
- 字符主体区域的中线距离 Baseline 的距离：$$\text{middleOffset} = \frac{\text{descent} - \text{ascent}}{2} - \text{descent} = \frac{-\text{ascent} - \text{descent}}{2}$$
- 因此，当文字垂直居中时，Baseline 的坐标应该在 `centerY` 的下方，其值为：
  $$\text{baselineY} = \text{centerY} + \frac{-(\text{ascent} + \text{descent})}{2}$$

#### 示例代码：
```kotlin
fun drawCenteredText(canvas: Canvas, text: String, rectF: RectF, paint: Paint) {
    // 1. 水平居中设置
    paint.textAlign = Paint.Align.CENTER
    val centerX = rectF.centerX()
    
    // 2. 测量计算垂直 Baseline y 坐标
    val centerY = rectF.centerY()
    val fontMetrics = paint.fontMetrics
    val baselineY = centerY - (fontMetrics.ascent + fontMetrics.descent) / 2
    
    // 3. 绘制文字
    canvas.drawText(text, centerX, baselineY, paint)
}
```

---

## 四、 总结

文字绘制是自定义 View 从“粗糙”迈向“精致”的必由之路。
- **Baseline** 是文字定位的根基。
- **FontMetrics** 提供了控制文字各条边界的精准数据。
- 在实际开发中，根据需求灵活运用 `measureText`、`getTextBounds` 以及 `FontMetrics.ascent / descent` 的计算公式，能够让你在任何复杂的 Canvas 场景下完美排版文本。