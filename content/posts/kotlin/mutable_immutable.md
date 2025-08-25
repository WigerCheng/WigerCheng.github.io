---
date: "2025-08-19T15:09:24+08:00"
title : 'Kotlin的可变性与不可变性'
---
## 基本类型

```kotlin
var a: Int = 1  
printInfo("old_a", a)  
val b: Int = a  
printInfo("old_b", b)  
a = 2  
printInfo("a", a)  
printInfo("b", b)
```

```java
int a = 1;  
printInfo("old_a", a);  
int b = a;  
printInfo("old_b", a);  
a = 2;  
printInfo("a", a);  
printInfo("b", b);
```

```bash
【284720968】old_a:1
【284720968】old_b:1
【511754216】a:2
【284720968】b:1
```

1. 创建一个Int对象值为1，其存放在内存的内存地址是*284720968*。
2. 创建一个可变引用a指向地址*284720968*。
3. 创建一个不可变引用b，并赋值a当前所持有的值1的地址*284720968*。
4. 创建一个Int对象值为2，假设其存放在内存的内存地址是*511754216*。
5. a是可变引用，将a=2，它的引用被更新了，现在的a指向的地址是*511754216*，现在a的值=2。
6. b是不可变引用，b仍然指向*284720968*即b的值仍然是1。

## 集合

Kotlin 的集合库对可变性和不可变性有清晰的区分。它为许多集合类型提供了两个接口：一个只读接口 (不可变) 和一个可写接口 (可变)。

### 不可变集合

```kotlin
val immutableList: List<Int> = listOf(1, 2, 3)  
printInfo("immutableList", immutableList)  
var list1: List<Int> = immutableList  
printInfo("old_list1", list1)  
val list2: List<Int> = list1  
printInfo("old_list2", list2)  
list1 += 4  
printInfo("list1", list1)  
printInfo("list2", list2) 
```

```java
Integer[] var3 = new Integer[]{1, 2, 3};  
List immutableList = CollectionsKt.listOf(var3);  
printInfo("immutableList", immutableList);  
printInfo("old_list1", immutableList);  
printInfo("old_list2", immutableList);  
List var10 = CollectionsKt.plus((Collection)immutableList, 4);  
printInfo("list1", var10);  
printInfo("list2", immutableList);  
```

```bash
【930990596】immutableList:[1, 2, 3]
【930990596】old_list1:[1, 2, 3]
【930990596】old_list2:[1, 2, 3]
【94438417】list1:[1, 2, 3, 4]
【930990596】list2:[1, 2, 3]
```

1. 创建一个不可变的列表immutableList值是\[1,2,3\]，其存放在内存的内存地址是*930990596*。
2. 创建一个可变引用list1，并赋值其等于immutableList，即指向地址是*930990596*。
3. 创建一个不可变引用list2，并赋值其等于list1，即指向地址是*930990596*。
4. 调用`list1 += 4`，即创建了一个新的不可变列表templist，值是\[1,2,3,4\]，其存放在内存的内存地址是*94438417*。
5. list1 是可变引用，将list1赋值等于templist，它的引用被更新了，现在的list1指向的地址是*94438417*，现在list1的值=\[1,2,3,4\]。
6. list2是不可变引用，list2仍然指向*930990596*即list2的值仍然是\[1,2,3\]。
7. 从Java代码就能看到，实际上创建了两个不同的不可变列表`immutableList`和`var10`，list1就是`var10`，list2就是`immutableList`。

### 可变集合

```kotlin
val mutableList: MutableList<Int> = mutableListOf(1, 2, 3)  
printInfo("mutableList", mutableList)  
val list3: MutableList<Int> = mutableList  
printInfo("old_list3", list3)  
val list4: MutableList<Int> = list3  
printInfo("old_list4", list4)  
list3 += 4  
printInfo("list3", list3)  
printInfo("list4", list4)  
```

```java
Integer[] var6 = new Integer[]{1, 2, 3};  
List mutableList = CollectionsKt.mutableListOf(var6);  
printInfo("mutableList", mutableList);  
printInfo("old_list3", mutableList);  
printInfo("old_list4", mutableList);   
((Collection)mutableList).add(4);  
printInfo("list3", mutableList);  
printInfo("list4", mutableList);  
```

```bash
【1109371569】mutableList:[1, 2, 3]
【1109371569】old_list3:[1, 2, 3]
【1109371569】old_list4:[1, 2, 3]
【1109371569】list3:[1, 2, 3, 4]
【1109371569】list4:[1, 2, 3, 4]
```

- MutableList是可变集合，val指向的内容如果是可变的话，对象内容可以改。即**引用不可变，内容可变**。而`list3 += 4`实际上是调用了`Collection.add`的方法，并没有生成新的对象引用，仍然指向的是*1109371569*这个集合实例。

## Map

- Map和集合类似，也分为可变和不可变Map。

### 不可变Map

```kotlin
val immutableMap: Map<String, Int> = mapOf("aaa" to 10)  
printInfo("immutableMap", immutableMap)  
var map1 = immutableMap  
printInfo("old_map1", map1)  
val map2 = map1  
printInfo("old_map2", map2)  
map1 = mapOf("aaa" to 20)  
printInfo("map1", map1)  
printInfo("map2", map2)
```

```java
Map immutableMap = MapsKt.mapOf(TuplesKt.to("aaa", 10));  
printInfo("immutableMap", immutableMap);  
printInfo("old_map1", immutableMap);  
printInfo("old_map2", immutableMap);  
Map map1 = MapsKt.mapOf(TuplesKt.to("aaa", 20));  
printInfo("map1", map1);  
printInfo("map2", immutableMap);
```

```bash
【1099983479】immutableMap:{aaa=10}
【1099983479】old_map1:{aaa=10}
【1099983479】old_map2:{aaa=10}
【1268447657】map1:{aaa=20}
【1099983479】map2:{aaa=10}
```

### 可变Map

```kotlin
val mutableMap: MutableMap<String, Int> = mutableMapOf("aaa" to 10)  
printInfo("mutableMap", mutableMap)  
val map3 = mutableMap  
printInfo("old_map3", map3)  
val map4 = map3  
printInfo("old_map4", map4)  
map3["aaa"] = 20  
printInfo("map3", map3)  
printInfo("map4", map4)
```

```java
Pair[] var12 = new Pair[]{TuplesKt.to("aaa", 10)};  
Map mutableMap = MapsKt.mutableMapOf(var12);  
printInfo("mutableMap", mutableMap);  
printInfo("old_map3", mutableMap);  
printInfo("old_map4", mutableMap);  
mutableMap.put("aaa", 20);  
printInfo("map3", mutableMap);  
printInfo("map4", mutableMap);
```

```bash
【1851691492】mutableMap:{aaa=10}
【1851691492】old_map3:{aaa=10}
【1851691492】old_map4:{aaa=10}
【1851691492】map3:{aaa=20}
【1851691492】map4:{aaa=20}
```

## 总结

- 使用 `val` 声明不可变引用 (推荐)，使用 `var` 声明可变引用。
- `val` 指向的对象如果是可变的 (如 `MutableList`)，对象内容依然可改。
