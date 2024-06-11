---
title: "serde"
author: ["zhangjun"]
lastmod: 2024-06-11T20:27:33+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 2
---

[serde crate](https://serde.rs/) 包含两层:

1.  data structures: 实现了 serialize 和 deserialize trait 的内置或自定义类型, 可以调用 serde crate
    来实现;
2.  data format: data structures 保存的文件格式, 如 json/toml/yaml/csv/url 等。任意实现了 data
    structures 的 serialize 和 deserialize trait 的类型都可以转换为 `任意` 支持的 data format;
    -   data format 在各种单独的 serde_XX crate 中提供, 如 serde_json/serde_yaml 等;

serde 为 Rust 内置 29 种类型[都提供了 data structure 实现](https://serde.rs/data-model.html#types), 所以一般只需要为自定义类型使用 #[derive]
宏来自动生成 ser/deser 的代码即可:

Cargo.toml:

```toml
[package]
name = "my-crate"
version = "0.1.0"
authors = ["Me <user@rust-lang.org>"]

[dependencies]
serde = { version = "1.0", features = ["derive"] } # 需要开启 derive 或 full feature
serde_json = "1.0"
```

src/main.rs:

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let point = Point { x: 1, y: 2 };

    // Convert the Point to a JSON string.
    let serialized = serde_json::to_string(&point).unwrap();

    // Prints serialized = {"x":1,"y":2}
    println!("serialized = {}", serialized);

    // Convert the JSON string back to a Point.
    let deserialized: Point = serde_json::from_str(&serialized).unwrap();

    // Prints deserialized = Point { x: 1, y: 2 }
    println!("deserialized = {:?}", deserialized);
}
```

使用三种 Attributes 对 Serialize and Deserialize 进行更灵活的配置：

-   Container attributes — 对 struct/enum 类型整体
-   Variant attributes —  只对 enum variant 成员
-   Field attributes —  对 struct 或 enum variant 成员

<!--listend-->

```rust
#[derive(Serialize, Deserialize)]
#[serde(deny_unknown_fields)]  // <-- this is a container attribute
struct S {
    #[serde(default)]  // <-- this is a field attribute
    f: i32,
}

#[derive(Serialize, Deserialize)]
#[serde(rename = "e")]  // <-- this is also a container attribute
enum E {
    #[serde(rename = "a")]  // <-- this is a variant attribute
    A(String),
}
```

Container attributes（对 struct/map/set/enum 等整体进行重命名）：

-   \#[serde(rename = "name")]
-   \#[serde(rename(serialize = "ser_name"))]
-   \#[serde(rename(deserialize = "de_name"))]
-   \#[serde(rename(serialize = "ser_name", deserialize = "de_name"))]
-   \#[serde(rename_all = "...")]
-   \#[serde(rename_all_fields = "...")]
    -   "lowercase", "UPPERCASE", "PascalCase", "camelCase", "snake_case",
        "SCREAMING_SNAKE_CASE","kebab-case", "SCREAMING-KEBAB-CASE".
-   \#[serde(rename_all(serialize = "..."))]
-   \#[serde(rename_all(deserialize = "..."))]
-   \#[serde(rename_all(serialize = "...", deserialize = "..."))]

-   \#[serde(deny_unknown_fields)]: 在 deserialization 时如果遇到 unknown filed 则报错.(默认是忽略该 field).

<!--listend-->

-   \#[serde(tag = "type")]             # {"type": "Request", "id": "...", "method": "...", "params": {...}}
-   \#[serde(tag = "t", content = "c")] # {"t": "Para", "c": [{...}, {...}]}
-   \#[serde(untagged)] # {"id": "...", "method": "...", "params": {...}}， 不添加 enum variant name

<!--listend-->

-   \#[serde(default)]

<!--listend-->

-   \#[serde(default = "path")]

-   \#[serde(transparent)]
-   \#[serde(from = "FromType")]
-   \#[serde(try_from = "FromType")]
-   \#[serde(into = "IntoType")]
-   \#[serde(crate = "...")]
-   \#[serde(expecting = "...")]

Variant attributes

-   \#[serde(rename = "name")]
-   \#[serde(rename(serialize = "ser_name"))]
-   \#[serde(rename(deserialize = "de_name"))]
-   \#[serde(rename(serialize = "ser_name", deserialize = "de_name"))]

<!--listend-->

-   \#[serde(alias = "name")] # 可以指定多个别名
-   \#[serde(rename_all = "...")]
    -   "lowercase", "UPPERCASE", "PascalCase", "camelCase", "snake_case", "SCREAMING_SNAKE_CASE",
        "kebab-case", "SCREAMING-KEBAB-CASE".

-   \#[serde(skip)]
-   \#[serde(skip_serializing)]
-   \#[serde(skip_deserializing)]

<!--listend-->

-   \#[serde(serialize_with = "path")]
-   \#[serde(deserialize_with = "path")]

-   \#[serde(with = "module")]
-   \#[serde(bound = "T: MyTrait")]
-   \#[serde(borrow)] and #[serde(borrow = "'a + 'b + ...")]
-   \#[serde(other)]
-   \#[serde(untagged)]

Field attributes

-   \#[serde(rename = "name")]
-   \#[serde(rename(serialize = "ser_name"))]
-   \#[serde(rename(deserialize = "de_name"))]
-   \#[serde(rename(serialize = "ser_name", deserialize = "de_name"))]
-   \#[serde(alias = "name")]
-   \#[serde(default)]
-   \#[serde(default = "path")]
-   \#[serde(flatten)]
-   \#[serde(skip)]
-   \#[serde(skip_serializing)]
-   \#[serde(skip_deserializing)]
-   \#[serde(skip_serializing_if = "path")]
-   \#[serde(serialize_with = "path")]
-   \#[serde(deserialize_with = "path")]
-   \#[serde(with = "module")] # `Combination of serialize_with and deserialize_with`. Serde will use
    $module::serialize as the serialize_with function and $module::deserialize as the deserialize_with
    function.

自定义序列化: Serializer/Deserializer 是 serde 的 data format package 提供的类型，如
serde_json/serde_yaml 等；

```rust
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer;
}

pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```


## <span class="section-num">1</span> 序列化 {#序列化}

自定义对象需要实现 Serialize trait，才能被序列化：

-   一般通过 #[derive(Serialize)] 宏来实现;

<!--listend-->

```rust
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer;
}
```

核心是调用 serializer 提供的序列化方法，该 serializer 是后续调用 serde data format 如
serde_json/serde_yaml 时传入的：

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let point = Point { x: 1, y: 2 };

    let serialized = serde_json::to_string(&point).unwrap(); // 调用 point 实现的 serialize() 方法
    println!("serialized = {}", serialized);

    let deserialized: Point = serde_json::from_str(&serialized).unwrap();
    println!("deserialized = {:?}", deserialized);
}
```

序列化 Rust 原始类型, 以 i32 为例：

-   serde 为 Rust 内置 29 种类型[都提供了 data structure 实现](https://serde.rs/data-model.html#types), 所以一般只需要为自定义类型使用 #[derive]
    宏来自动生成 ser/deser 的代码即可:

<!--listend-->

```rust
impl Serialize for i32 {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_i32(*self) // 根据 Self 的类型，调用 serialize_XX 的方法。
    }
}
```

序列化 seq/map 类型：

```rust
use serde::ser::{Serialize, Serializer, SerializeSeq, SerializeMap};

impl<T> Serialize for Vec<T>  // T 必须也实现了 Serialize
where
    T: Serialize,
{
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut seq = serializer.serialize_seq(Some(self.len()))?; // 序列化 顺序 类型，如 Vec/HashSet
        for e in self {
            seq.serialize_element(e)?;
        }
        seq.end()
    }
}

impl<K, V> Serialize for MyMap<K, V>
where
    K: Serialize,
    V: Serialize,
{
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut map = serializer.serialize_map(Some(self.len()))?; // 序列化 map 类型
        for (k, v) in self {
            map.serialize_entry(k, v)?;
        }
        map.end()
    }
}
```

序列化 struct：

```rust
use serde::ser::{Serialize, Serializer, SerializeStruct};

struct Color {
    r: u8,
    g: u8,
    b: u8,
}

impl Serialize for Color {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        // 3 is the number of fields in the struct.
        let mut state = serializer.serialize_struct("Color", 3)?;
        state.serialize_field("r", &self.r)?; // 不需要指定 field 类型
        state.serialize_field("g", &self.g)?;
        state.serialize_field("b", &self.b)?;
        state.end()
    }
```

序列化 enum：

```rust
enum E {
    // Use three-step process:
    //   1. serialize_struct_variant
    //   2. serialize_field
    //   3. end
    Color { r: u8, g: u8, b: u8 },

    // Use three-step process:
    //   1. serialize_tuple_variant
    //   2. serialize_field
    //   3. end
    Point2D(f64, f64),

    // Use serialize_newtype_variant.
    Inches(u64),

    // Use serialize_unit_variant.
    Instance,
}
```


## <span class="section-num">2</span> 反序列化 {#反序列化}

类型需要实现 Deserialize trait 来进行反序列化:

-   一般通过 #[derive(Deserialize)] 宏来实现该 trait.

<!--listend-->

```rust
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```

核心是自定义类型要实现 Deserialize trait，而该 trait 的 deserializer 需要调用某一个 deserialize_XX()
 方法， 该方法的参数是一个自定义的 Visitor 对象：

-   Visitor 的关联类型为反序列化生成的对象类型；
-   Visitor 对象需要实现 expecting 和一系列 visit_XX() 方法，后续由 deserializer 调用；
-   deserialize_XX() 不一定调用 Visitor 的 visit_XX() 方法，具体取决于 deserializer 的实现；

例如：先为 i32 定义一个实现 Visitor trait 的对象, 该对象的关联类型 Value 与最终要解码生成的对象类型一致（这里是 i32）；

```rust
use std::fmt;

use serde::de::{self, Visitor};

struct I32Visitor;

impl<'de> Visitor<'de> for I32Visitor {
    type Value = i32; // 要 deserilize 的类型

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result { // 异常时显示的字符串。
        formatter.write_str("an integer between -2^31 and 2^31")
    }

    fn visit_i8<E>(self, value: i8) -> Result<Self::Value, E>
    where
        E: de::Error,
    {
        Ok(i32::from(value))
    }

    fn visit_i32<E>(self, value: i32) -> Result<Self::Value, E>
    where
        E: de::Error,
    {
        Ok(value)
    }

    fn visit_i64<E>(self, value: i64) -> Result<Self::Value, E>
    where
        E: de::Error,
    {
        use std::i32;
        if value >= i64::from(i32::MIN) && value <= i64::from(i32::MAX) {
            Ok(value as i32)
        } else {
            Err(E::custom(format!("i32 out of range: {}", value)))
        }
    }

    // Similar for other methods:
    //   - visit_i16
    //   - visit_u8
    //   - visit_u16
    //   - visit_u32
    //   - visit_u64
}
```

1.  为 i32 类型定义 Deserialize 实现, Deserializer 是 serde_json/yaml 等提供的类型：

<!--listend-->

```rust
impl<'de> Deserialize<'de> for i32 {
    fn deserialize<D>(deserializer: D) -> Result<i32, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_i32(I32Visitor) // 传入为 i32 定义的 Visitor
    }
}
```
