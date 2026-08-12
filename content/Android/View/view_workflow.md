---
title: 深入浅出 Android View 工作流程：Measure、Layout 与 Draw 三部曲
date: 2026-08-12T12:10:00
tags:
  - Android/View
---

# 深入浅出 Android View 工作流程：Measure、Layout 与 Draw 三部曲

当我们在 Activity 中调用 `setContentView(layoutResID)` 时，Android 系统内部究竟发生了什么？一个 xml 布局文件是如何变成屏幕上鲜活的界面的？

这离不开 Android 系统中 View 最为核心的生命周期流程——**View 的工作流程**。简单来说，View 的工作流程主要由三大阶段组成：**Measure（测量）**、**Layout（布局）** 和 **Draw（绘制）**。它们以自顶向下的树状调用依次进行，最终决定了控件的大小、位置和外观。

---

## 一、 工作流程三大主线概览

| 步骤 | 对应方法 | 核心职责 | 决定了什么 |
| :--- | :--- | :--- | :--- |
| **1. Measure (测量)** | `measure()` -> `onMeasure()` | 确定 View 的测量宽度与测量高度。对于 ViewGroup，还要递归测量子 View。 | **尺寸** (宽/高) |
| **2. Layout (布局)** | `layout()` -> `onLayout()` | 确定 View 自身在父容器中的顶点坐标（Top/Left/Right/Bottom）。ViewGroup 在这里指定各个子 View 的摆放位置。 | **位置** (在屏幕哪儿) |
| **3. Draw (绘制)** | `draw()` -> `onDraw()` | 将 View 的内容绘制到 Canvas（画布）上。 | **外观** (画出什么样) |

---

## 二、 第一步：Measure（测量尺寸）

在 Android 中，一个控件的大小不是随便决定的，而是由**父容器的约束**与**自身期望的大小**共同商量决定的。这个商量的媒介就是 **`MeasureSpec`**。

### 1. 什么是 MeasureSpec？
`MeasureSpec` 代表一个 32 位的 int 值，高 2 位代表 **SpecMode（测量模式）**，低 30 位代表 **SpecSize（规格大小）**。
- **EXACTLY**（精确模式）：父容器已经检测出 View 所需的精确大小，此时 View 的最终大小就是 SpecSize。对应布局中的 `match_parent` 或具体数值（如 `100dp`）。
- **AT_MOST**（最大模式）：子容器可以是规格大小以内的任意大小。对应布局中的 `wrap_content`。
- **UNSPECIFIED**（未指定模式）：父容器不对 View 有任何限制，要多大给多大（多用于系统内部或 ScrollView 嵌套等特殊场景）。

### 2. 测量流程的流动
1. **ViewGroup**：调用 `measure()` 触发测量。ViewGroup 会遍历子 View，计算子 View 的 MeasureSpec 并调用子 View 的 `measure()`。
2. **View**：在 `onMeasure(widthMeasureSpec, heightMeasureSpec)` 中根据父容器给出的规格计算出自己的宽高，最后必须调用 `setMeasuredDimension(width, height)` 来保存测量宽高。

> [!TIP]
> 如果要写自定义 View 并支持 `wrap_content`，必须在 `onMeasure` 中针对 `AT_MOST` 模式给出一个默认值。否则，`wrap_content` 的大小会默认等同于父容器剩余的最大空间（等同于 `match_parent`）。

---

## 三、 第二步：Layout（确定位置）

确定好尺寸后，下一步就是把控件摆放在屏幕合适的位置上。

### 1. 布局原理
1. 父容器（ViewGroup）的 `layout()` 方法被调用。
2. 在 `layout()` 方法中，父容器通过 `setFrame()` 确定自己四个顶点的位置。
3. 接着调用 `onLayout()` 方法，ViewGroup 会遍历子 View，根据不同的排版逻辑（如线性排列、相对排列等），计算出子 View 的四个坐标顶点（left, top, right, bottom），然后调用子 View 的 `layout(l, t, r, b)` 方法。
4. 子 View 重复此步骤，直到整棵 View 树的所有节点位置都被确定。

### 2. layout() 与 onLayout() 的区别
- **`layout()`**：确定 View 自身的位置，由系统调用，不建议重写。
- **`onLayout()`**：确定子 View 的位置。在 `View` 中它是空实现；而在 `ViewGroup` 中它是一个**抽象方法**，所有自定义布局容器（如自定义 FlowLayout）都必须实现此方法以指定子元素的排放逻辑。

---

## 四、 第三步：Draw（内容绘制）

万事俱备，最后一步是把 View 的像素渲染出来。

### 1. 绘制的六个步骤
`View.java` 的 `draw()` 源码中清晰地注释了绘制的步骤：
1. **绘制背景**：`drawBackground(canvas)`。
2. **保存 Canvas 涂层**（可选，准备渐变等特效）。
3. **绘制自身内容**：调用 `onDraw(canvas)`。**这是自定义 View 最核心的复写点**。
4. **绘制子 View**：调用 `dispatchDraw(canvas)`。ViewGroup 会在此方法内递归调用子 View 的绘制。
5. **绘制淡出边缘和通道**（可选）。
6. **绘制装饰**（如滚动条、前景等）：`onDrawForeground(canvas)`。

### 2. 核心重写点
- **普通自定义 View**：重写 `onDraw()`，使用 Canvas 和 Paint 绘制控件主体。
- **自定义 ViewGroup**：默认情况下，为了提高性能，系统会为 ViewGroup 设置 `WILL_NOT_DRAW` 标记，导致其不调用 `onDraw()`。如果 ViewGroup 需要绘制自身内容（例如画一个背景水印或网格），需要显式调用 `setWillNotDraw(false)` 关闭这个优化。

---

## 五、 总结与路线延伸

View 的工作流程是一个经典的**树状递归模式**：
- **尺寸获取**：`measure()` -> `onMeasure()` -> 保存尺寸。
- **位置摆放**：`layout()` -> `onLayout()` -> 放置子项。
- **画面渲染**：`draw()` -> `onDraw()` & `dispatchDraw()` -> 渲染子项。

了解了基本工作流程，如果你想亲自动手绘制一些漂亮的图形，可以直接进入下一阶段：学习 [Canvas 图形绘制](canvas.md) 与 [Paint 画笔配置](paint.md)。