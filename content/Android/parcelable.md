---
title: Android 序列化之 Parcelable
---

## 📌 概述与核心结论

在 Android 开发中，数据在进程间通信（IPC，如 Activity/Service 间传递 Intent/Bundle）需要进行序列化。Android 提供了专门的高性能序列化机制 —— **`Parcelable`**。

> [!important] **核心要点一句话总结**
> `Parcelable` 是 Android 为**内存中高效进程间通信 (IPC)** 量身打造的序列化接口。通过 C++ 底层连续内存（Parcel）的读写，避免了反射和频繁磁盘 I/O。**切记：Parcel 仅限于内存传输，绝不能用于磁盘持久化或网络传输！**

---

## ⚔️ Parcelable vs Serializable 对比

| 比较维度 | Parcelable | Serializable |
| :--- | :--- | :--- |
| **设计初衷** | Android 专门为 IPC 内存传输定制 | Java 原生通用序列化接口 |
| **底层原理** | 基于 C++ `Parcel` 在内存中直接按顺序读写二进制流 | 基于 Java 反射机制和频繁的临时对象创建与 I/O |
| **性能/开销** | **极高**（性能提升可达数倍至数十倍），低内存开销 | **较低**，大量产生反射和临时对象，易引发 GC |
| **持久化支持** | **不支持**（系统版本或结构变动会导致二进制解析崩溃） | **支持**（可通过 `serialVersionUID` 维护跨版本兼容） |
| **编写复杂度** | 需手写模版代码（可用 `@Parcelize` 自动生成） | 极简（只需实现空标记接口 `Serializable`） |

---

## 🧱 什么是 Parcel

> "Container for a message (data and object references) that can be sent through an IBinder..."
> — [Android 官方 Parcel 文档](https://developer.android.google.cn/reference/android/os/Parcel?hl=zh_cn)

`Parcel` 是一个数据和对象引用的传输容器：

- **IPC 消息载体**：配合 `IBinder` 机制，发送端将数据打平打包（flatten），接收端从内存流中解包（unflatten）。
- **支持引用传输**：除了基本类型外，还可以传输活的 `IBinder` 对象（在接收端自动转换为 Binder 代理对象）。
- **内存限制**：Binder 事务缓冲区大小有限制（通常总计为 1MB，由该进程所有正在进行的 IPC 共享），大图或超长文本切勿存入 `Parcel`，否则会触发 `TransactionTooLargeException`。

---

## ✍️ 方式一：手动实现 `Parcelable` 接口

手动实现需要编写 3 个关键部分：

1. `writeToParcel(parcel, flags)`：将对象数据按顺序写入 `Parcel`。
2. `describeContents()`：描述内容类型（通常返回 `0`；若包含文件描述符则返回 `CONTENTS_FILE_DESCRIPTOR` 即 `1`）。
3. `companion object CREATOR : Parcelable.Creator<T>`：包含从 `Parcel` 构造对象的反序列化逻辑。

### 完整代码示例

```kotlin
import android.os.Parcel
import android.os.Parcelable

// 嵌套的小型数据类
data class Pet(
    val name: String
) : Parcelable {
    // 反序列化构造函数：顺序必须与 writeToParcel 一致
    constructor(parcel: Parcel) : this(
        parcel.readString() ?: ""
    )

    override fun writeToParcel(parcel: Parcel, flags: Int) {
        parcel.writeString(name)
    }

    override fun describeContents(): Int = 0

    companion object CREATOR : Parcelable.Creator<Pet> {
        override fun createFromParcel(parcel: Parcel): Pet = Pet(parcel)
        override fun newArray(size: Int): Array<Pet?> = arrayOfNulls(size)
    }
}

// 主数据类
data class Person(
    val id: Long,
    var name: String,
    var age: Int,
    var isStudent: Boolean,
    var pets: List<Pet>
) : Parcelable {

    // 反序列化构造函数
    constructor(parcel: Parcel) : this(
        id = parcel.readLong(),
        name = parcel.readString() ?: "",
        age = parcel.readInt(),
        // Boolean 序列化通常转为 Byte/Int (1 代表 true, 0 代表 false)
        isStudent = parcel.readByte() != 0.toByte(),
        // 嵌套 Parcelable 列表的反序列化
        pets = parcel.createTypedArrayList(Pet) ?: emptyList()
    )

    // 序列化写入
    override fun writeToParcel(parcel: Parcel, flags: Int) {
        parcel.writeLong(id)
        parcel.writeString(name)
        parcel.writeInt(age)
        parcel.writeByte(if (isStudent) 1.toByte() else 0.toByte())
        // 写入 Parcelable 类型的集合
        parcel.writeTypedList(pets)
    }

    override fun describeContents(): Int = 0

    companion object CREATOR : Parcelable.Creator<Person> {
        override fun createFromParcel(parcel: Parcel): Person = Person(parcel)
        override fun newArray(size: Int): Array<Person?> = arrayOfNulls(size)
    }
}
```

> [!warning] **手动实现的坑点**
> 
> 1. **读写顺序一致**：`readXxx()` 的顺序必须与 `writeXxx()` 的顺序 **100% 精确匹配**，否则会导致数据交错错乱甚至崩溃。
> 2. **Nullable 安全**：`readString()` 等可能返回 `null`，务必做好空安全兜底（例如 `?: ""`）。
> 3. **Boolean 处理**：Parcel 缺乏原生 `writeBoolean` 简易 API（早期版本），习惯上使用 `writeByte(1/0)` 或 `writeInt(1/0)` 替代。

---

## 🚀 方式二：使用 `@Parcelize` 自动生成器

Android 官方推荐在 Kotlin 中使用 `kotlin-parcelize` 编译器插件，它会在编译期自动生成 `writeToParcel` 和 `CREATOR` 代码。

### 1. 配置 Gradle 插件

在新版 Gradle 中使用 `plugins` 块引入：

```groovy
// build.gradle (Module 级)
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
    id 'kotlin-parcelize'
}
```

*(老版本使用 `apply plugin: "kotlin-parcelize"`)`*

### 2. 编写数据类

只需添加 `@Parcelize` 注解并实现 `Parcelable` 接口即可：

```kotlin
import android.os.Parcelable
import kotlinx.parcelize.Parcelize

@Parcelize
data class Person(
    val id: Long,
    var name: String,
    var age: Int,
    var isStudent: Boolean,
    var pets: List<Pet>
) : Parcelable
```

### `@Parcelize` 原生支持的数据类型

`kotlin-parcelize` 插件功能非常强大，默认开箱支持丰富的数据类型：

- **基元类型及包装类**：`Int`, `Long`, `Float`, `Double`, `Boolean`, `Byte`, `Char` 等。
- **对象与枚举**：`String`, `CharSequence`, `Enum` 全系列。
- **Android 框架特定类型**：`Bundle`, `IBinder`, `IInterface`, `FileDescriptor`, `Size`, `SizeF` 等。
- **系统稀疏数组**：`SparseArray`, `SparseIntArray`, `SparseLongArray`, `SparseBooleanArray` 等。
- **异常与接口实现**：`Exception`、任何 `Serializable`（如 `Date`）及其他 `Parcelable` 实现。
- **集合类型**：
  - 标准接口：`List` (映射为 `ArrayList`), `Set` (映射为 `LinkedHashSet`), `Map` (映射为 `LinkedHashMap`)。
  - 具体实现：`ArrayList`, `LinkedList`, `HashSet`, `LinkedHashSet`, `TreeSet`, `HashMap`, `LinkedHashMap`, `ConcurrentHashMap` 等。
- **数组与 Nullable**：上述所有支持类型的数组以及可空（`T?`）版本。

---

## 🛠️ 进阶技巧：自定义 `Parceler`

当某个字段的类型来自第三方库（既不能修改源码添加 `@Parcelize`，也没有默认实现 `Parcelable`）时，可以通过自定义 `Parceler<T>` 来指定读写规则。

### 1. 定义 Parceler 实现

```kotlin
import android.os.Parcel
import kotlinx.parcelize.Parceler

// 假设 Phone 是无法修改源码第三方数据类
data class Phone(val model: String)

// 定义第三方类的序列化与反序列化逻辑
object PhoneClassParceler : Parceler<Phone> {
    override fun create(parcel: Parcel): Phone {
        return Phone(model = parcel.readString() ?: "Unknown")
    }

    override fun Phone.write(parcel: Parcel, flags: Int) {
        parcel.writeString(model)
    }
}
```

### 2. 三种作用域注解绑定方式

```kotlin
import kotlinx.parcelize.Parcelize
import kotlinx.parcelize.TypeParceler
import kotlinx.parcelize.WriteWith

// 方式 1：类级别注解（作用于该类中所有指定类型字段）
@Parcelize
@TypeParceler<Phone, PhoneClassParceler>()
data class PhoneOwner1(val phone: Phone) : Parcelable

// 方式 2：构造参数级别注解（作用于特定属性）
@Parcelize
data class PhoneOwner2(
    @TypeParceler<Phone, PhoneClassParceler>() val phone: Phone
) : Parcelable

// 方式 3：类型行内注解（使用 @WriteWith 语法糖）
@Parcelize
data class PhoneOwner3(
    val phone: @WriteWith<PhoneClassParceler>() Phone
) : Parcelable
```

---

## ⚠️ 踩坑指南与最佳实践

1. **绝对不要将 Parcel 持久化到本地磁盘**
   - `Parcel` 的二进制布局并不保证跨设备、跨 Android 版本的向前/向后兼容性。升级 App 或 OS 版本后反序列化可能导致应用崩溃。本地文件或数据库存储请选用 JSON、Protobuf 或 Room。
2. **警惕 `TransactionTooLargeException`**
   - 单个 Binder 事务数据上限为 1MB。传递大量 List 数据或 Bitmap 时，应优先考虑单例仓库、ViewModel 或本地数据库主键传输。
3. **ClassLoader 问题**
   - 在多 Bundle / 插件化框架中，反序列化带泛型的 `Parcelable` 或 `Bundle` 时，若遇到 `ClassNotFoundException`，需显示传入当前类加载器：`parcel.readParcelable(classLoader)`。

---

## 📝 总结

1. **原理**：`Parcelable` 专为内存 IPC 设计，基于 C++ 内存流的顺序读写，性能远高于基于反射的 `Serializable`。
2. **开发首选**：在 Kotlin 项目中强烈推荐使用 `@Parcelize` 注解插件，减少模版代码并规避手动按序读写的出错风险。
3. **扩展处理**：使用 `Parceler` 轻松解决第三方无接口支持类的序列化需求。
