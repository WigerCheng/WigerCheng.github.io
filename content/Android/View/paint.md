---
title: Android Custom View 必备：神笔 Paint 的基础属性与高级配置指南
date: 2026-08-12T12:10:00
tags:
  - Android/View
---

# Android Custom View 必备：神笔 Paint 的基础属性与高级配置指南

如果说 `Canvas`（画布）定义了我们要画什么图形，那么 `Paint`（画笔）就决定了这个图形以怎样的姿态和样式呈现。

在自定义 View 的世界里，`Paint` 就像是一支充满魔法的神笔。通过给它设置不同的属性，我们不仅能控制颜色、线条粗细，还能开启抗锯齿、配置描边样式、控制排版大小等。

本文将详细介绍 `Paint` 类中最常用、最基础的几个核心配置方法，助你快速上手自定义 View 绘图。

---

## 一、 设置绘制模式：`setStyle`

`setStyle(Paint.Style style)` 决定了图形是以填充（实心）还是描边（空心）的方式画出来。

### 1. 样式参数（Paint.Style）
- **`Paint.Style.FILL`**：填充模式。仅填充图形内部，不绘制轮廓边缘（默认值）。
- **`Paint.Style.STROKE`**：描边模式。仅绘制图形轮廓（勾边），不填充内部。
- **`Paint.Style.FILL_AND_STROKE`**：合并模式。既绘制轮廓又填充内部。

### 2. 示例效果
```java
// 设置为描边模式
paint.setStyle(Paint.Style.STROKE);
```

![描边与填充样式对比](paint_style.png)

---

## 二、 设置画笔颜色：`setColor`

使用 `setColor(int color)` 来指定绘制图形的色彩。

### 1. 设置方式
可以直接传入 Android 系统自带的颜色常量，或者通过十六进制 ARGB 传入自定义色彩：

```java
// 使用预设红
paint.setColor(Color.RED);

// 使用 16 进制设置带透明度的颜色（透明度为 88% 的浅蓝色）
paint.setColor(0xCC3F51B5);
```

![设置画笔颜色效果](paint_color.png)

---

## 三、 设置线条宽度：`setStrokeWidth`

当画笔的绘制模式被设置为 `STROKE`（描边）或 `FILL_AND_STROKE` 时，可以通过 `setStrokeWidth(float width)` 来控制轮廓线条的粗细。

### 1. 示例代码
```java
// 设置线条宽度为 20 像素
paint.setStrokeWidth(20);
```

![线条粗细对比](paint_stroke_width.jpg)

> [!NOTE]
> `setStrokeWidth` 的单位是像素（pixel）。在做屏幕适配时，建议将 `dp` 转换为 `px` 再设置进去，以防在不同密度的屏幕上看起来线条粗细不一。

---

## 四、 设置文字大小：`setTextSize`

当使用 Canvas 的 `drawText` 系列方法绘制文字时，此属性用于设置文字的字号大小。

### 1. 示例代码
```java
// 设置文字大小为 40 像素
paint.setTextSize(40);
```

---

## 五、 设置抗锯齿开关：`setAntiAlias`

由于屏幕是由一个一个细小的方形像素点组成的，在绘制倾斜线条或圆形曲线时，边缘容易出现类似“楼梯”一样的锯齿。

通过开启抗锯齿，系统会在边缘做平滑的插值模糊处理，使圆弧边缘看起来更加圆滑。

### 1. 开启方式
可以在初始化画笔时传入 flag，或者调用 `setAntiAlias(true)`：

```java
// 方式一：构造时传入参数
Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);

// 方式二：动态设置
paint.setAntiAlias(true);
```

![抗锯齿开启与关闭对比](paint_anti_alias.png)

> [!TIP]
> 开启抗锯齿会消耗稍微多一点的绘制性能，但在现代 Android 设备上，这一点性能开销几乎可以忽略不计。为了保证界面精致感，除了绘制纯直线外，建议绘制圆弧、文本等曲线图形时默认开启抗锯齿。

---

## 六、 设置端点形状：`setStrokeCap`

当绘制线条（`drawLine`）或点（`drawPoint`）且画笔模式为 `STROKE` 时，`setStrokeCap(Paint.Cap cap)` 可以控制线条或点的端点样式。

### 1. 样式参数（Paint.Cap）
- **`Paint.Cap.BUTT`**：平头。线段终点处直接切齐，不多出一丁点儿（默认值）。
- **`Paint.Cap.ROUND`**：圆头。在端点处会额外多绘制一个半圆（如果绘制的是点，则点会呈现为圆点）。
- **`Paint.Cap.SQUARE`**：方头。在端点处额外多绘制一个半宽的方块（如果绘制的是点，则点会呈现为方点）。

```java
// 设置画笔的端点为圆头
paint.setStrokeCap(Paint.Cap.ROUND);
```

---

## 七、 总结

`Paint` 就像是自定义 View 绘图的控制面板。熟练掌握 `setStyle`、`setColor`、`setStrokeWidth`、`setAntiAlias` 以及 `setStrokeCap`，你就拥有了驾驭 Canvas 的核心底座。

如果想让你的图形更具个性，可以使用这些 Paint 属性去配合 [Canvas 的几何绘制](canvas.md) 或 [Path 的曲线勾勒](path.md)，创造出丰富多彩的自定义视觉效果。
