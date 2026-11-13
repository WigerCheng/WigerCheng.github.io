---
title: Kotlin lazy：延迟初始化、线程安全模式与实现原理
tags:
  - Kotlin
---

`lazy` 是 Kotlin 提供的延迟初始化机制。它的核心思想很简单：**属性第一次被访问时才执行初始化逻辑，之后再次访问会直接返回第一次计算出来的结果。**

```kotlin
val name: String by lazy {
    println("初始化 name")
    "Kotlin"
}

fun main() {
    println("开始")

    println(name)
    // 初始化 name
    // Kotlin

    println(name)
    // Kotlin
}
```

上面这个例子中，程序启动时并不会马上初始化 `name`。只有第一次访问 `name` 时，`lazy` 代码块才会执行。第二次访问时，代码块不会再次执行，而是直接返回第一次缓存的结果。

## 为什么需要 lazy

有些属性的初始化成本比较高，但它们不一定每次都会被用到。例如：

- 读取配置文件
- 创建重量级对象
- 初始化正则表达式
- 构建缓存数据
- 在 Android 中延迟获取 View、Adapter、Manager 等对象

如果在对象创建时就立刻初始化这些属性，可能会浪费启动时间和内存。使用 `lazy` 后，可以把初始化推迟到真正需要它的时候。

```kotlin
class UserRepository {
    val cache: MutableMap<String, User> by lazy {
        println("init cache")
        mutableMapOf()
    }
}
```

如果 `cache` 一直没有被访问，初始化代码就不会执行。

## lazy 基本语法

`lazy` 通常和[[kotlin_delegation#属性委托|属性委托]]一起使用：

```kotlin
val property: String by lazy {
    "value"
}
```

这里的 `by lazy { ... }` 是属性委托。`lazy { ... }` 会返回一个 `Lazy<T>` 对象，而属性的 getter 会委托给这个 `Lazy<T>` 对象的 `value`。

也可以不使用 `by`，直接保存 `Lazy<T>` 对象：

```kotlin
val lazyName: Lazy<String> = lazy {
    "Kotlin"
}

fun main() {
    println(lazyName.value)
}
```

两种写法的区别是：

```kotlin
val name by lazy { "Kotlin" }      // name 的类型是 String
val lazyName = lazy { "Kotlin" }   // lazyName 的类型是 Lazy<String>
```

日常开发中，最常见的是第一种 `by lazy` 写法。

## lazy 只能用于 val

`lazy` 适合用于只读属性，也就是 `val`。

```kotlin
val name: String by lazy {
    "Kotlin"
}
```

它不适合用于 `var`，因为 `lazy` 的语义是“第一次计算后缓存结果，并且这个结果在 Lazy 生命周期内不再变化”。

如果一个属性后续需要反复赋值，那它就不应该使用 `lazy`，而应该直接使用普通 `var`。

## lazy 的实现入口

Kotlin 中最常用的 `lazy` 函数如下：

```kotlin
public actual fun <T> lazy(initializer: () -> T): Lazy<T> =
    SynchronizedLazyImpl(initializer)
```

调用 `lazy { ... }` 后，返回的是一个 `Lazy<T>` 对象。

`Lazy<T>` 是一个接口，核心只有两个成员：

```kotlin
public interface Lazy<out T> {
    public val value: T

    public fun isInitialized(): Boolean
}
```

- `value`：获取延迟初始化后的值。
- `isInitialized()`：判断当前 Lazy 是否已经完成初始化。

当属性通过 `by lazy` 委托给 `Lazy<T>` 后，访问属性本质上就是访问 `Lazy<T>.value`。

## 第一次访问时发生了什么

以默认的 `SynchronizedLazyImpl` 为例，整体流程可以简化理解为：

1. `Lazy` 内部先保存一个特殊标记，表示当前还没有初始化。
2. 第一次访问 `value` 时，发现还没初始化，于是执行 `initializer`。
3. 把 `initializer` 的返回值缓存起来。
4. 清空 `initializer` 引用，避免继续持有不必要的 lambda。
5. 后续再次访问 `value`，直接返回缓存值。

可以用伪代码理解：

```kotlin
class SimpleLazy<T>(
    private var initializer: (() -> T)?
) : Lazy<T> {

    private var cached: Any? = UNINITIALIZED

    override val value: T
        get() {
            if (cached === UNINITIALIZED) {
                cached = initializer!!.invoke()
                initializer = null
            }
            return cached as T
        }

    override fun isInitialized(): Boolean {
        return cached !== UNINITIALIZED
    }
}
```

真实实现还会考虑线程安全、内存可见性和并发访问。

## isInitialized

如果你拿到的是 `Lazy<T>` 对象，可以通过 `isInitialized()` 判断它是否已经初始化。

```kotlin
val lazyName = lazy {
    "Kotlin"
}

fun main() {
    println(lazyName.isInitialized()) // false

    println(lazyName.value) // Kotlin

    println(lazyName.isInitialized()) // true
}
```

如果使用的是 `val name by lazy { ... }`，直接拿到的是 `name` 的值，而不是 `Lazy<T>` 对象。需要判断初始化状态时，可以把委托对象单独保存起来：

```kotlin
private val nameDelegate = lazy {
    "Kotlin"
}

val name: String by nameDelegate

fun main() {
    println(nameDelegate.isInitialized())
}
```

## lazy 的线程安全模式

`lazy` 支持三种线程安全模式：

```kotlin
val value by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
    "Kotlin"
}
```

三种模式分别是：

| 模式           | 是否线程安全 | 初始化次数           | 适合场景                 |
| -------------- | ------------ | -------------------- | ------------------------ |
| `SYNCHRONIZED` | 是           | 只初始化一次         | 默认选择，多线程安全     |
| `PUBLICATION`  | 是           | initializer 可能多次 | 可接受重复初始化的场景   |
| `NONE`         | 否           | 不保证               | 确定只在单线程访问的场景 |

### SYNCHRONIZED

`SYNCHRONIZED` 是默认模式。

```kotlin
val name by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
    "Kotlin"
}
```

它会使用锁保证同一时刻只有一个线程执行初始化逻辑，并且初始化完成后的值对其他线程可见。

特点：

- 多线程安全。
- 初始化逻辑只会成功执行一次。
- 有锁开销。
- 适合大多数默认场景。

如果你没有特别理由，直接使用默认的 `lazy { ... }` 就是这个模式。

### PUBLICATION

`PUBLICATION` 允许多个线程同时执行初始化逻辑，但最终只有一个结果会被保存。

```kotlin
val name by lazy(LazyThreadSafetyMode.PUBLICATION) {
    println("init")
    "Kotlin"
}
```

在并发场景下，`initializer` 可能执行多次，但最后只有一个线程的计算结果会成为最终值。其他线程即使也算出了结果，也会被丢弃。

特点：

- 线程安全。
- 不保证 `initializer` 只执行一次。
- 适合初始化逻辑可以重复执行、且没有副作用的场景。

如果初始化逻辑会写文件、发请求、打点、修改全局状态，就不适合使用 `PUBLICATION`，因为它可能被执行多次。

### NONE

`NONE` 不做任何线程安全保证。

```kotlin
val name by lazy(LazyThreadSafetyMode.NONE) {
    "Kotlin"
}
```

特点：

- 没有锁开销。
- 性能最好。
- 非线程安全。
- 多线程同时访问时行为不确定。

只有在你能确定这个属性只会在单线程访问时，才适合使用 `NONE`。例如某些只在主线程访问的 UI 层属性。

## initializer 抛异常会怎样

如果 `lazy` 的初始化代码抛出异常，那么这次初始化不会被认为成功。下次访问时，会再次尝试执行初始化逻辑。

```kotlin
var count = 0

val value by lazy {
    count++
    if (count == 1) error("first failed")
    "success"
}

fun main() {
    runCatching { println(value) }
    println(value) // success
}
```

这个行为很重要：`lazy` 只缓存成功返回的结果，不会缓存异常。

## lazy 可以返回 null 吗

可以。

```kotlin
val name: String? by lazy {
    null
}
```

Kotlin 的 `lazy` 内部使用一个特殊标记表示“未初始化”，而不是用 `null` 表示未初始化。因此即使初始化结果是 `null`，也可以正确区分“已经初始化且值为 null”和“还没初始化”。

## 局部变量也可以 lazy

`lazy` 不只能用于类属性，也可以用于局部变量。

```kotlin
fun printIfNeeded(needPrint: Boolean) {
    val message by lazy {
        println("create message")
        "Hello Kotlin"
    }

    if (needPrint) {
        println(message)
    }
}
```

如果 `needPrint` 是 `false`，`message` 就不会被初始化。

局部变量使用 `lazy` 的场景相对少一些，但在某些条件分支复杂、初始化成本较高的代码里会很有用。

## lazy 和 lateinit 的区别

`lazy` 和 `lateinit` 都和“稍后初始化”有关，但它们解决的问题不一样。

| 对比项           | `lazy`                 | `lateinit`                                |
| ---------------- | ---------------------- | ----------------------------------------- |
| 适用变量         | `val`                  | `var`                                     |
| 初始化方式       | 第一次访问时自动初始化 | 手动赋值                                  |
| 是否可重新赋值   | 否                     | 是                                        |
| 是否允许基础类型 | 是                     | 否                                        |
| 是否线程安全     | 可选择线程安全模式     | 不提供线程安全机制                        |
| 未初始化访问     | 自动执行初始化逻辑     | 抛 `UninitializedPropertyAccessException` |

使用建议：

- 初始化逻辑可以写成表达式，并且只希望初始化一次，用 `lazy`。
- 属性需要由外部框架或生命周期回调稍后赋值，用 `lateinit`。

例如 Android 里：

```kotlin
private val adapter by lazy {
    UserAdapter()
}

private lateinit var binding: ActivityMainBinding
```

`adapter` 可以第一次使用时创建；`binding` 通常需要在 `onCreate()` 中手动赋值。

## Android 中使用 lazy 的注意点

在 Android 里，`lazy` 很常见，但要注意生命周期。

```kotlin
class MainActivity : AppCompatActivity() {
    private val adapter by lazy {
        UserAdapter()
    }
}
```

这种写法通常没问题，因为 `adapter` 跟随 Activity 生命周期一起创建和销毁。

**但如果 `lazy` 持有了 View、Context、Fragment binding 等短生命周期对象，就要小心内存泄漏。** 例如 Fragment 的 View 生命周期比 Fragment 本身短，如果在 Fragment 属性中用 `lazy` 缓存 View binding，就可能在 `onDestroyView()` 后继续持有旧 View。

这类场景更适合使用和 View 生命周期绑定的委托，或者在 `onDestroyView()` 中手动清空引用。

## 使用建议

日常使用 `lazy` 可以遵循这些原则：

- 优先用于 `val`。
- 初始化成本高、但不一定会用到的属性适合 lazy。
- 初始化逻辑不要有不可重复的副作用。
- 默认使用 `SYNCHRONIZED`，除非明确知道线程访问场景。
- 主线程 UI 属性可以考虑 `LazyThreadSafetyMode.NONE`，但前提是确定不会跨线程访问。
- Fragment 中不要随意用 `lazy` 长期持有 View。

## 总结

`lazy` 是 Kotlin 中非常实用的延迟初始化工具。它通过属性委托把初始化逻辑推迟到第一次访问时执行，并且只缓存成功初始化后的结果。

默认的 `lazy { ... }` 使用 `SYNCHRONIZED` 模式，线程安全但有锁开销；`PUBLICATION` 允许初始化逻辑并发执行多次，但最终只保存一个结果；`NONE` 性能最好，但完全不保证线程安全。

理解 `lazy` 时，可以抓住三点：第一次访问才初始化、成功后缓存结果、线程安全模式决定并发行为。只要这三点清楚，`lazy` 的使用场景和注意事项就很容易判断。
