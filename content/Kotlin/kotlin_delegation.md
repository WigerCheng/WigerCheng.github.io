---
title: Kotlin 委托：用 by 把职责交给更合适的对象
tags:
  - Kotlin
---

委托是一种很常见的设计思想：一个对象不直接完成某件事，而是把这件事交给另一个对象完成。

在 Java 里，如果想复用某个接口实现，通常要手写一堆转发方法。Kotlin 则把这种模式做成了语言特性，通过 `by` 关键字就可以完成类委托和属性委托。

```kotlin
class LogManager(logger: Logger) : Logger by logger
```

这行代码的意思是：`LogManager` 实现了 `Logger` 接口，但具体实现交给 `logger` 对象处理。

## 委托解决什么问题

委托主要解决两个问题：

- 减少重复的转发代码。
- 把对象职责拆开，让当前类只保留自己真正关心的逻辑。

如果没有委托，我们经常会写出这样的代码：

```kotlin
interface Logger {
    fun log(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String) {
        println("Console Log: $message")
    }
}

class LogManager(private val logger: Logger) : Logger {
    override fun log(message: String) {
        logger.log(message)
    }
}
```

`LogManager#log()` 只是把调用转发给 `logger`。如果接口里有很多方法，这种模板代码会越来越多。

使用 Kotlin 类委托后，可以直接写成：

```kotlin
class LogManager(private val logger: Logger) : Logger by logger
```

可读性更强，重复代码也更少。

## 类委托

类委托的语法是：

```kotlin
class Wrapper(delegate: Interface) : Interface by delegate
```

来看一个完整例子：

```kotlin
fun interface Logger {
    fun log(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String) {
        println("Console Log: $message")
    }
}

class FileLogger : Logger {
    override fun log(message: String) {
        println("Saving to file: $message")
    }
}

class LogManager(logger: Logger) : Logger by logger

fun main() {
    val consoleLog = LogManager(ConsoleLogger())
    consoleLog.log("Hello")
    // Console Log: Hello

    val fileLog = LogManager(FileLogger())
    fileLog.log("Hello")
    // Saving to file: Hello
}
```

`LogManager` 自己没有实现 `log()`，但它通过 `Logger by logger` 把 `Logger` 接口的实现委托给了构造函数传入的对象。

如果当前类想接管某个方法，也可以自己重写：

```kotlin
class LogManager(private val logger: Logger) : Logger by logger {
    override fun log(message: String) {
        logger.log("[LogManager] $message")
    }
}
```

当前类显式重写的方法优先级更高，其余没有重写的方法仍然交给委托对象。

## 属性委托

属性委托用于把属性的读取和写入交给另一个对象处理。

```kotlin
class Example {
    var prop: String by Delegate()
}
```

委托对象需要提供 `getValue()` 方法。如果属性是 `var`，还需要提供 `setValue()` 方法。

```kotlin
import kotlin.reflect.KProperty

class Delegate {
    private var realValue: String = "Kotlin"

    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        println("getValue: ${property.name}")
        return realValue
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("setValue: ${property.name}, value=$value")
        realValue = value
    }
}

class Example {
    var name: String by Delegate()
}

fun main() {
    val example = Example()

    println(example.name)
    // getValue: name
    // Kotlin

    example.name = "Android"
    // setValue: name, value=Android
}
```

从使用者角度看，`name` 就像一个普通属性；但从编译器角度看，访问 `name` 会被转换成对委托对象的调用。

大致可以理解成：

```kotlin
class Example {
    private val nameDelegate = Delegate()

    var name: String
        get() = nameDelegate.getValue(this, ::name)
        set(value) = nameDelegate.setValue(this, ::name, value)
}
```

这就是属性委托的本质：属性本身只保留访问入口，真正的数据存取逻辑放在委托对象里。

## getValue 和 setValue 的参数

属性委托方法通常会看到这几个参数：

```kotlin
operator fun getValue(thisRef: Any?, property: KProperty<*>): String

operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String)
```

它们分别表示：

| 参数       | 含义                                                |
| ---------- | --------------------------------------------------- |
| `thisRef`  | 属性所属对象。顶层属性或局部变量委托时可以是 `null` |
| `property` | 被委托的属性信息，例如属性名                        |
| `value`    | `setValue()` 中接收到的新值                         |

如果不想手写函数签名，也可以实现标准库里的接口：

```kotlin
import kotlin.properties.ReadOnlyProperty
import kotlin.properties.ReadWriteProperty
import kotlin.reflect.KProperty

val readonly by ReadOnlyProperty<Any?, String> { _, property ->
    "value from ${property.name}"
}

var writable by object : ReadWriteProperty<Any?, String> {
    private var value = "Kotlin"

    override fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return value
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        this.value = value
    }
}
```

`ReadOnlyProperty` 适合 `val`，`ReadWriteProperty` 适合 `var`。

## lazy：延迟初始化

`lazy` 是最常见的属性委托。它用于延迟初始化一个只读属性。

```kotlin
val config: String by lazy {
    println("init config")
    "Kotlin"
}

fun main() {
    println(config)
    // init config
    // Kotlin

    println(config)
    // Kotlin
}
```

第一次访问 `config` 时，`lazy` 代码块会执行，并把结果缓存起来。后续再次访问时，不会重复执行代码块，而是直接返回缓存值。

`lazy` 适合这些场景：

- 初始化成本较高。
- 属性不一定每次都会用到。
- 初始化结果只需要计算一次。

`lazy` 默认是线程安全的。如果确认只在单线程中访问，也可以指定模式：

```kotlin
val config by lazy(LazyThreadSafetyMode.NONE) {
    "Kotlin"
}
```

## observable：监听属性变化

`Delegates.observable()` 可以监听属性变化。

```kotlin
import kotlin.properties.Delegates

class User {
    var name: String by Delegates.observable("init") { property, old, new ->
        println("${property.name}: $old -> $new")
    }
}

fun main() {
    val user = User()
    user.name = "Kotlin"
    // name: init -> Kotlin
}
```

`observable` 会在属性赋值完成后触发回调，因此回调里看到的新值已经生效。

## vetoable：拦截属性赋值

`Delegates.vetoable()` 和 `observable()` 类似，也能监听属性变化。区别在于，`vetoable()` 可以决定本次赋值是否生效。

```kotlin
import kotlin.properties.Delegates

var score by Delegates.vetoable(0) { _, old, new ->
    new >= old
}

fun main() {
    score = 10
    println(score) // 10

    score = 5
    println(score) // 10
}
```

上面这个例子里，只有新值大于或等于旧值时，赋值才会成功。

## Map 委托

`Map` 也可以作为属性委托。此时属性名会作为 key，从 Map 中取值。

```kotlin
class User(private val map: Map<String, Any?>) {
    val id: Long by map
    val name: String by map
}

fun main() {
    val user = User(
        mapOf(
            "id" to 1L,
            "name" to "Wiger"
        )
    )

    println(user.id) // 1
    println(user.name) // Wiger
}
```

这种写法适合把动态数据映射成对象属性，比如配置、JSON、数据库字段等。不过它依赖 key 和属性名匹配，如果 key 不存在或类型不匹配，就会在运行时出错。

## 局部变量委托

委托不仅可以用于类属性，也可以用于局部变量。

```kotlin
fun main() {
    val message by lazy {
        println("create message")
        "Hello"
    }

    println(message)
    println(message)
}
```

局部变量委托的机制和属性委托类似，访问变量时会触发委托对象的 `getValue()`。

## provideDelegate

`provideDelegate()` 可以在属性和委托对象绑定时介入。

普通属性委托是在访问属性时调用 `getValue()` 或 `setValue()`；而 `provideDelegate()` 更早，它会在属性初始化阶段被调用。

```kotlin
import kotlin.properties.PropertyDelegateProvider
import kotlin.properties.ReadOnlyProperty
import kotlin.reflect.KProperty

class VersionProperty(private val isDebug: Boolean) :
    PropertyDelegateProvider<Any?, ReadOnlyProperty<Any?, String>> {

    override fun provideDelegate(
        thisRef: Any?,
        property: KProperty<*>
    ): ReadOnlyProperty<Any?, String> {
        return ReadOnlyProperty { _, _ ->
            if (isDebug) "1.0.0.Debug" else "1.0.0"
        }
    }
}

class AppInfo(isDebug: Boolean) {
    val version: String by VersionProperty(isDebug)
}
```

它常用于根据属性名、注解、运行环境等信息，动态决定真正使用哪个委托对象。

## 什么时候适合使用委托

委托很好用，但不适合为了炫技到处使用。它更适合这些场景：

- 多个类共享同一套接口实现，可以用类委托减少转发代码。
- 属性访问需要统一处理，例如懒加载、监听变化、校验赋值。
- 动态数据需要映射成对象属性，例如配置或 JSON。
- 希望把复杂的属性逻辑从业务类里拆出去。

如果一个属性只是普通字段，直接写字段会更清晰。

## 总结

Kotlin 委托的核心是 `by`：`by` 后面的对象负责真正处理被委托出去的行为。

- 类委托把接口实现交给另一个对象。
- 属性委托把 getter/setter 交给委托对象。
- `lazy` 用于延迟初始化只读属性。
- `observable` 用于监听属性赋值后的变化。
- `vetoable` 用于决定一次赋值是否生效。
- `Map` 委托可以把 key-value 数据映射成对象属性。
- `provideDelegate()` 可以在属性绑定委托时介入。

理解委托后，再看 Kotlin 标准库和 Android 开发中大量的 `by lazy`、`by viewModels()`、`by remember`，就会更容易明白它们背后真正做的事情。

## 参考

- [Delegated properties](https://kotlinlang.org/docs/delegated-properties.html)
- [Kotlin | 委托机制 & 原理 & 应用](https://juejin.cn/post/6958346113552220173)
- [类声明的右边也能写 by？Kotlin 的接口委托是这么用的](https://rengwuxian.com/delegation/)
