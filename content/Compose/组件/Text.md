---
date: 2025-08-29T14:41:10+08:00
draft: true
title: Compose文本显示
tags:
  - Compose/组件
---
> 当前使用的compose版本是1.9.4

在Compose UI中，我们可以使用Material库封装的`Text`或者使用compose.foundation下的`BasicText`来显示文本。

```kotlin
@Composable
fun BasicText(
    text: String,
    modifier: Modifier = Modifier,
    style: TextStyle = TextStyle.Default,
    onTextLayout: ((TextLayoutResult) -> Unit)? = null,
    overflow: TextOverflow = TextOverflow.Clip,
    softWrap: Boolean = true,
    maxLines: Int = Int.MAX_VALUE,
    minLines: Int = 1,
    color: ColorProducer? = null,
    autoSize: TextAutoSize? = null,
) {}

@Composable
fun BasicText(
    text: AnnotatedString,
    modifier: Modifier = Modifier,
    style: TextStyle = TextStyle.Default,
    onTextLayout: ((TextLayoutResult) -> Unit)? = null,
    overflow: TextOverflow = TextOverflow.Clip,
    softWrap: Boolean = true,
    maxLines: Int = Int.MAX_VALUE,
    minLines: Int = 1,
    inlineContent: Map<String, InlineTextContent> = mapOf(),
    color: ColorProducer? = null,
    autoSize: TextAutoSize? = null,
) {}
```

## 显示文本

Text组件提供了两个重载的方法，text参数支持传`String`或者`AnnotatedString`，前者是直接显示一个普通的字符串，后者是可以支持设置不同样式的文本，类似于View体系下的TextView的setText支持显示不同样式的文本。

原理其实是和View体系是类似的，可以把`AnnotatedString`当作是Compose版的`SpannedString`。

### 显示普通的字符串

Text组件支持直接显示字符串和显示字符串资源的文字。

```kotlin
@Preview(locale = "en")
@Composable
private fun SimpleText() {
    Column {
        //简单的字符串
        BasicText("Hello World")
        //字符串资源
        //<string name="hello_world">Hello World</string>
        BasicText(stringResource(R.string.hello_world))
        //字符串资源（带参数）
        //<string name="congratulate">Happy %1$s %2$d</string>
        BasicText(stringResource(R.string.congratulate, "New Year", 2025))
        //数量字符串（复数）
        //<plurals name="runtime_format">
        //    <item quantity="one">%1$d minute</item>
        //    <item quantity="other">%1$d minutes</item>
        //</plurals>
        BasicText(
            pluralStringResource(
                R.plurals.runtime_format,
                1,
                1
            )
        )
        BasicText(
            pluralStringResource(
                R.plurals.runtime_format,
                2,
                2
            )
        )
    }
}

```

![Simple Text](simple_text.png)

## 设置文本样式

### 文本颜色

```kotlin
@Preview
@Composable
private fun ColorText() {
    Column {
        val textStyle = LocalTextStyle.current
        BasicText("Hello World", color = { Color.Magenta })
        BasicText("Hello World", style = textStyle.copy(color = Color.Green))
        BasicText("Hello World", style = textStyle.copy(color = Color.Green), color = { Color.Red })
        Text("Hello World", color = Color.Yellow)
        Text("Hello World", style = textStyle.copy(Color.Blue))
        Text("Hello World", color = Color.Black, style = textStyle.copy(Color.Blue))
    }
}
```

![Color Text](color_text.png)

Text和BasicText均有两种改变字体颜色的方式，一种是修改style的color属性，另一种是直接传入color参数。

在BasicText中，color参数类型是`ColorProducer?`, ColorProducer其实就是替代`()->Color`的SAM接口，使用时方法的返回值是颜色就可以。
看`TextAnnotatedStringNode`源码，color参数实际上是overrideColor，字面意思就是重写颜色，优先级最高的，次之取TextStyle的color值，最后兜底是黑色。

```kotlin
drawIntoCanvas { canvas ->
    val overrideColorVal = overrideColor?.invoke() ?: Color.Unspecified
    val color =
        if (overrideColorVal.isSpecified) {
            overrideColorVal
        } else if (style.color.isSpecified) {
            style.color
        } else {
            Color.Black
        }
}
```

在Text中，color参数的原理就是改变style的color值。先区color参数的颜色值，如果没指定就取TexxtStyle的color值，最后取的是LocalContentColor.current的颜色值。

```kotlin
val textColor = color.takeOrElse { style.color.takeOrElse {LocalContentColor.current } }

BasicText(
    ...,
    style =
        style.merge(
            color = textColor,
            fontSize = fontSize,
            fontWeight = fontWeight,
            textAlign = textAlign ?: TextAlign.Unspecified,
            lineHeight = lineHeight,
            fontFamily = fontFamily,
            textDecoration = textDecoration,
            fontStyle = fontStyle,
            letterSpacing = letterSpacing,
        ),
    ...
)
```

## 配置文本布局

### 处理文段行

在Text中支持用`minLines`设置最小显示行数，用`maxLines`设置最大显示行数，用`sortWrap`设置文段长度超过Text宽度是否自动换行。

> 以下截图是基于180dp\*180dp, maxLines = 3, minLines = 2, 上面sortWrap = true, 下面sortWrap = false。

![line](line.png)

### 处理文字溢出

在Text组件中可以通过`overflow`来指定文字溢出的规则，默认是`TextOverflow.Clip`。

> 以下截图是基于120dp\*120dp的 Box，120\*70的Text，每个文本重复24次。

| 图片                                      | 规则             | 描述                       | 补充                                                                                                                          |
| --------------------------------------- | -------------- | ------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| ![clip](clip.png)              | Clip           | 文本过长将会裁掉超出容器的那部份         | -                                                                                                                           |
| ![visible](visible.png)        | Visible        | 无论容器有没有足够的空间，所有文本都会渲染出来。 | 1. 容器内溢出的区域是不能响应点击的，即红色区域是不能响应点击的，除非将`height`切换成`heightIn(min=)`。</br>2.如果其他的修饰符会裁减，Visible将失效，例如`Modifier.clipToBounds()`。 |
| ![start_ellipsis](start.png)   | StartEllipsis  | 文本开头使用省略号                | 在Android中要求是单行文本[^single_line]才生效                                                                                           |
| ![middle_ellipsis](middle.png) | MiddleEllipsis | 文本中间使用省略号                | 在Android中要求是单行文本[^single_line]才生效                                                                                           |
| ![ellipsis](ellipsis.png)      | Ellipsis       | 文本末尾使用省略号                | -                                                                                                                           |

[^single_line]: 单行文本：softwrap = false | maxLine = 1
