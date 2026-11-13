---
title: Kotlin 中的 val、var 与集合可变性
draft: false
tags:
  - Kotlin
---

在 Kotlin 里，`val` 和 `var` 经常被解释成“不可变”和“可变”。这个说法不算错，但很容易让人产生一个误解：只要用了 `val`，它指向的东西就一定不能改。

更准确的说法应该是：

- `val` 表示引用不可重新赋值。
- `var` 表示引用可以重新赋值。
- 对象内容能不能修改，取决于对象本身是不是可变类型。

所以，`val` 限制的是“这个变量还能不能指向别的对象”，不是“这个对象内部还能不能变化”。

```kotlin
val list = mutableListOf(1, 2, 3)
list += 4

// list = mutableListOf(5, 6) // 编译错误
```

上面这段代码里，`list` 是 `val`，所以它不能再指向另一个集合；但它指向的是 `MutableList`，集合内部的元素仍然可以被修改。

## val 和 var 控制的是引用

```kotlin
var a: Int = 1
val b: Int = a

a = 2

println(a) // 2
println(b) // 1
```

这段代码很简单，但能说明 `val` 和 `var` 的核心区别。

`a` 是 `var`，所以可以从 `1` 重新赋值为 `2`。`b` 是 `val`，它在初始化时拿到的是 `a` 当时的值，也就是 `1`。后面 `a = 2` 改变的是 `a` 这个变量的值，不会影响 `b`。

如果把它放到引用的角度理解，可以认为：

- `var` 允许变量重新绑定到另一个值。
- `val` 只允许变量绑定一次。

不过对于 `Int`、`String` 这类基础类型或不可变类型，通常不需要过度纠结它们在 JVM 上到底是装箱还是拆箱。理解重点是：重新赋值影响的是变量本身，不会自动影响其他已经拿到值的变量。

## val 不等于对象不可变

真正容易混淆的是对象，尤其是集合。

```kotlin
data class User(var name: String)

val user = User("Wiger")
user.name = "Kotlin"

// user = User("Android") // 编译错误
```

这里 `user` 是 `val`，所以不能再让它指向另一个 `User` 对象。但是 `User.name` 是 `var`，因此可以修改这个对象内部的状态。

这就是 Kotlin 可变性里最重要的一层区分：

- 变量引用是否可变：由 `val` / `var` 决定。
- 对象内部状态是否可变：由对象自身的 API 决定。

如果希望对象内部也不可变，应该让对象的属性也尽量使用 `val`：

```kotlin
data class User(val name: String)
```

这样 `user.name = "Kotlin"` 也无法编译通过。

## Kotlin 集合的两套接口

Kotlin 集合库把集合接口分成了只读接口和可变接口：

| 只读接口    | 可变接口           |
| ----------- | ------------------ |
| `List<T>`   | `MutableList<T>`   |
| `Set<T>`    | `MutableSet<T>`    |
| `Map<K, V>` | `MutableMap<K, V>` |

这里有一个细节：`List<T>` 更准确地说是“只读接口”，而不是绝对意义上的“不可变集合”。它不提供 `add()`、`remove()` 这类修改方法，所以你不能通过 `List<T>` 这个引用去修改集合。

但如果它背后实际指向的是一个 `MutableList`，这个集合仍然可能被其他可变引用修改。

```kotlin
val mutable = mutableListOf(1, 2, 3)
val readonly: List<Int> = mutable

mutable += 4

println(readonly) // [1, 2, 3, 4]
```

`readonly` 的类型是 `List<Int>`，它自己不能调用 `add()`；但它和 `mutable` 指向的是同一个集合对象。因此当 `mutable` 修改了集合内容后，`readonly` 读到的结果也会变化。

这也是为什么在 Kotlin 里要区分两个概念：

- 只读视图：当前引用不能修改对象。
- 真正不可变：对象本身没有任何修改入口。

Kotlin 标准库的 `List`、`Set`、`Map` 主要提供的是只读视图。

## List 使用 += 时发生了什么

```kotlin
var list1: List<Int> = listOf(1, 2, 3)
val list2: List<Int> = list1

list1 += 4

println(list1) // [1, 2, 3, 4]
println(list2) // [1, 2, 3]
```

这段代码看起来像是在原集合上添加元素，但其实不是。

因为 `list1` 的类型是 `List<Int>`，它没有 `add()` 方法。这里的 `list1 += 4` 会被理解成：

```kotlin
list1 = list1 + 4
```

也就是：

1. 基于原来的 `list1` 创建一个新集合。
2. 新集合中包含原来的元素和新元素 `4`。
3. 把 `list1` 重新指向这个新集合。

所以 `list1` 变成了 `[1, 2, 3, 4]`，而 `list2` 仍然指向旧集合 `[1, 2, 3]`。

这也是为什么这里的 `list1` 必须是 `var`。如果写成 `val`，就无法把新集合重新赋值给它：

```kotlin
val list: List<Int> = listOf(1, 2, 3)
// list += 4 // 编译错误
```

## MutableList 使用 += 时发生了什么

```kotlin
val list3: MutableList<Int> = mutableListOf(1, 2, 3)
val list4: MutableList<Int> = list3

list3 += 4

println(list3) // [1, 2, 3, 4]
println(list4) // [1, 2, 3, 4]
```

这次结果不同，因为 `list3` 的类型是 `MutableList<Int>`。它有 `add()` 方法，所以 `list3 += 4` 会修改原集合，效果类似：

```kotlin
list3.add(4)
```

`list3` 和 `list4` 指向同一个 `MutableList` 对象，因此通过 `list3` 修改集合后，通过 `list4` 也能看到变化。

这正是 `val` 最容易误导人的地方：

```kotlin
val list = mutableListOf(1, 2, 3)
list += 4 // 可以，修改集合内容

// list = mutableListOf(5, 6) // 不可以，val 不能重新赋值
```

`val` 只是让 `list` 这个引用不能换目标，但没有冻结它指向的集合对象。

## Map 也是一样

```kotlin
var map1: Map<String, Int> = mapOf("score" to 10)
val map2 = map1

map1 = mapOf("score" to 20)

println(map1) // {score=20}
println(map2) // {score=10}
```

`Map` 是只读接口，不能通过 `map1["score"] = 20` 修改内容。如果想改变结果，只能创建一个新 Map，并让 `map1` 指向它。

而 `MutableMap` 可以直接修改原对象：

```kotlin
val map3: MutableMap<String, Int> = mutableMapOf("score" to 10)
val map4 = map3

map3["score"] = 20

println(map3) // {score=20}
println(map4) // {score=20}
```

原因和 `MutableList` 一样：`map3`、`map4` 指向同一个可变对象。

## 选择 val 还是 var

日常写 Kotlin 时，可以优先使用 `val`。这并不是因为 `val` 一定让对象不可变，而是因为它能减少“变量被重新指向别处”的可能性，让代码更容易推理。

如果一个变量初始化之后不需要再指向别的对象，就使用 `val`：

```kotlin
val name = "Kotlin"
val users = mutableListOf<User>()
```

如果变量确实需要在不同值之间切换，再使用 `var`：

```kotlin
var selectedUser: User? = null
var retryCount = 0
```

对于集合，还要看你是否希望暴露修改能力。

如果只希望外部读取，不希望外部修改，可以把可变集合藏在内部，对外暴露只读接口：

```kotlin
class UserRepository {
    private val _users = mutableListOf<User>()

    val users: List<User>
        get() = _users

    fun addUser(user: User) {
        _users += user
    }
}
```

这样外部拿到的是 `List<User>`，不能直接添加或删除元素；而类内部仍然可以通过 `_users` 维护数据。

## 总结

`val` 和 `var` 讨论的是变量引用，不是对象本身。

- `val`：引用不可重新赋值。
- `var`：引用可以重新赋值。
- `List`、`Set`、`Map`：只读接口，不暴露修改方法。
- `MutableList`、`MutableSet`、`MutableMap`：可变接口，可以修改对象内容。
- `List += item`：通常创建新集合，再重新赋值。
- `MutableList += item`：通常调用 `add()`，修改原集合。

写 Kotlin 时，推荐默认使用 `val`，需要重新赋值时再使用 `var`；默认对外暴露只读集合，确实需要外部修改时再暴露 `MutableList`、`MutableMap`。这样代码的可变范围更小，也更容易维护。
