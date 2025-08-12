---
categories:
- Android
- Animation
date: "2022-04-24T00:00:00Z"
tags:
- Animation
title: Android动画之圆形揭露动画
---

## 圆形揭露动画
圆形揭露动画，可以让一个View裁剪成一个圆形，并以指定的圆心做半径变化动画。
当您显示或隐藏一组界面元素时，可给用户提供视觉连续性。

## 使用圆形揭露动画

通过调用`ViewAnimationUtils.createCircularReveal()`返回一个`Animator`实例，调用`animator.start()`显示动画。


`createCircularReveal()` 动画采用五个参数。
- View -> 即要执行动画的View。
- centerX -> 指定圆心的x点坐标。
- centerY -> 指定圆心的y点坐标。
- startRadius -> 动画开始前剪裁圆形的半径 。
- endRadius -> 动画结束时的圆形半径。

![效果图](circle_animation.gif)


[代码实现参考](https://github.com/WigerCheng/MyLearnApplication/blob/master/app/src/main/java/com/example/myapplication/animation/CircleAnimationActivity.kt)
```kotlin
findViewById<Button>(R.id.btn_circle_scale_down).setOnClickListener {
            val image = findViewById<ImageView>(R.id.img_circle)
            val width = image.width
            val height = image.height
            image.visibility = View.VISIBLE
            ViewAnimationUtils.createCircularReveal(
                image,
                width / 2,
                height / 2,
                0F,
                width.toFloat()
            ).apply {
                duration = 2000L
            }.also {
                it.start()
            }
        }
```