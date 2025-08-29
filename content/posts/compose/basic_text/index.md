---
date: "2025-08-29T14:41:10+08:00"
draft: true
title : 'Compose文本显示'
---
在Compose UI中，我们使用`BasicText`或者直接使用Material封装的`Text`来显示文本。

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

## 传入文本

`text`是必填参数，可以接收`String`和`AnnotatedString`两种类型。也可以通过`stringResource()`方法使用项目定义的字符串资源。

### 普通字符串

```kotlin
BasicText("Hello World")
```

### 字符串资源

