---
tags:
  - Other
---
[文档](https://protobuf.dev/programming-guides/proto3/)

# 定义一个消息类型

```proto
syntax = "proto3";

/**
 * SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response.
 */
message SearchRequest {
  string query = 1;
  // Which page number do we want?
  int32 page_number = 2;
  // Number of results to return per page.
  int32 results_per_page = 3;
  Corpus corpus = 4;//[[#枚举]]
}

message SearchResponse {
 ...
}

```
- 第一行的`syntax = "proto3";` 指定了使用proto3的版本，如果没有指定则编译器会认为使用proto2的版本。
- `string, int32` //[[#标量值类型]]
- `Corpus corpus;` //[[#枚举]]
- `1,2,3,4`//[[#属性数字]]
- /\*\*\*/  多行注释, 
- // 单行注释
- 可以多个message写在同一个.proto文件，如代码中的`SearchResponse`。

# 属性数字
- 必须给每一个属性分配一个**`1`到`536870911`**的数字。
	- 一个message里分配的数字**必须是唯一**的。
	- 不能使用`19,000` 到 `19,999` 的数字，因为它们已经被Protocol Buffers实现占用。
	- ⚠️单纯修改属性的数字=删除这个属性同时创建一个相同类型的新的属性。
	- **属性数字不能重复使用！** 
	- 属性数字从1开始排序使用，因为1-15占用的大小是一字节，16-2047占用两个字节...

# Protocol Buffer Type
## 标量值类型
[Type表格](https://protobuf.dev/programming-guides/proto3/#scalar)

| Proto Type | Notes                                                                                                                                           |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| double     |                                                                                                                                                 |
| float      |                                                                                                                                                 |
| int32      | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. |
| int64      | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. |
| uint32     | Uses variable-length encoding.                                                                                                                  |
| uint64     | Uses variable-length encoding.                                                                                                                  |
| sint32     | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s.                            |
| sint64     | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s.                            |
| fixed32    | Always four bytes. More efficient than uint32 if values are often greater than 228.                                                             |
| fixed64    | Always eight bytes. More efficient than uint64 if values are often greater than 256.                                                            |
| sfixed32   | Always four bytes.                                                                                                                              |
| sfixed64   | Always eight bytes.                                                                                                                             |
| bool       |                                                                                                                                                 |
| string     | A string must always contain UTF-8 encoded or 7-bit ASCII text, and cannot be longer than 232.                                                  |
| bytes      | May contain any arbitrary sequence of bytes no longer than 232.                                                                                 |

| Proto Type | C++ Type    | Java/Kotlin Type[1] | Python Type[3]                   | Go Type | Ruby Type                      | C# Type    | PHP Type          | Dart Type | Rust Type   |
| ---------- | ----------- | ------------------- | -------------------------------- | ------- | ------------------------------ | ---------- | ----------------- | --------- | ----------- |
| double     | double      | double              | float                            | float64 | Float                          | double     | float             | double    | f64         |
| float      | float       | float               | float                            | float32 | Float                          | float      | float             | double    | f32         |
| int32      | int32_t     | int                 | int                              | int32   | Fixnum or Bignum (as required) | int        | integer           | int       | i32         |
| int64      | int64_t     | long                | int/long[4]                      | int64   | Bignum                         | long       | integer/string[6] | Int64     | i64         |
| uint32     | uint32_t    | int[2]              | int/long[4]                      | uint32  | Fixnum or Bignum (as required) | uint       | integer           | int       | u32         |
| uint64     | uint64_t    | long[2]             | int/long[4]                      | uint64  | Bignum                         | ulong      | integer/string[6] | Int64     | u64         |
| sint32     | int32_t     | int                 | int                              | int32   | Fixnum or Bignum (as required) | int        | integer           | int       | i32         |
| sint64     | int64_t     | long                | int/long[4]                      | int64   | Bignum                         | long       | integer/string[6] | Int64     | i64         |
| fixed32    | uint32_t    | int[2]              | int/long[4]                      | uint32  | Fixnum or Bignum (as required) | uint       | integer           | int       | u32         |
| fixed64    | uint64_t    | long[2]             | int/long[4]                      | uint64  | Bignum                         | ulong      | integer/string[6] | Int64     | u64         |
| sfixed32   | int32_t     | int                 | int                              | int32   | Fixnum or Bignum (as required) | int        | integer           | int       | i32         |
| sfixed64   | int64_t     | long                | int/long[4]                      | int64   | Bignum                         | long       | integer/string[6] | Int64     | i64         |
| bool       | bool        | boolean             | bool                             | bool    | TrueClass/FalseClass           | bool       | boolean           | bool      | bool        |
| string     | std::string | String              | str/unicode[5]                   | string  | String (UTF-8)                 | string     | string            | String    | ProtoString |
| bytes      | std::string | ByteString          | str (Python 2), bytes (Python 3) | []byte  | String (ASCII-8BIT)            | ByteString | string            | List      | ProtoBytes  |

## 枚举类型

```proto
enum Corpus {
  CORPUS_UNSPECIFIED = 0;
  CORPUS_UNIVERSAL = 1;
  CORPUS_WEB = 2;
  CORPUS_IMAGES = 3;
  CORPUS_LOCAL = 4;
  CORPUS_NEWS = 5;
  CORPUS_PRODUCTS = 6;
  CORPUS_VIDEO = 7;
}
```

## Map类型
```proto
message Test6 {
  map<string, int32> g = 7;
}
```
map字段实际上就是下面的特殊重复字段的简写，它的实现和`repeat`一样
```proto
message Test6 {
  message g_Entry {
    optional string key = 1;
    optional int32 value = 2;
  }
  repeated g_Entry g = 7;
}
```
