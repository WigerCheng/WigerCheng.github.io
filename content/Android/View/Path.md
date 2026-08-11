---
draft: true
tags:
  - Android/View
---
- 🎈Path用来描述图形路径的对象。
- `Path` 可以描述直线、二次曲线、三次曲线、圆、椭圆、弧形、矩形、圆角矩形。把这些图形结合起来，就可以描述出很多复杂的图形。
- `Path` 有两类方法，一类是直接描述路径的，另一类是辅助的设置或计算。

## 直接描述路径
### 添加子图形【添加的完整封闭图形（除了 `addPath()` ）】
####  `addCircle(float x, float y, float radius, Direction dir)` 添加圆
- x,y 为圆心坐标，radius为圆的半径，dir画圆的方向。
```
   path.addCircle(300, 300, 200, Path.Direction.CW);
   canvas.drawPath(path, paint);
```
![[../../../zob-source/view/path_circle.png]]
- 实现效果同`canvas.drawCircle(x, y, radius, paint)`

#### `addOval(float left, float top, float right, float bottom, Direction dir) / addOval(RectF oval, Direction dir)` 添加椭圆

#### ` addRect(float left, float top, float right, float bottom, Direction dir) / addRect(RectF rect, Direction dir)` 添加矩形

#### `addRoundRect(RectF rect, float rx, float ry, Direction dir) / addRoundRect(float left, float top, float right, float bottom, float rx, float ry, Direction dir) / addRoundRect(RectF rect, float[] radii, Direction dir) / addRoundRect(float left, float top, float right, float bottom, float[] radii, Direction dir)` 添加圆角矩形

#### `addPath(Path path)` 添加另一个 Path

### 画线（直线或曲线）【添加的只是一条线】
#### `*lineTo` 画直线  从**当前位置**向目标位置画一条直线
- `lineTo(x, y)`，x和y是**绝对坐标**。
- `rLineTo(x, y)`，x和y是相对当前位置的**相对坐标**。
- ```
    path.lineTo(100, 100); // 由当前位置 (0, 0) 向 (100, 100) 画一条直线
    path.rLineTo(100, 0); // 由当前位置 (100, 100) 向正右方 100 像素的位置画一条直线
- ![[../../../zob-source/view/path_line.png]]
#### `*quadTo` 画二次贝塞尔曲线
- `quadTo(float x1, float y1, float x2, float y2)`，二次贝塞尔曲线的起点就是当前位置，而参数中的 `x1`, `y1` 和 `x2`, `y2` 则分别是**控制点**和**终点**的坐标。
- `rQuadTo(float dx1, float dy1, float dx2, float dy2)` ，二次贝塞尔曲线的起点就是当前位置，而参数中的 `x1`, `y1` 和 `x2`, `y2` 则分别是**控制点**和**终点**的**相对坐标**。
#### `*cubicTo` 画三次贝塞尔曲线
- `cubicTo(float x1, float y1, float x2, float y2, float x3, float y3)`，三次贝塞尔曲线的起点就是当前位置，而参数中的 `x1`, `y1` 、 `x2`, `y2` 和 `x3`, `y3` 则分别是**控制点1**、**控制点2**和**终点**的坐标。
- `rCubicTo(float x1, float y1, float x2, float y2, float x3, float y3)`，三次贝塞尔曲线的起点就是当前位置，而参数中的 `x1`, `y1` 、 `x2`, `y2` 和 `x3`, `y3` 则分别是**控制点1**、**控制点2**和**终点**的**相对坐标**。
#### `arcTo` 画弧形
- `arcTo(RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo) `
- `arcTo(float left, float top, float right, float bottom, float startAngle, float sweepAngle, boolean forceMoveTo) `
- `arcTo(RectF oval, float startAngle, float sweepAngle)`
## Direction
- 路径方向有两种：顺时针 (`CW` clockwise) 和逆时针 (`CCW` counter-clockwise) 。
- 对于普通情况，这个参数填 `CW` 还是填 `CCW` 没有影响。
- 它只是在**需要填充图形** (`Paint.Style` 为 `FILL` 或 `FILL_AND_STROKE`) ，并且**图形出现自相交**时，用于判断填充范围的。
- ![[../../../zob-source/view/direction_1.png]]
- ![[../../../zob-source/view/direction_2.png]]
- ![[../../../zob-source/view/direction_3.png]]

## FillType
- `EVEN_ODD`
	- 即 even-odd rule （奇偶原则）：对于平面中的任意一点，向任意方向射出一条射线，**这条射线和图形相交的次数（相交才算，相切不算哦）如果是奇数，则这个点被认为在图形内部，是要被涂色的区域；如果是偶数，则这个点被认为在图形外部，是不被涂色的区域**。
	- ![[../../../zob-source/view/even_odd.png]]
- `WINDING` （默认值）
	- 即 non-zero winding rule （非零环绕数原则）：首先，它需要你图形中的所有线条都是有绘制方向的,然后，同样是从平面中的点向任意方向射出一条射线，但计算规则不一样：**以 0 为初始值，对于射线和图形的所有交点，遇到每个顺时针的交点（图形从射线的左边向右穿过）把结果加 1，遇到每个逆时针的交点（图形从射线的右边向左穿过）把结果减 1**，最终把所有的交点都算上，**得到的结果如果不是 0，则认为这个点在图形内部，是要被涂色的区域；如果是 0，则认为这个点在图形外部，是不被涂色的区域。**
	- ![[../../../zob-source/view/winding.png]]
- `INVERSE_EVEN_ODD`
- `INVERSE_WINDING`
- ![[../../../zob-source/view/widing&even_odd.png]]