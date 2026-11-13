---
title: Android Canvas 绘图艺术（二）：Path 路径的高级几何绘制与填充规则
date: 2026-08-12T12:10:00
tags:
  - Android/View
---

# Android Canvas 绘图艺术（二）：Path 路径的高级几何绘制与填充规则

在 Android 自定义 View 的图形绘制中，简单的圆、矩形、椭圆只能满足基本需求。当你需要绘制折线图、心形、星形、齿轮或者复杂的拟物图标时，仅凭基础图形就无能为力了。

此时，我们需要祭出 `Canvas` 的灵魂搭档——**`Path`（路径）**。

`Path` 是用来描述任意二维几何图形轮廓的对象。你可以将它理解为画笔在画布上移动的轨迹。它既可以组合直线与曲线，又能处理图形相交时的复杂填充逻辑。本文将为你深入解析 Path 的几何绘制方法与底层的填充规则。

---

## 一、 Path 的两类方法概览

`Path` 的方法主要可以分为两大类：
1. **直接描述路径**：包括添加完整几何图形（如圆、矩形）和画线（直线、贝塞尔曲线、圆弧）。
2. **辅助设置与计算**：如设置填充规则（FillType）、重置路径（reset）等。

---

## 二、 基础几何体添加（直接封闭图形）

Path 允许我们将一些基础的闭合几何图形直接添加到当前路径中。除了特别说明，添加这些图形时都需要指定一个绘制方向——**`Path.Direction`**。

### 1. 添加圆形：`addCircle`
- **方法签名**：
  ```java
  public void addCircle(float x, float y, float radius, Path.Direction dir)
  ```
  - `x`, `y`：圆心坐标。
  - `radius`：圆的半径。
  - `dir`：绘制方向，`Path.Direction.CW`（顺时针）或 `Path.Direction.CCW`（逆时针）。

- **示例代码**：
  ```java
  path.addCircle(300, 300, 200, Path.Direction.CW);
  canvas.drawPath(path, paint);
  ```
  ![添加圆路径示例](../../../zob-source/view/path_circle.png)
  *(效果等同于直接调用 `canvas.drawCircle()`)*

### 2. 添加椭圆、矩形与圆角矩形
- **`addOval`**：添加椭圆。
  ```java
  public void addOval(float left, float top, float right, float bottom, Path.Direction dir)
  ```
- **`addRect`**：添加矩形。
  ```java
  public void addRect(float left, float top, float right, float bottom, Path.Direction dir)
  ```
- **`addRoundRect`**：添加圆角矩形。
  ```java
  public void addRoundRect(RectF rect, float rx, float ry, Path.Direction dir)
  ```
  - `rx`, `ry`：圆角在 X 轴和 Y 轴上的半径。还可以通过传入 `float[] radii` 数组来分别为四个角指定不同的圆角半径。

### 3. 追加另一个 Path：`addPath`
可以将一个已有的 `Path` 路径合并到当前的 `Path` 中：
```java
path.addPath(otherPath);
```

---

## 三、 画线（直线与高阶曲线）

这类方法用于绘制单条没有闭合的线段。其核心特点是：**每一次画线的起点都是上一次画线的终点（当前位置）。** 默认的初始位置是 `(0, 0)`。

### 1. 画直线：`lineTo` 与相对坐标 `rLineTo`
- **`lineTo(x, y)`**：从当前位置向绝对坐标 `(x, y)` 画一条直线。
- **`rLineTo(dx, dy)`**：这里的 `r` 代表 **Relative（相对）**。它是从当前位置出发，向水平方向偏移 `dx` 像素、竖直方向偏移 `dy` 像素的目标位置画一条直线。

- **示例代码**：
  ```java
  // 假设初始位置在 (0, 0)
  path.lineTo(100, 100);  // 由 (0, 0) 向 (100, 100) 画直线
  path.rLineTo(100, 0);   // 由 (100, 100) 向正右方偏移 100 像素，即画线到 (200, 100)
  ```
  ![画线示例](../../../zob-source/view/path_line.png)

> [!TIP]
> 如果不想从默认的 `(0, 0)` 开始画线，可以在最开始调用 **`moveTo(x, y)`** 移动当前画笔位置，而不会留下绘制痕迹。

### 2. 画二次贝塞尔曲线：`quadTo` & `rQuadTo`
利用控制点绘制平滑的曲线。
- **方法签名**：
  ```java
  public void quadTo(float x1, float y1, float x2, float y2)
  ```
  - `(x1, y1)`：贝塞尔曲线的**控制点**。
  - `(x2, y2)`：曲线的**终点**（起点为当前画笔位置）。

### 3. 画三次贝塞尔曲线：`cubicTo` & `rCubicTo`
相比二次贝塞尔曲线，拥有两个控制点，可以勾勒出更复杂的曲线（如心形波浪）。
- **方法签名**：
  ```java
  public void cubicTo(float x1, float y1, float x2, float y2, float x3, float y3)
  ```
  - `(x1, y1)`、`(x2, y2)`：两个**控制点**。
  - `(x3, y3)`：曲线的**终点**。

### 4. 画弧形：`arcTo`
用于截取椭圆的一部分弧线追加到路径中。
- **方法签名**：
  ```java
  public void arcTo(RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo)
  ```
  - `oval`：外接椭圆的范围。
  - `startAngle`：起始角度（以 3 点钟方向为 0°，顺时针为正角度）。
  - `sweepAngle`：扫过的角度。
  - `forceMoveTo`：如果是 `true`，会强制将画笔 `moveTo` 到弧形起点，不绘制与上一段路径的连接线；如果是 `false`（默认），则会绘制一条从上一段终点到该弧形起点的连接线。

---

## 四、 路径的方向与自相交填充规则

在上面添加几何图形时，我们接触到了 `Path.Direction`（CW/CCW）。对于非自相交的简单轮廓，这个方向几乎没有视觉影响。但当图形**发生自相交**（如大圆套小圆，或者绘制五角星）且画笔是 **`FILL`（填充模式）** 时，方向和 **`FillType`** 就共同决定了哪些区域该涂色，哪些区域该留白。

### 1. Path.Direction 的作用
- **`Path.Direction.CW`**：顺时针方向（Clockwise）。
- **`Path.Direction.CCW`**：逆时针方向（Counter-Clockwise）。

![不同方向的效果](../../../zob-source/view/direction_1.png)
![不同方向的效果2](../../../zob-source/view/direction_2.png)
![不同方向的效果3](../../../zob-source/view/direction_3.png)

### 2. 核心填充规则：FillType

我们可以通过 `path.setFillType(FillType ft)` 设置四种不同的填充规则：

#### ① `EVEN_ODD`（奇偶原则）
这是一个简单的纯几何判定规则，**与线条绘制的方向（CW/CCW）无关**。
- **规则**：对于平面中的任意一点，向屏幕外任意方向射出一条射线。计算该射线与 Path 所有轮廓线条的相交次数（相交才算，相切不算）。
  - 如果相交次数是**奇数**，判定该点在图形内部，**需要填充颜色**。
  - 如果相交次数是**偶数**，判定该点在图形外部，**保持留白**。

![奇偶原则示意](../../../zob-source/view/even_odd.png)

#### ② `WINDING`（非零环绕数原则，默认值）
该原则高度依赖轮廓线的**绘制方向**。
- **规则**：同样从平面中的一点向任意方向发射一条射线。初始计数值为 0。
  - 沿着射线方向，每当遇到一条**顺时针穿过**射线的轮廓线（从射线左侧向右穿过），计数值 **加 1**。
  - 每当遇到一条**逆时针穿过**射线的轮廓线（从射线右侧向左穿过），计数值 **减 1**。
  - 扫描完所有相交线后：如果最终累计结果**不是 0**，判定该点在图形内部，**需要填充颜色**；如果结果**等于 0**，判定点在图形外部，**保持留白**。

![非零环绕数原则示意](../../../zob-source/view/winding.png)

#### ③ `INVERSE_EVEN_ODD` 与 `INVERSE_WINDING`
这两者分别是前两者的**反转模式**：即原本要填充颜色的区域现在留白，原本留白的外部区域全部填满颜色（相当于底片反转）。

![填充规则对比汇总](../../../zob-source/view/widing&even_odd.png)

---

## 五、 总结与延伸

掌握了 `Path` 的直线、高阶曲线的勾勒，以及它在面对自相交时的 `FillType` 填充规则，你就可以随心所欲地绘制各种复杂的矢量图形了。

至此，Canvas 与画笔的几何绘图板块已基本完结。接下来，建议继续攻读 [Canvas 绘图之文字测量与精细排版](text.md)，掌握最后一块拼图——如何在自定义 View 中完美排版和居中文字。