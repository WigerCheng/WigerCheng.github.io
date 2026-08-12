---
title: Android 经典动画：手把手教你实现帧动画（Frame Animation）与防 OOM 避坑指南
tags:
  - Android/Animation
---

在动画的分类中，**帧动画（Frame Animation / Drawable Animation）** 是最符合直觉的一种。它的原理和传统的“翻页手动画册”一样：将一系列设计好的静态图片按指定的顺序和时间间隔连续播放，从而欺骗人眼，在屏幕上形成动态的视觉画面。

虽然随着 Lottie 和属性动画的兴起，帧动画的使用场景有所减少，但在一些特定的复古游戏效果、简单 Loading 状态或特定序列帧播放时，它依然是一件不可或缺的工具。

本文将为你详解在 Android 中实现帧动画的完整步骤，并指出容易引发 OOM（内存溢出）的经典雷区。

---

## 1. 在 Android 中实现帧动画的四步曲

在 Android 中，帧动画的载体通常是 `AnimationDrawable` 类。我们可以完全通过 XML 资源文件轻松配置。

### 第一步：准备序列帧素材
将你的动画素材（一系列格式相同的图片，如 `.png` 或 `.webp`）放入 `res/drawable` 文件夹中。建议将图片命名为连续的编号，如 `animation1.png`, `animation2.png` ... `animationN.png`，方便管理。

![[sourse_location.png]]

---

### 第二步：编写动画资源 XML 文件
在 `res/drawable` 文件夹下新建一个动画 XML 文件（例如 `frame_anim.xml`），根节点为 `<animation-list>`。

`oneshot` 属性决定了动画是否只播放一遍：
- `false`：循环播放（默认）。
- `true`：只播放一遍，停留在最后一帧。

```xml
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">
    <!-- 每帧对应的图片和持续展示的时长（单位毫秒） -->
    <item android:drawable="@drawable/animation1" android:duration="200" />
    <item android:drawable="@drawable/animation2" android:duration="200" />
    <item android:drawable="@drawable/animation3" android:duration="200" />
    <item android:drawable="@drawable/animation4" android:duration="200" />
    <item android:drawable="@drawable/animation5" android:duration="200" />
    <item android:drawable="@drawable/animation6" android:duration="200" />
    <item android:drawable="@drawable/animation7" android:duration="200" />
    <item android:drawable="@drawable/animation8" android:duration="200" />
    <item android:drawable="@drawable/animation9" android:duration="200" />
    <item android:drawable="@drawable/animation10" android:duration="200" />
    <item android:drawable="@drawable/animation11" android:duration="200" />
    <item android:drawable="@drawable/animation12" android:duration="200" />
</animation-list>
```

---

### 第三步：在布局中声明承载动画的 ImageView
我们可以将该动画 XML 作为 `ImageView` 的 `src` 或 `background` 引入：

```xml
<ImageView
    android:id="@+id/img_frame"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:background="@drawable/frame_anim" />
```

---

### 第四步：在 Kotlin 中获取并播放动画
因为帧动画的载体是 `AnimationDrawable`，所以我们需要获取对应 View 的 `background` 并强转成 `AnimationDrawable` 实例来操控播放。

```kotlin
import android.graphics.drawable.AnimationDrawable
import android.os.Bundle
import android.widget.Button
import android.widget.ImageView
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class FrameAnimationActivity : AppCompatActivity() {

    private lateinit var frameAnimation: AnimationDrawable

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_frame_animation)

        val imgFrame = findViewById<ImageView>(R.id.img_frame)
        val tvOneShot = findViewById<TextView>(R.id.tv_one_shot)
        
        // 1. 从 ImageView 中获取 AnimationDrawable 实例
        frameAnimation = imgFrame.background as AnimationDrawable

        // 2. 显示是否为单次播放
        tvOneShot.text = "是否为单次播放: ${frameAnimation.isOneShot}"

        // 3. 安全开始播放按钮
        findViewById<Button>(R.id.btn_start).setOnClickListener {
            // 防抖动处理：如果正在运行，先停掉再从头开始
            if (frameAnimation.isRunning) {
                frameAnimation.stop()
            }
            frameAnimation.start()
        }

        // 4. 停止播放按钮
        findViewById<Button>(R.id.btn_stop).setOnClickListener {
            if (frameAnimation.isRunning) {
                frameAnimation.stop()
            }
        }
    }
}
```

---

## 2. 播放效果对比

| 循环播放 (oneshot = false) | 单次播放 (oneshot = true) |
| :---: | :---: |
| ![[no_one_shot.gif\|280]] | ![[one_shot.gif\|280]] |

---

## 3. 🚨 致命雷区：帧动画与 OOM（内存溢出）

帧动画是 Android 开发中**最容易产生 OOM 崩溃**的场景之一。

### 为什么会 OOM？
当系统解析 `<animation-list>` 时，`AnimationDrawable` 会**一次性将 XML 中定义的每一帧图片全部加载进内存中缓存**！
这意味着，如果你有一个 30 帧的动画，每帧图片解析后占用 2MB 内存，哪怕你还没开始播放，系统也会瞬间吃掉你 60MB 的 JVM 内存。如果动画图片分辨率较高、帧数较多，极易在配置较低的安卓设备上引发 OutOfMemoryError。

### 优选替代方案：
为了防止内存爆炸，在不同场景下建议采用以下更好的方案：
1. **Lottie 动画（强烈推荐）**：目前绝大多数 UI 动效、加载动效，均推荐让设计师使用 AE 导出 json 文件，使用 Lottie 库播放。不仅性能极佳，而且完全矢量化，适配各屏幕尺寸。
2. **WebP 或 APNG 动图**：使用支持动图播放的第三方库（如 Glide、Coil）去加载一张 WebP 动图。
3. **矢量路径动画（AnimatedVectorDrawable）**：如果只是简单的几何线条拉伸、SVG 图像变换，可以使用矢量动画，占用内存极小。
4. **动态解码（当必须使用大量序列帧时）**：使用 `Handler` 或协程，自己写一个轻量定时器，通过动态解码替换 `ImageView` 的图片源，每次内存里只保留单张图片的 Bitmap。
