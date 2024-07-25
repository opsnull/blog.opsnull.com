---
title: "serde_json"
author: ["zhangjun"]
lastmod: 2024-07-25T10:07:04+08:00
draft: false
series: ["rust crate"]
series_order: 17
---

serde_json crate 解析。

<!--more-->

<details>
<summary>变更历史</summary>
<div class="details">

2024-06-11 首次创建
</div>
</details>

serde_json 提供了 `serde_josn::Value` 类型, 可以用于反序列化无类型的 JSON 字符串, 后续可以对 Value 使用 map
index 操作来获得成员(类型是 &amp;Value):

-   可以使用 string 来获得 map 类型值, 可以使用整型 index 来获得数组成员;
-   如果 index 的 map key 或 array index 不存在, 则返回 `Value::Null`;
-   打印 Value 时, 返回 quota string. 如果不显示字符串 quota 双引号, 则可以调用 Value::as_str() 返回的字符串;

<!--listend-->

```rust
enum Value {
    Null,
    Bool(bool),
    Number(Number),
    String(String),
    Array(Vec<Value>),
    Object(Map<String, Value>),
}

use serde_json::{Result, Value};

fn untyped_example() -> Result<()> {
    // Some JSON input data as a &str. Maybe this comes from the user.
    let data = r#"
        {
            "name": "John Doe",
            "age": 43,
            "phones": [
                "+44 1234567",
                "+44 2345678"
            ]
        }"#;
    // Parse the string of data into serde_json::Value.
    let v: Value = serde_json::from_str(data)?;
    // Access parts of the data by indexing with square brackets.
    println!("Please call {} at the number {}", v["name"], v["phones"][0]); // 返回值类型是 &Value
    Ok(())
}
```

对于实现了 Serialize 和 Deserialize trait 的自定义类型, 可以序列化和反序列化对应类型的值:

```rust
use serde::{Deserialize, Serialize};
use serde_json::Result;

#[derive(Serialize, Deserialize)]
struct Person {
    name: String,
    age: u8,
    phones: Vec<String>,
}

fn typed_example() -> Result<()> {
    // Some JSON input data as a &str. Maybe this comes from the user.
    let data = r#"
        {
            "name": "John Doe",
            "age": 43,
            "phones": [
                "+44 1234567",
                "+44 2345678"
            ]
        }"#;

    // Parse the string of data into a Person object. This is exactly the same function as the one that
    // produced serde_json::Value above, but now we are asking it for a Person as output.
    let p: Person = serde_json::from_str(data)?;

    // Do things just like with any other Rust data structure.
    println!("Please call {} at the number {}", p.name, p.phones[0]);

    Ok(())
}
```

在对象实现了 Serialize trait 后, 可以使用 serde_json::to_string() 将类型对象转换为 JSON string :

```rust
use serde::{Deserialize, Serialize};
use serde_json::Result;

#[derive(Serialize, Deserialize)]
struct Address {
    street: String,
    city: String,
}

fn print_an_address() -> Result<()> {
    // Some data structure.
    let address = Address {
        street: "10 Downing Street".to_owned(),
        city: "London".to_owned(),
    };

    // Serialize it to a JSON string.
    let j = serde_json::to_string(&address)?;

    // Print, write to a file, or send to an HTTP server.
    println!("{}", j);

    Ok(())
}
```

json!() macro 从字符串创建 serde_json::Value 对象，它支持变量和表达式插值, 也可以调用其它宏 如 format!();

```rust
use serde_json::json;

fn main() {
    // The type of `john` is `serde_json::Value`
    let john = json!({
        "name": "John Doe",
        "age": 43,
        "phones": [
            "+44 1234567",
            "+44 2345678"
        ]
    });

    println!("first phone number: {}", john["phones"][0]);

    // Convert to a string of JSON and print it out
    println!("{}", john.to_string());
}


let full_name = "John Doe";
let age_last_year = 42;
// The type of `john` is `serde_json::Value`
let john = json!({
    "name": full_name,
    "age": age_last_year + 1,
    "phones": [
        format!("+44 {}", random_phone())
    ]
});
```

Deseralize 方法:

from_reader
: Deserialize an instance of type T from an I/O stream of JSON.

from_slice
: Deserialize an instance of type T from bytes of JSON text.

from_str
: Deserialize an instance of type T from a string of JSON text.

from_value
: Interpret a serde_json::Value as an instance of type T.

<!--listend-->

```rust
// 下面返回的 Result 是 serde_json::Result, 它的 Err 为 serde_json::Error 类型

// 从实现 std::io::Read trait 的 reader 读取
pub fn from_reader<R, T>(rdr: R) -> Result<T>
where
    R: Read,
    T: DeserializeOwned,

// 从 &[u8] 读取
pub fn from_slice<'a, T>(v: &'a [u8]) -> Result<T>
where
    T: Deserialize<'a>,

// 从 &str 读取
pub fn from_str<'a, T>(s: &'a str) -> Result<T>
where
    T: Deserialize<'a>,

// 从 serde_json::Value 读取
pub fn from_value<T>(value: Value) -> Result<T, Error>
where
    T: DeserializeOwned,


// 示例
use serde::Deserialize;
use std::error::Error;
use std::fs::File;
use std::io::BufReader;
use std::path::Path;

#[derive(Deserialize, Debug)]
struct User {
    fingerprint: String,
    location: String,
}
fn read_user_from_file<P: AsRef<Path>>(path: P) -> Result<User, Box<dyn Error>> {
    // Open the file in read-only mode with buffer.
    let file = File::open(path)?;
    let reader = BufReader::new(file);

    // Read the JSON contents of the file as an instance of `User`.
    let u = serde_json::from_reader(reader)?;

    // Return the `User`.
    Ok(u)
}
fn main() {
    let u = read_user_from_file("test.json").unwrap();
    println!("{:#?}", u);
}


#[derive(Deserialize, Debug)]
struct User {
    fingerprint: String,
    location: String,
}
fn main() {
    // The type of `j` is `&[u8]`
    let j = b"
        {
            \"fingerprint\": \"0xF9BA143B95FF6D82\",
            \"location\": \"Menlo Park, CA\"
        }";

    let u: User = serde_json::from_slice(j).unwrap();
    println!("{:#?}", u);
}


#[derive(Deserialize, Debug)]
struct User {
    fingerprint: String,
    location: String,
}

fn main() {
    // The type of `j` is `&str`
    let j = "
        {
            \"fingerprint\": \"0xF9BA143B95FF6D82\",
            \"location\": \"Menlo Park, CA\"
        }";

    let u: User = serde_json::from_str(j).unwrap();
    println!("{:#?}", u);
}


use serde::Deserialize;
use serde_json::json; // 导入 json!() 宏
#[derive(Deserialize, Debug)]
struct User {
    fingerprint: String,
    location: String,
}
fn main() {
    // The type of `j` is `serde_json::Value`
    let j = json!({
        "fingerprint": "0xF9BA143B95FF6D82",
        "location": "Menlo Park, CA"
    });

    let u: User = serde_json::from_value(j).unwrap();
    println!("{:#?}", u);
}
```

Serialize 方法:

to_string
: Serialize the given data structure as a String of JSON.

to_string_pretty
: Serialize the given data structure as a pretty-printed String of JSON.

to_value
: Convert a T into serde_json::Value which is an enum that can represent any valid JSON data.

to_vec
: Serialize the given data structure as a JSON byte vector.

to_vec_pretty
: Serialize the given data structure as a pretty-printed JSON byte vector.

to_writer
: Serialize the given data structure as JSON into the I/O stream.

to_writer_pretty
: Serialize the given data structure as pretty-printed JSON into the I/O

<!--listend-->

```rust
// 下面返回的 Result 是 serde_json::Result, 它的 Err 为 serde_json::Error 类型

pub fn to_string<T>(value: &T) -> Result<String> where T: ?Sized + Serialize,
// 美化输出
pub fn to_string_pretty<T>(value: &T) -> Result<String> where T: ?Sized + Serialize,

pub fn to_value<T>(value: T) -> Result<Value, Error> where T: Serialize,

// 将对象序列化为 Vec<u8>
pub fn to_vec<T>(value: &T) -> Result<Vec<u8>> where T: ?Sized + Serialize,
pub fn to_vec_pretty<T>(value: &T) -> Result<Vec<u8>> where T: ?Sized + Serialize,

// 将对象序列化为 UTF-8 字节流, 然后写入 writer
pub fn to_writer<W, T>(writer: W, value: &T) -> Result<()> where W: Write, T: ?Sized + Serialize,
pub fn to_writer_pretty<W, T>(writer: W, value: &T) -> Result<()> where W: Write, T: ?Sized + Serialize,

// 示例
use serde::Serialize;
use serde_json::json;
use std::error::Error;

#[derive(Serialize)]
struct User {
    fingerprint: String,
    location: String,
}

fn compare_json_values() -> Result<(), Box<dyn Error>> {
    let u = User {
        fingerprint: "0xF9BA143B95FF6D82".to_owned(),
        location: "Menlo Park, CA".to_owned(),
    };

    // The type of `expected` is `serde_json::Value`
    let expected = json!({
        "fingerprint": "0xF9BA143B95FF6D82",
        "location": "Menlo Park, CA",
    });

    let v = serde_json::to_value(u).unwrap();
    assert_eq!(v, expected);

    Ok(())
}
```

serde_json 提供了 `Error` 和 `Result` 类型, 用于在 Serialize 或 Deserialize 出错时返回:

```rust
use serde_json::Value;
use std::io::{self, ErrorKind, Read};
use std::process;

struct ReaderThatWillTimeOut<'a>(&'a [u8]);

impl<'a> Read for ReaderThatWillTimeOut<'a> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        if self.0.is_empty() {
            Err(io::Error::new(ErrorKind::TimedOut, "timed out"))
        } else {
            self.0.read(buf)
        }
    }
}

fn main() {
    let reader = ReaderThatWillTimeOut(br#" {"k": "#);

    let _: Value = match serde_json::from_reader(reader) {
        Ok(value) => value,
        Err(error) => {
            if error.io_error_kind() == Some(ErrorKind::TimedOut) {
                // Maybe this application needs to retry certain kinds of errors.

            } else {
                eprintln!("error: {}", error);
                process::exit(1);
            }
        }
    };
}
```
