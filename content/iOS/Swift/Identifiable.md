---
title: Swift的Identifiable协议
tags:
  - iOS/Swift
---
## 核心概念

`Identifiable`在Swift标准库中是一个很重要的协议。它的作用很简单，就是为对象提供一个**唯一的身份标识**。

### 协议定义

```swift
public protocol Identifiable<ID> {

    /// A type representing the stable identity of the entity associated with
    /// an instance.
    associatedtype ID : Hashable

    /// The stable identity of the entity associated with this instance.
    var id: Self.ID { get }
}
```

- **ID**: 必须遵循 `Hashable` 协议（确保 ID 可以作为字典的 Key 或在集合中进行快速查找）。
- **id**: 该属性必须是唯一的，通常由 `UUID` 或数据库主键承担。

## 实现方式

### 1. 使用UUID

>[!tip]
> 适用于内存的临时对象，确保每次创建实例都有唯一 ID。

```swift
struct User: Identifiable {
    let id = UUID()
    let name: String
}
```

### 2.使用唯一业务主键

>[!tip]
> 使用于服务器返回或数据库的唯一实体主键

```swift
struct Product: Identifiable {
    let id: Int
    let title: String
}
```

## 使用场景

### 1. Foreach

在ForEach中，集合的元素必须遵循`Identifiable`协议，否则你需要在Foreach的构造方法中提供`id`参数。

```swift
// 方式 1：模型已遵循 Identifiable
ForEach(users) { user in
    Text(user.name)
}

// 方式 2：模型未遵循，手动指明唯一标识属性
ForEach(users, id: \.name) { user in
    Text(user.name)
}
```
