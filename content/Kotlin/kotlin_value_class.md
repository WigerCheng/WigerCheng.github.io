---
title: Kotlin Value Class：用低成本类型包装提升代码语义
tags:
  - Kotlin
---

在业务代码里，我们经常会遇到这种情况：很多数据底层类型都一样，但业务含义完全不同。

```kotlin
fun loadUser(id: Long) {
}

fun loadOrder(id: Long) {
}
```

`UserId` 和 `OrderId` 底层可能都是 `Long`，但它们显然不是一回事。如果所有地方都直接使用 `Long`，编译器就无法帮我们发现“把订单 ID 传给用户接口”这种错误。

Kotlin 的 Value Class 就是为这类场景准备的：它可以创建一个有业务语义的新类型，同时在很多运行时场景下尽量使用底层类型表示，减少普通包装类的额外开销。

## 从普通包装类说起

我们当然可以用普通类包装一个 ID：

```kotlin
data class UserId(val value: Long)
data class OrderId(val value: Long)

fun loadUser(id: UserId) {
}

fun loadOrder(id: OrderId) {
}
```

这样一来，`UserId` 和 `OrderId` 就变成了两个不同类型，类型系统可以帮我们拦住误传参数。

问题是，普通类通常意味着额外对象。对于 ID、金额、单位这类轻量包装，很多时候我们只是想增加类型语义，并不一定希望每次都创建包装对象。

Value Class 的目标就是解决这个矛盾：既保留类型语义，又尽量减少运行时包装成本。

## 声明 Value Class

在 JVM 平台上，声明 Value Class 需要使用 `value class` 和 `@JvmInline`：

```kotlin
@JvmInline
value class UserId(val value: Long)
```

它有几个基本限制：

- 主构造函数有且只能有一个属性。
- 主构造属性必须使用 `val`。
- JVM 平台需要添加 `@JvmInline` 注解。

使用时，它看起来像一个普通类型：

```kotlin
@JvmInline
value class Password(val value: String)

fun login(password: Password) {
}

fun main() {
    login(Password("123456"))
}
```

`Password` 和 `String` 不再是同一个类型，所以不能直接把普通字符串传给 `login()`。这让函数签名更有业务含义，也更不容易被误用。

## Value Class 可以有哪些成员

Value Class 可以声明方法、计算属性和 `init` 初始化块。

```kotlin
@JvmInline
value class Name(val value: String) {
    init {
        require(value.isNotBlank())
    }

    val length: Int
        get() = value.length

    fun greet() {
        println("Hello, $value")
    }
}
```

这里的 `length` 是计算属性，它没有自己的 backing field，只是基于主构造属性 `value` 计算结果。

Value Class 不能有额外的幕后字段：

```kotlin
@JvmInline
value class Name(val value: String) {
    // var cached: String = value // 不允许
}
```

原因是 Value Class 的设计目标是轻量包装。如果它可以持有额外状态，就会破坏这种模型。

## Value Class 可以实现接口

Value Class 不能继承其他类，也不能被继承。它是 final 的。

但是它可以实现接口：

```kotlin
interface Printable {
    fun prettyPrint(): String
}

@JvmInline
value class Name(val value: String) : Printable {
    override fun prettyPrint(): String {
        return "Name($value)"
    }
}
```

这让 Value Class 既能保持轻量，又能接入已有的接口抽象。

## 运行时表现：boxed 与 unboxed

Value Class 在运行时可能有两种表现形式：

- `unboxed`：直接表现为底层类型。
- `boxed`：表现为包装类。

Kotlin 编译器会尽量使用 `unboxed` 形式来减少对象创建，但在某些情况下必须使用 `boxed`。

一个常用判断方式是：当 Value Class 被当作它自己使用时，通常可以 unboxed；当它被当作其他类型使用时，通常需要 boxed。

```kotlin
interface I

@JvmInline
value class Foo(val value: Int) : I

fun asValueClass(foo: Foo) {}
fun asInterface(i: I) {}
fun asNullable(foo: Foo?) {}
fun <T> asGeneric(value: T) {}

fun main() {
    val foo = Foo(42)

    asValueClass(foo) // 通常可以 unboxed
    asInterface(foo) // boxed，因为被当作接口 I 使用
    asNullable(foo) // boxed，因为 Foo 和 Foo? 是不同类型
    asGeneric(foo) // boxed，因为被当作泛型 T 使用
}
```

这也是 Value Class 和普通类最不一样的地方：它在源码层面是一个独立类型，但运行时并不总是一个独立对象。

因此 Kotlin 禁止对 Value Class 使用引用相等 `===`。因为它可能是底层类型，也可能是包装对象，引用相等没有稳定意义。

## JVM 方法名冲突

Value Class 在 JVM 上可能会被编译成底层类型，这会带来方法签名冲突问题。

```kotlin
@JvmInline
value class UInt(val value: Int)

fun compute(value: Int) {
}

fun compute(value: UInt) {
}
```

从 Kotlin 角度看，这两个函数参数类型不同；但在 JVM 上，`UInt` 可能表现为 `Int`，两个方法就可能都接近 `compute(int)`，产生签名冲突。

为了解决这个问题，Kotlin 编译器会对涉及 Value Class 的函数名做 name mangling，也就是给方法名加上一段稳定后缀，避免 JVM 签名冲突。

这个机制对 Kotlin 调用方通常是透明的，但对 Java 调用方可能不够友好。此时可以用 `@JvmName` 指定 Java 侧看到的方法名：

```kotlin
@JvmInline
value class UInt(val value: Int)

fun compute(value: Int) {
}

@JvmName("computeUInt")
fun compute(value: UInt) {
}
```

这样 Java 调用时就可以使用更明确的方法名。

## 适合使用 Value Class 的场景

Value Class 最适合用来表达“底层类型一样，但业务含义不同”的值。

### ID 类型

```kotlin
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class OrderId(val value: Long)

fun loadUser(id: UserId) {
}

fun loadOrder(id: OrderId) {
}
```

`UserId` 和 `OrderId` 都是 `Long`，但它们不能混用。

### 单位类型

```kotlin
@JvmInline
value class Pixels(val value: Int)

@JvmInline
value class Seconds(val value: Int)
```

这样可以避免把秒数当成像素传进去。

### 业务包装类型

```kotlin
@JvmInline
value class Email(val value: String)

@JvmInline
value class Token(val value: String)
```

虽然底层都是 `String`，但类型名能表达更明确的业务含义。

## 不适合使用 Value Class 的场景

Value Class 也不是普通类的替代品。

如果一个类型需要多个字段、可变状态、继承体系，或者需要稳定的对象身份，那就应该使用普通类或 data class。

不适合的例子：

```kotlin
// Value Class 不适合表达这种多字段对象
data class User(
    val id: Long,
    val name: String,
    val age: Int
)
```

Value Class 更像是“给单个值加一层类型语义”，而不是完整的领域对象模型。

## Value Class 的限制

常见限制可以总结为：

- 主构造函数只能有一个属性。
- 主构造属性必须是 `val`。
- 不能声明带 backing field 的额外属性。
- 不能继承其他类。
- 不能被其他类继承。
- 不能使用引用相等 `===`。
- 在泛型、接口、可空类型等场景下可能发生装箱。

这些限制都和它的设计目标有关：Value Class 要尽可能轻量，因此不能像普通类一样随意保存状态或参与复杂继承。

## 总结

Value Class 的核心价值是：用低成本的方式，让基础类型拥有更明确的业务语义。

它适合包装 ID、单位、Token、Email 等单值类型。这样既能让函数签名更清晰，也能让编译器帮我们发现误传参数的问题。

不过，Value Class 并不等于“永远没有对象开销”。在泛型、接口、可空类型等场景下，它仍然可能 boxed；在 JVM 互操作时，也需要注意方法名冲突和 `@JvmName`。

如果一个类型只是想给 `String`、`Int`、`Long` 这类基础值增加业务含义，Value Class 很适合；如果它需要多个字段和复杂状态，普通类会更合适。

## 参考

- [Inline value classes | Kotlin Documentation](https://kotlinlang.org/docs/inline-classes.html)
- [Design Notes on Kotlin Value Classes](https://github.com/Kotlin/KEEP/blob/master/notes/value-classes.md#design-notes-on-kotlin-value-classes)
