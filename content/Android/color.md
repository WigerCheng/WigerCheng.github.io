---
title: Android 颜色
tags:
  - Compose
---

## 📌 概述与颜色机制

在 Android 开发中，颜色值遵循 **ARGB** 标准，通常使用 32 位整型（32-bit Integer）或以 `#` / `0x` 开头的十六进制数值表示。

ARGB 依次代表：

- **A (Alpha)**：透明度 / 不透明度（Alpha Channel）
- **R (Red)**：红色分量
- **G (Green)**：绿色分量
- **B (Blue)**：蓝色分量

每个分量的取值范围为 `0 ~ 255`（即十六进制的 `0x00 ~ 0xFF`）。

> [!important] **Alpha 通道表示法**
>
> - **`0x00` (0)**：完全透明（Fully Transparent）
> - **`0xFF` (255)**：完全不透明（Fully Opaque）
> - 在 8 位十六进制表示法中（如 `#AARRGGBB`），前两位控制透明度，后六位控制 RGB 颜色。

---

## 📊 不透明度 (Alpha) 十六进制快速对照表

在日常 UI 开发与设计稿（如 Figma / Sketch）对接时，设计规范通常提供百分比不透明度（Opacity）。以下是常用百分比到十六进制 Alpha 值的映射表：

| 不透明度 (Opacity) | 透明度 (Transparency) | 16进制 (Hex) |
| :--: | :--: | :--: |
| **100%** | 0% | `FF` |
| **95%** | 5% | `F2` |
| **90%** | 10% | `E6` |
| **87%** | 13% | `DE` |
| **80%** | 20% | `CC` |
| **75%** | 25% | `BF` |
| **70%** | 30% | `B3` |
| **60%** | 40% | `99` |
| **50%** | 50% | `80` |
| **38%** | 62% | `60` |
| **30%** | 70% | `4D` |
| **20%** | 80% | `33` |
| **12%** | 88% | `1F` |
| **10%** | 90% | `1A` |
| **5%** | 95% | `0D` |
| **0%** | 100% | `00` |

---

## 🏛️ 1. 传统 View 体系中的颜色使用

### 1.1 资源定义与代码加载 (`colors.xml`)

在 XML 资源中定义颜色：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- 标准 8 位 ARGB (带 Alpha 前缀) -->
    <color name="primary_color">#FF6200EE</color>
    <!-- 半透明颜色 (50% Alpha) -->
    <color name="primary_alpha50">#806200EE</color>
    <!-- 简写 6 位 RGB (默认为 100% 不透明度) -->
    <color name="brand_red">#FF0000</color>
</resources>
```

在传统 View 代码中动态加载与解析：

```kotlin
// 1. 通过 ContextCompat 获取 Int 类型的颜色值
val colorInt: Int = ContextCompat.getColor(context, R.color.primary_color)

// 2. 解析颜色字符串 (注意：必须包含 '#' 前缀)
val parsedColorInt: Int = Color.parseColor("#FF6200EE")

// 3. 使用 Color.argb 动态构造整型颜色值
val customColorInt: Int = Color.argb(255, 98, 0, 238)
```

---

## 🎨 2. Jetpack Compose 中的 Color

在 Jetpack Compose 中，颜色使用 `androidx.compose.ui.graphics.Color` 结构体表示，具备强类型安全与高效的不可变变换支持。

### 2.1 创建 Compose Color 的常用方式

```kotlin
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.res.colorResource

// 方式 1：通过十六进制 Long 值创建 (必须带 0xFF Alpha 前缀！)
val primary = Color(0xFF6200EE)

// 方式 2：带透明度的十六进制 (例如 50% Alpha = 0x80)
val primary50 = Color(0x806200EE)

// 方式 3：通过 Float 参数构造 (red, green, blue, alpha 取值范围为 0.0f ~ 1.0f)
val customRed = Color(red = 1.0f, green = 0.2f, blue = 0.2f, alpha = 0.9f)

// 方式 4：在 Composable 函数中直接引用 xml 资源
@Composable
fun MyComponent() {
    val themeColor = colorResource(id = R.color.primary_color)
}
```

> [!danger] **踩坑预警：Compose Color 缺省 Alpha 陷阱**
> 如果写成 `Color(0x6200EE)`，由于缺少最高两位的 Alpha 字节，实际上 Alpha 值为 `0x00`（全透明），界面上会表现为**完全透明的黑色**！
> **正确写法**必须包含 Alpha 前缀：`Color(0xFF6200EE)`。

---

### 2.2 Compose Color 的常用变换与特殊颜色

```kotlin
val baseColor = Color(0xFF6200EE)

// 1. 动态调整透明度 (alpha 范围 0.0f ~ 1.0f)
val halfTransparentColor = baseColor.copy(alpha = 0.5f)

// 2. 预设特殊颜色
val transparent = Color.Transparent  // 完全透明 Color
val unspecified = Color.Unspecified  // 未指定颜色 (用于 Component 缺省值)
```

---

## 🔄 3. Compose Color 与 View Color (Int) 互相转换

在混合开发或迁移项目中，经常需要在 Compose `Color`（`androidx.compose.ui.graphics.Color`）与传统 View `Color`（`Int` 进制数值 / `android.graphics.Color`）之间进行转换。

### 3.1 核心互转代码

```kotlin
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.toArgb
import androidx.core.content.ContextCompat
import android.graphics.Color as AndroidColor

// ========================================================
// 1. Compose Color -> View Color (Int)
// ========================================================
val composeColor = Color(0xFF6200EE)

// 使用扩展函数 toArgb() 将 Compose Color 转为 Int (ARGB 格式)
val viewColorInt: Int = composeColor.toArgb()


// ========================================================
// 2. View Color (Int) -> Compose Color
// ========================================================
val viewColorInt2: Int = android.graphics.Color.RED

// 直接将 Int 传入 Color 构造函数
val composeColor2: Color = Color(viewColorInt2)


// ========================================================
// 3. XML 资源 ID (R.color.xxx) -> Compose Color
// ========================================================
// 场景 A：在 Composable 函数内使用
@Composable
fun ComposeDemo() {
    val composeColorFromRes: Color = colorResource(id = R.color.primary_color)
}

// 场景 B：在非 Composable 环境 (如普通 Helper 类)
val composeColorFromContext: Color = Color(
    ContextCompat.getColor(context, R.color.primary_color)
)


// ========================================================
// 4. android.graphics.Color 对象 (API 26+) -> Compose Color
// ========================================================
val androidColorObj: AndroidColor = AndroidColor.valueOf(1.0f, 0.0f, 0.0f, 1.0f)

// 方式 1：通过 toArgb() 转接
val composeColor3: Color = Color(androidColorObj.toArgb())

// 方式 2：使用框架提供的扩展函数 toComposeColor()
val composeColor4: Color = androidColorObj.toComposeColor()
```

---

### 3.2 实用封装工具扩展函数 (Kotlin Extensions)

为方便项目全局调用，建议整理如下工具扩展函数：

```kotlin
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.toArgb

/**
 * 将十六进制颜色字符串 (如 "#FF6200EE" 或 "#6200EE") 解析为 Compose Color
 */
fun String.toComposeColor(): Color {
    val colorInt = android.graphics.Color.parseColor(this)
    return Color(colorInt)
}

/**
 * 将 Compose Color 转为十六进制字符串 (格式: "#AARRGGBB")
 */
fun Color.toHexARGB(): String {
    return String.format("#%08X", this.toArgb())
}

/**
 * 将 Compose Color 转为十六进制字符串 (格式: "#RRGGBB"，忽略 Alpha)
 */
fun Color.toHexRGB(): String {
    return String.format("#%06X", 0xFFFFFF and this.toArgb())
}
```

---

## ⚠️ 常见踩坑与最佳实践总结

1. **十六进制 Alpha 忘加 `0xFF`**：Compose 中直接写 `Color(0xRRGGBB)` 会导致透明度为 `0x00`（不可见），切记写成 `Color(0xFFRRGGBB)`。
2. **字节顺序区分**：
   - **Android (ARGB)**：`#AARRGGBB` (Alpha 在最前的 2 位)
   - **Web / CSS (RGBA)**：`#RRGGBBAA` (Alpha 在最后的 2 位)
   - 处理后端接口或 Web 传来的 Hex 字符串时，务必提前确认字节顺序。
3. **性能推荐**：对同一颜色动态修改透明度时，推荐使用 `color.copy(alpha = 0.5f)`，避免频繁重新解析字符串或调用硬件资源。
