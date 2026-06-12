---
title: Kotlin之委托
tags:
  - Kotlin
---
>[!note] 委托
>
>- 一个对象将消息委托给另一个对象来处理。
>- Kotlin 通过 `by` 关键字可以更加优雅地实现委托。

## Kotlin的委托类型

1. **[[#类委托]]：** 一个类的方法不在该类中定义，而是直接委托给另一个对象来处理。
2. **[[#属性委托]]：** 一个类的属性不在该类中定义，而是直接委托给另一个对象来处理。
3. **[[#局部变量委托]]：** 一个局部变量不在该方法中定义，而是直接委托给另一个对象来处理。

### 类委托

- `class <类名>(b : <基础接口>) : <基础接口> by <基础对象>`

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
    consoleLog.log("Hello, Kotlin!")  
    //Console Log: Hello, Kotlin!  
    val fileLog = LogManager(FileLogger())  
    fileLog.log("Hello, Kotlin!")  
    //Saving to file: Hello, Kotlin!  
}
```

### 属性委托

- `val/var <属性名> : <类型> by <基础对象>`
- 委托类必须提供`getValue()`方法，可变属性同时必须提供`setValue()`方法。
- **在每个属性委托的实现的背后，Kotlin 编译器都会生成辅助属性并委托给它。 例如，对于属性 prop，会生成「辅助属性」 prop$delegate。**
- 而 prop 的 getter() 和 setter() 方法只是简单地委托给辅助属性的 getValue() 和 setValue() 处理。

```kotlin
源码：
class Example {
    // 被委托属性
    var prop: String by Delegate() // 基础对象
}

--------------------------------------------------------
编译器生成的字节码：
class Example {
    private val prop$delegate = Delegate()
    // 被委托属性
    var prop: String
        get() = prop$delegate.getValue(this, this:prop)
        set(value : String) = prop$delegate.setValue(this, this:prop, value)
}
```

```kotlin
class Example {
    var prop: String by Delegate()
}

class Delegate {
    private var _realValue: String = "程"

    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        println("getValue")
        return _realValue
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("setValue")
        _realValue = value
    }
}

fun main() {
    val e = Example()
    println(e.prop)
    //getValue
 //程
    e.prop = "Cheng"
 //setValue
    println(e.prop)
    //getValue
 //Cheng
}
```

- `thisRef` —— 必须与**属性所有者类型相同或者是它的超类型**。
- `property` —— 必须是**类型 `KProperty<*>`或其超类型**。
- `value` —— 必须和属性**同类型或者是它的超类型**。

#### ReadOnlyProperty / ReadWriteProperty

- 实现属性委托或局部委托时，除了定义类 Delegate 外，还可以直接使用 Kotlin 标准库中的两个接口：`ReadOnlyProperty` / `ReadWriteProperty`
- 对于 val 变量使用 `ReadOnlyProperty`，而 var 变量实现`ReadWriteProperty`，使用这两个接口可以方便地让 IDE 帮你生成函数签名。

```kotlin
val name by ReadOnlyProperty<Any?, String> { thisRef, property -> "Cheng" }
var name1 by object :ReadWriteProperty<Any?, String>{
    override fun getValue(thisRef: Any?, property: KProperty<*>): String {
      return  "Cheng"
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        
    }
}
```

## Kotlin的委托进阶

### 延迟属性委托 lazy

- lazy 是一个标准库函数，参数为一个 Lambda 表达式，返回值为一个 Lazy 实例，使用 lazy 可以实现延迟属性委托，在委托对象比较耗资源的场景会非常有用。
- **首次访问属性是，会执行 lazy 函数的 lambda 表达式并将结果记录到「背域」，后续调用 getter() 方法只是直接返回「背域」的值。**
- [[#局部变量委托]]

### 可观察属性 ObservableProperty

- 使用 `Delegates.observable()` 可以实现可观察属性，函数接受两个参数：第一个参数为初始值，第二个参数为属性值变化的回调。
- 函数的返回值是 ObservableProperty 可观察属性，它在调用 setValue(...) 是触发回调。

```kotlin
class User {
    var name: String by Delegates.observable("初始值") { prop, old, new ->
        println("旧值：$old -> 新值：$new")
    }
}

fun main(args: Array<String>) {
    val user = User()
    user.name = "第一次赋值"//旧值：初始值 -> 新值：第一次赋值
    user.name = "第二次赋值"//旧值：第一次赋值 -> 新值：第二次赋值
}
```

### 使用 Map 存储属性值

- `Map` 也可以用来实现属性委托，从而此时**字段名是 Key，属性值是 Value**。

```kotlin
val user = User(mapOf(  
    "id" to 1L,  
    "name" to "Wiger"  
))  
println(user.id)//1  
println(user.name)//Wiger  
println(user.map)//{id=1, name=Wiger}
```

## 参考

- [Kotlin | 委托机制 & 原理 & 应用](https://juejin.cn/post/6958346113552220173)
- [Android | ViewBinding 与 Kotlin 委托双剑合璧](https://juejin.cn/post/6960914424865488932 "https://juejin.cn/post/6960914424865488932")
- [类声明的右边也能写 by？Kotlin 的接口委托是这么用的](https://rengwuxian.com/delegation/)
- [Delegated properties](https://kotlinlang.org/docs/delegated-properties.html)
- [Delegating Delegates to Kotlin](https://medium.com/androiddevelopers/delegating-delegates-to-kotlin-ee0a0b21c52b)
- [Built-in Delegates](https://medium.com/androiddevelopers/built-in-delegates-4811947e781f)
- [Kotlin Vocabulary | Kotlin 委托代理](https://mp.weixin.qq.com/s/5UuXsWA0_xf9cNvRtI-fQw)

```kotlin
package io.wiger.delegation  
  
import java.text.SimpleDateFormat  
import java.util.*  
import kotlin.properties.Delegates  
import kotlin.properties.PropertyDelegateProvider  
import kotlin.properties.ReadOnlyProperty  
import kotlin.reflect.KProperty  
  
class TimeProperty(private val time: Long) : ReadOnlyProperty<Long?, String> {  
    override fun getValue(thisRef: Long?, property: KProperty<*>): String {  
        val format = SimpleDateFormat("yyyy-MM-dd-HH:mm:ss", Locale.getDefault()).format(Date(time))  
        return "time:$format"  
    }  
}  
  
class Info(  
    val isDebug: Boolean = false  
) {  
    val version: String by VersionProperty(isDebug)  
  
    override fun toString(): String {  
        return "isDebug:${isDebug} version:${version}"  
    }  
}  
  
abstract class GetVersionProperty : ReadOnlyProperty<Any?, String>  
  
class DebugProperty : GetVersionProperty() {  
    override fun getValue(thisRef: Any?, property: KProperty<*>): String = "1.0.0.Debug"  
}  
  
class ReleaseProperty : GetVersionProperty() {  
    override fun getValue(thisRef: Any?, property: KProperty<*>): String = "1.0.0"  
}  
  
class VersionProperty(private val isDebug: Boolean) : PropertyDelegateProvider<Any?, GetVersionProperty> {  
    override fun provideDelegate(  
        thisRef: Any?,  
        property: KProperty<*>  
    ): GetVersionProperty {  
        return if (isDebug) DebugProperty() else ReleaseProperty()  
    }  
}  
  
fun main() {  
    val a by TimeProperty(System.currentTimeMillis())  
    println(a)  
    val b by VersionProperty(true)  
    println(b)  
    val c by VersionProperty(false)  
    println(c)  
    val d = Info(false)  
    println(d)  
    val e = Info(true)  
    println(e)  
    var f by Delegates.vetoable(0) { _, o, n ->  
        n > o  
    }  
    println(f)  
    f = 3  
    println(f)  
    f = 1  
    println(f)  
    f = 4  
    println(f)  
}
```
