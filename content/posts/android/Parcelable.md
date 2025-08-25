---
date: "2022-03-16T00:00:00Z"
title: Android序列化之Parcelable
---

## 1、什么是Parcel

> Container for a message (data and object references) that can be sent through an IBinder. A Parcel can contain both flattened data that will be unflattened on the other side of the IPC (using the various methods here for writing specific types, or the general `Parcelable` interface), and references to live `IBinder` objects that will result in the other side receiving a proxy IBinder connected with the original IBinder in the Parcel.
>
> [Parcel](https://developer.android.google.cn/reference/android/os/Parcel?hl=zh_cn)

- 一个在IBinder中传输的消息容器。
- 进程间通信，发送端打包数据，接收端解包数据，实现数据传输。
- `Parcel` 不是通用序列化机制，您绝不能将任何 `Parcel` 数据存储在磁盘上或通过网络发送。

## 2、Parcel支持的类型

- Byte, ByteArray
- Double, DoubleArray
- Float, FloatArray
- Int, IntArray
- Long, LongArray
- String, StringArray
- [...](https://developer.android.google.cn/reference/android/os/Parcel?hl=zh_cn#primitives)

## 3、实现Parcelable(手动实现Parcelable接口)

```kotlin
import android.os.Parcel
import android.os.Parcelable

data class Pet(
    val name: String
) : Parcelable {
    constructor(parcel: Parcel) : this(parcel.readString() ?: "pet")

    override fun writeToParcel(parcel: Parcel, flags: Int) {
        parcel.writeString(name)
    }

    override fun describeContents(): Int {
        return 0
    }

    companion object CREATOR : Parcelable.Creator<Pet> {
        override fun createFromParcel(parcel: Parcel): Pet {
            return Pet(parcel)
        }

        override fun newArray(size: Int): Array<Pet?> {
            return arrayOfNulls(size)
        }
    }
}

data class Person(
    val id: Long,
    var name: String,
    var age: Int,
    var isStudent: Boolean,
    var pets: List<Pet>
) : Parcelable {
    constructor(parcel: Parcel) : this(
        parcel.readLong(),
        parcel.readString() ?: "",
        parcel.readInt(),
        parcel.readByte() != 0.toByte(),
        parcel.createTypedArrayList(Pet) ?: emptyList()
    )

    override fun writeToParcel(parcel: Parcel, flags: Int) {
        parcel.writeLong(id)
        parcel.writeString(name)
        parcel.writeInt(age)
        parcel.writeByte(if (isStudent) 1 else 0)
        parcel.writeTypedList(pets)
    }

    override fun describeContents(): Int {
        return 0
    }

    companion object CREATOR : Parcelable.Creator<Person> {
        override fun createFromParcel(parcel: Parcel): Person {
            return Person(parcel)
        }

        override fun newArray(size: Int): Array<Person?> {
            return arrayOfNulls(size)
        }
    }

}
```

## 4、实现Parcelable（通过 Parcelable 实现生成器 实现）

[Parcelable 实现生成器](https://developer.android.google.cn/kotlin/parcelize?hl=zh-cn)

1. 添加 `kotlin-parcelize` 插件。

   ``` groovy
   plugins {
       id 'kotlin-parcelize'
   }
   ```

   ``` groovy
   apply plugin: "kotlin-parcelize"
   ```

2. 编写数据类

   ``` kotlin
   @Parcelize
   data class Person(
    val id: Long,
    var name: String,
    var age: Int,
    var isStudent: Boolean,
    var pets: List<Pet>
   ) : Parcelable
   ```

### @Parcelize 支持的类型

- 基元类型（及其 boxed 版本）

- 对象和枚举

- `String`、`CharSequence`

- `Exception`

- `Size`、`SizeF`、`Bundle`、`IBinder`、`IInterface`、`FileDescriptor`

- `SparseArray`、`SparseIntArray`、`SparseLongArray`、`SparseBooleanArray`

- 所有 `Serializable`（包括 `Date`）和 `Parcelable` 实现

- 所有受支持类型的集合：

  `List`（映射到 `ArrayList`）、`Set`（映射到 `LinkedHashSet`）、`Map`（映射到 `LinkedHashMap`）

  - 此外，还有一些具体实现：`ArrayList`、`LinkedList`、`SortedSet`、`NavigableSet`、`HashSet`、`LinkedHashSet`、`TreeSet`、`SortedMap`、`NavigableMap`、`HashMap`、`LinkedHashMap`、`TreeMap`、`ConcurrentHashMap`

- 所有受支持类型的数组

- 所有受支持类型的可为 null 版本

### 自定义Parceler

``` kotlin
data class Phone(
    val model: String
)

object PhoneClassParceler : Parceler<Phone> {
    override fun create(parcel: Parcel): Phone {
        return Phone(parcel.readString() ?: "iPhone")
    }

    override fun Phone.write(parcel: Parcel, flags: Int) {
        parcel.writeString(model)
    }

}

@Parcelize
@TypeParceler<Phone, PhoneClassParceler>()
data class PhoneOwner(val phone: Phone) : Parcelable

@Parcelize
data class PhoneOwner1(@TypeParceler<Phone, PhoneClassParceler>() val phone: Phone) : Parcelable

@Parcelize
data class PhoneOwner2(val phone: @WriteWith<PhoneClassParceler>() Phone) : Parcelable
```
