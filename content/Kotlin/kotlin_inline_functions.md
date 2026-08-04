---
title: Kotlin 内联函数：inline、noinline、crossinline 与 reified
tags:
  - Kotlin
---

Kotlin 的 `inline` 经常被简单理解成“提升性能”。这个理解没有错，但不够准确。

内联函数真正重要的使用场景，是高阶函数。它可以让编译器把函数体和 lambda 参数展开到调用处，从而减少函数调用开销和 lambda 对象创建成本。同时，`inline` 还带来了两个 Kotlin 里很有用的能力：非局部返回和 `reified` 泛型。

## 为什么需要 inline

先看一个普通高阶函数：

```kotlin
fun runBlock(block: () -> Unit) {
    block()
}

fun main() {
    runBlock {
        println("Hello")
    }
}
```

这段代码写起来很自然，但从 JVM 的角度看，lambda 可能会被编译成对象，调用 `block()` 也会产生一次间接调用。

如果这个高阶函数很短、调用很频繁，这些额外成本就可能变得明显。

把函数声明成 `inline`：

```kotlin
inline fun runBlock(block: () -> Unit) {
    block()
}
```

编译器会尽量把 `runBlock` 的函数体和传入的 lambda 展开到调用处。也就是说，调用处最终更接近：

```kotlin
println("Hello")
```

这就是内联函数最核心的价值。

## inline 做了什么

下面是一个简单例子：

```kotlin
inline fun inlinePrint() {
    println("print")
}

fun testInline() {
    inlinePrint()
}
```

编译后，`testInline()` 中不再只是调用 `inlinePrint()`，而是直接包含 `println("print")` 的逻辑。

如果是普通函数：

```kotlin
fun normalPrint() {
    println("print")
}

fun testNormal() {
    normalPrint()
}
```

`testNormal()` 里仍然会保留一次普通函数调用。

所以，`inline` 的基本作用可以概括为：把被调用函数的代码复制到调用位置。

## 什么时候适合使用 inline

`inline` 适合这些场景：

- 函数是高阶函数，接收 lambda 参数。
- 函数体比较短。
- 函数会被频繁调用。
- 希望配合 `reified` 使用运行时泛型类型。

Kotlin 标准库里很多常见函数都是内联函数，例如 `let`、`run`、`apply`、`also`、`use`、`repeat`、`filter` 等。

但不建议给普通大函数随手加 `inline`。因为内联会把函数体复制到每个调用点，如果函数体很大，可能导致字节码体积膨胀，反而得不偿失。

## noinline：禁止某个 lambda 被内联

默认情况下，传给内联函数的 lambda 参数也会被内联。

```kotlin
inline fun doWork(block: () -> Unit) {
    block()
}
```

但有些时候，我们不希望某个 lambda 被内联。比如需要把 lambda 保存到变量中，或者继续传给另一个非内联函数。

这时可以使用 `noinline`：

```kotlin
inline fun doWork(
    inlined: () -> Unit,
    noinline notInlined: () -> Unit
) {
    inlined()
    runLater(notInlined)
}

fun runLater(block: () -> Unit) {
    block()
}
```

`inlined` 会被内联，`notInlined` 不会被内联。因为 `notInlined` 仍然是一个函数对象，所以它可以作为参数继续传递。

简单理解：

- `inline` 参数会被展开，不再像普通对象一样存在。
- `noinline` 参数不会被展开，可以像普通 lambda 一样保存和传递。

## 非局部返回

内联函数还有一个容易让人惊讶的特性：传给内联函数的 lambda 可以直接返回外层函数。

```kotlin
inline fun runInline(block: () -> Unit) {
    block()
}

fun test() {
    runInline {
        return
    }

    println("这里不会执行")
}
```

这里的 `return` 返回的不是 lambda，而是外层的 `test()` 函数。这叫非局部返回。

为什么可以这样？因为 lambda 被内联到了 `test()` 的调用位置，所以它本质上已经变成了 `test()` 函数体的一部分。

普通高阶函数不能这样写：

```kotlin
fun runNormal(block: () -> Unit) {
    block()
}

fun test() {
    runNormal {
        // return // 编译错误
    }
}
```

如果只是想从 lambda 中返回，可以使用标签返回：

```kotlin
fun test() {
    runNormal {
        return@runNormal
    }

    println("继续执行")
}
```

## crossinline：禁止非局部返回

有些内联函数虽然接收 lambda，但不会直接在当前调用栈里执行 lambda，而是把它放到另一个对象或执行上下文里。

例如：

```kotlin
inline fun runCross(crossinline block: () -> Unit) {
    val runnable = Runnable {
        block()
    }
    runnable.run()
}
```

这个 lambda 被放进了 `Runnable`。如果此时允许 lambda 写非局部 `return`，语义就会变得不安全：它到底要从哪个函数返回？

因此 Kotlin 提供了 `crossinline`：

```kotlin
fun test() {
    runCross {
        println("Hello")
        // return // 编译错误
    }
}
```

`crossinline` 的含义是：这个 lambda 仍然可以被内联，但禁止在里面使用非局部返回。

## reified：保留泛型类型

JVM 上的普通泛型会发生类型擦除。比如下面这种写法不能通过编译：

```kotlin
fun <T> printType() {
    // println(T::class) // 编译错误
}
```

因为运行时拿不到 `T` 的具体类型。

但是在内联函数里，可以使用 `reified`：

```kotlin
inline fun <reified T> printType() {
    println(T::class)
}

fun main() {
    printType<String>()
    // class kotlin.String
}
```

`reified` 必须和 `inline` 一起使用。原因是函数内联后，编译器可以在调用点知道真实的类型参数，并把它替换进去。

Android 中一个很常见的例子是封装页面跳转：

```kotlin
inline fun <reified T> Context.startActivity() {
    startActivity(Intent(this, T::class.java))
}

context.startActivity<DetailActivity>()
```

如果没有 `reified`，通常就需要显式传入 `Class<T>`：

```kotlin
fun <T> Context.startActivity(clazz: Class<T>) {
    startActivity(Intent(this, clazz))
}

context.startActivity(DetailActivity::class.java)
```

`reified` 让这种 API 更简洁，也更符合 Kotlin 的调用习惯。

## inline 的代价

`inline` 并不是越多越好。

它的主要代价是代码体积。因为函数体会被复制到每个调用点，如果一个内联函数很大，或者被调用很多次，生成的字节码可能会变大。

所以使用 `inline` 时可以遵循一个简单原则：

- 短小的高阶函数适合内联。
- 需要 `reified` 泛型时必须内联。
- 普通函数、复杂函数、大函数不要随手内联。

## inline、noinline、crossinline 的区别

| 修饰符        | 作用                                     |
| ------------- | ---------------------------------------- |
| `inline`      | 内联函数体和可内联 lambda                |
| `noinline`    | 禁止某个 lambda 参数被内联               |
| `crossinline` | 允许 lambda 被内联，但禁止非局部返回     |
| `reified`     | 让内联函数中的泛型参数保留运行时类型信息 |

## 总结

Kotlin 内联函数的核心不是“所有函数都加上就更快”，而是为高阶函数服务。

`inline` 通过把函数体展开到调用处，减少函数调用和 lambda 对象创建成本；`noinline` 用于保留某个 lambda 的对象形态；`crossinline` 用于禁止不安全的非局部返回；`reified` 则利用内联后的调用点类型信息，解决普通泛型运行时类型被擦除的问题。

实际使用时，可以记住一句话：短小、频繁调用、带 lambda 参数的函数适合内联；需要运行时泛型类型时使用 `inline + reified`。

## 参考

- [Inline functions | Kotlin Documentation](https://kotlinlang.org/docs/inline-functions.html)
- [内联函数与具体化的类型参数 - Kotlin 语言中文站](https://www.kotlincn.net/docs/reference/inline-functions.html)
