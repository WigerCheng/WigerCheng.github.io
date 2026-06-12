[[Paint]]

## 颜色填充

🎈常用于在绘制之前设置底色，或者在绘制之后为界面设置半透明蒙版。

- `drawColor(color)`
- `drawRGB(int r, int g, int b)`
- `drawARGB(int a, int r, int g, int b)`

## 画圆

- `drawCircle(float centerX, float centerY, float radius, Paint paint)`
  -  `centerX` `centerY` 是圆心的坐标，第三个参数 `radius` 是圆的半径，单位都是像素。
  - `canvas.drawCircle(300, 300, 200, paint);`
  - ![[canva_circle.jpg]]

## 画矩形

- `drawRect(RectF rect, Paint paint)`
- `drawRect(Rect rect, Paint paint)`
- `drawRect(float left, float top, float right, float bottom, Paint paint)`
  - `left`, `top`, `right`, `bottom` 是矩形四条边的坐标。

 ```java
 paint.setStyle(Style.FILL);
 canvas.drawRect(100, 100, 500, 500, paint);
 
 paint.setStyle(Style.STROKE);
 canvas.drawRect(700, 100, 1100, 500, paint);
 ```

- ![[canva_rect.jpg]]

## 画点

- `drawPoint(float x, float y, Paint paint)`
  - `x` 和 `y` 是点的坐标。
  - 点的大小可以通过 `paint.setStrokeWidth(width)` 来设置
  - 点的形状可以通过 `paint.setStrokeCap(cap)`来设置
    - `ROUND` 画出来是圆形的点
    - `SQUARE` 或 `BUTT` 画出来是方形的点

```java
paint.setStrokeWidth(20);
paint.setStrokeCap(Paint.Cap.ROUND);
canvas.drawPoint(50, 50, paint);
```

![[canva_point_1.jpg]]

```java
paint.setStrokeWidth(20);
paint.setStrokeCap(Paint.Cap.SQUARE);
canvas.drawPoint(50, 50, paint);
```

![[canva_point_2.jpg]]

## 画点（批量）

- `drawPoints(float[] pts, int offset, int count, Paint paint)`
- `drawPoints(float[] pts, Paint paint)`
  - 它和 `drawPoint()` 的区别是可以画多个点。
  - `pts` 这个数组是点的坐标，每两个成一对。
  - `offset` 表示跳过数组的前几个数再开始记坐标。
  - `count` 表示一共要绘制几个点。

```java
float[] points = {0, 0, 50, 50, 50, 100, 100, 50, 100, 100, 150, 50, 150, 100};
// 绘制四个点：(50, 50) (50, 100) (100, 50) (100, 100)
canvas.drawPoints(points, 2 /* 跳过两个数，即前两个 0 */,
          8 /* 一共绘制 8 个数（4 个点）*/, paint);
```

![[canva_points.jpg]]

## 画椭圆

- `drawOval(RectF rect, Paint paint)`
- `drawOval(float left, float top, float right, float bottom, Paint paint)`
  - 只能绘制横着的或者竖着的椭圆，不能绘制斜的（斜的倒是也可以，但不是直接使用 `drawOval()`，而是配合几何变换，后面会讲到）
  - `left`, `top`, `right`, `bottom` 是这个椭圆的左、上、右、下四个边界点的坐标。

```java
paint.setStyle(Style.FILL);
canvas.drawOval(50, 50, 350, 200, paint);

paint.setStyle(Style.STROKE);
canvas.drawOval(400, 50, 700, 200, paint);
```

![[canva_oval.jpg]]
