---
title: "thiserror"
author: ["zhangjun"]
lastmod: 2024-08-04T17:09:16+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 5
---

thiserror crate 解析.

<!--more-->

<details>
<summary>变更历史</summary>
<div class="details">

2024-06-10 Mon  review 重写.

2024-05-24 Fri  首次创建
</div>
</details>

[thiserror](https://google.github.io/comprehensive-rust/error-handling/thiserror-and-anyhow.html) 使用 derive macro 来创建自定义 Error 类型，消除实现 Display/Error/From 等标准 trait 的样板式代码。

常规的 Rust Error 定义方式：

1.  Result&lt;Config, &amp;'static str&gt; 的 Error 类型是字符串，缺点是需要解析内容才知道具体的错误类型和细节；
2.  Result&lt;(), Box&lt;dyn std::error::Error&gt;&gt; 是动态 Error，需要 downcase 来转换才知道错误类型；
    -   好处是：基本上标准库所有 Error 类型都可以自动转换为 std::error::Error；

<!--listend-->

```rust
impl Config {
    // 使用 &'static str
    pub fn new(mut args: std::env::Args) -> Result<Config, &'static str> {
        args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };

        let filename = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file name"),
        };

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config {
            query,
            filename,
            case_sensitive,
        })
    }

    // 使用 Box<dyn std::error::Error>
    pub fn run(config: Config) -> Result<(), Box<dyn std::error::Error>> {
        let contents = fs::read_to_string(config.filename)?;

        let results = if config.case_sensitive {
            search(&config.query, &contents)
        } else {
            search_case_insensitive(&config.query, &contents)
        };

        for line in results {
            println!("{}", line);
        }

        Ok(())
    }
```

标准库 std:error::Error:

1.  没有必须要实现的方法，故大部分情况下可以作为 marker trait 来实现;
2.  标准库和三方库的各种自定义错误类型都需要实现的 trait，这样后续可以使用 Box&lt;dyn std::error::Error&gt;
    来统一处理;
3.  source() 方法返回更低一个层次的 Errror, 这样后续在显示 Error string 时,会提示 caused by 之类的信息;

<!--listend-->

```rust
pub trait Error: Debug + Display {
    // Provided methods
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }
    fn description(&self) -> &str { ... }
    fn cause(&self) -> Option<&dyn Error> { ... }
    fn provide<'a>(&'a self, request: &mut Request<'a>) { ... }
}
```

自定义错误类型例子：包含大量样板式代码实现:

```rust
// https://betterprogramming.pub/a-simple-guide-to-using-thiserror-crate-in-rust-eee6e442409b

use std::{error::Error, fmt::Debug};

// 自定义错误类型, 封装了底层的其它错误类型( std::io::Error 都是具体 struct 类型而非 trait)
enum CustomError {
    FileReadError(std::io::Error),
    RequestError(reqwest::Error),
    FileDeleteError(std::io::Error),
}

// 将其它 Error 类型转换为 CustomError 类型
//
// 更好的方式是实现 From<std::io::Error>
impl From<reqwest::Error> for CustomError {
    fn from(e: reqwest::Error) -> Self {
        CustomError::RequestError(e)
    }
}

// 实现 std::error::Error, 同时自定义 source() 方法, 返回更底层的错误对象
//
// 因为大量的标准库类型实现了 std::io::Error trait，所以实现该 From trait 后，它们都可以转换为
// CustomError
impl std::error::Error for CustomError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            CustomError::FileReadError(s) => Some(s),
            CustomError::RequestError(s) => Some(s),
            CustomError::FileDeleteError(s) => Some(s),
        }
    }
}

// 由于 std::error::Error 是 Debug 和 Display 的 子 trait, 所以自定义错误类型还需要实现这两个 trait
impl Debug for CustomError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        writeln!(f, "{}", self)?;
        if let Some(source) = self.source() {
            writeln!(f, "Caused by:\n\t{}", source)?;
        }
        Ok(())
    }
}

// 根据错误类型的 variant, 显示不同的错误信息
impl std::fmt::Display for CustomError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            CustomError::FileReadError(_) => write!(f, "failed to read the key file"),
            CustomError::RequestError(_) => write!(f, "failed to send the api request"),
            CustomError::FileDeleteError(_) => write!(f, "failed to delete the key file"),
        }
    }
}

// 使用自定义 CustomError
fn main() {
    println!("{:?}", make_request().unwrap_err());
}

fn make_request() -> Result<(), CustomError> {
    use CustomError::*;

    // 使用 map_err 将其它错误转换为自定义错误
    let key = std::fs::read_to_string("some-key-file").map_err(FileReadError)?;
    reqwest::blocking::get(format!("http:key/{}", key))?.error_for_status()?;
    std::fs::remove_file("some-key-file").map_err(FileDeleteError)?;
    Ok(())
}
```

使用 `thiserror crate` 来重写上面的例子, 可以大大简化自定义错误类型的样板式代码:

-   消除了实现 impl std::error::Error, std::fmt::Display 和 From&lt;reqwest::Error&gt;；
-   自定义错误消息可以引用错误信息（如字段值）；
-   可以使用 cargo-expand 来展开 derive 宏；
-   \#[source] 和 #[from] 的区别：source 是类型在实现 std::error::Error 的 source() 方法，用于指定更深层次的错误类型，后续需要用于调用该方法来获取深层次的报错信息。from 是生成从其他错误类型转换为自定义错误类型的 From trait；

<!--listend-->

```rust
// https://betterprogramming.pub/a-simple-guide-to-using-thiserror-crate-in-rust-eee6e442409b
use std::{error::Error, fmt::Debug};

// Error 实现 std::error::Error trait
#[derive(thiserror::Error)]
enum CustomError {
    #[error("failed to read the key file")] // #[error] 实现 Display trait
    FileReadError(#[source] std::io::Error), // #[source] 实现 impl std::error::Error for CustomeError, 自定义 source() 方法。

    #[error("failed to send the api request")]
    RequestError(#[from] reqwest::Error),   // #[from] 实现 impl From<reqwest::Error> for CustomeError

    #[error("failed to delete the key file")]
    FileDeleteError(#[source] std::io::Error),
}

impl Debug for CustomError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        writeln!(f, "{}", self)?;
        if let Some(source) = self.source() { // 使用 #[source]
            writeln!(f, "Caused by:\n\t{}", source)?;
        }
        Ok(())
    }
}

// 使用 cargo-expand 来参开宏

// 所有的 #[source] 都被实现为 std::error::Error 的 source() 方法
#[allow(unused_qualifications)]
impl std::error::Error for CustomError {
    fn source(&self) -> ::core::option::Option<&(dyn std::error::Error + 'static)> {
        use thiserror::__private::AsDynError as _;
        #[allow(deprecated)]
        match self {
            CustomError::FileReadError { 0: source, .. } => {
                ::core::option::Option::Some(source.as_dyn_error())
            }
            CustomError::RequestError { 0: source, .. } => {
                ::core::option::Option::Some(source.as_dyn_error())
            }
            CustomError::FileDeleteError { 0: source, .. } => {
                ::core::option::Option::Some(source.as_dyn_error())
            }
        }
    }
}

// 所有的 #[error] 都被实现为 Display trait(内部会进行 variant 判断匹配)
#[allow(unused_qualifications)]
impl ::core::fmt::Display for CustomError {
    fn fmt(&self, __formatter: &mut ::core::fmt::Formatter) -> ::core::fmt::Result {
        #[allow(unused_variables, deprecated, clippy::used_underscore_binding)]
        match self {
            CustomError::FileReadError(_0) => __formatter.write_str("failed to read the key file"),
            CustomError::RequestError(_0) => {
                __formatter.write_str("failed to send the api request")
            }
            CustomError::FileDeleteError(_0) => {
                __formatter.write_str("failed to delete the key file")
            }
        }
    }
}

// 所有的 #[from] 都被实现为 From trait, 所以在一个错误类型中, 不能对同一个类型多次添加 #[from] macro:
#[allow(unused_qualifications)]
impl ::core::convert::From<reqwest::Error> for CustomError {
    #[allow(deprecated)]
    fn from(source: reqwest::Error) -> Self {
        CustomError::RequestError { 0: source }
    }
}


// 使用 CustomError
fn main() {
    println!("{:?}", make_request().unwrap_err());
}

fn make_request() -> Result<(), CustomError> {
    use CustomError::*;
    let key = std::fs::read_to_string("some-key-file").map_err(FileReadError)?;
    reqwest::blocking::get(format!("http:key/{}", key))?.error_for_status()?;
    std::fs::remove_file("some-key-file").map_err(FileDeleteError)?;
    Ok(())
}
```

thiserror 也可以用于 struct 类型

```rust
// thiserror::Error 也可以用于 struct
#[derive(thiserror::Error, Debug)]
#[error("error type1 {i} {b} {s}")]
struct SimpleError2 {
    // struct filed 不能使用 #[error]
    i: i32,
    b: bool,
    s: String,
}
```

注：

1.  任何实现了 `std::error::Error` or dereferences to `dyn std::error::Error` 都可以作为 #[source], 例如
    `anyhow::Error` 类型可以作为 source;
2.  对同一种错误类型，#[source] 可以指定多次，但是对于 #[from] 只能指定一次。
3.  `#[error(transparent)]` 表示将 forward the source and Display methods straight through to an
    underlying error without adding an additional message；

<!--listend-->

```rust
#[derive(thiserror::Error, Debug)]
enum SimpleError {
    // #[error(transparent)] 是不指定 error message，但显示对应 error 类型的 Display message。
    #[error(transparent)]
    //ErrorOther(anyhow::Error, i32), // rustc: error: #[error(transparent)] requires exactly one field
    ErrorOther(anyhow::Error)
}

// thiserror::Error 也可以用于 struct 错误类型
#[derive(thiserror::Error, Debug)]
#[error("struct: {i} {b} {s}")]
struct SimpleError2 {
    // struct filed 不能使用 #[error]
    i: i32,
    b: bool,
    s: String,
    // #[from] o: anyhow::Error, // rustc: error: deriving From requires no fields other than source and backtrace
    // #[from] source: anyhow::Error, // rustc: error: deriving From requires no fields other than source and backtrace
}

// struct 错误类型的 field 如果使用 #[from], 则只能包含两个固定的 field name: source 和 backtrace.
#[derive(thiserror::Error, Debug)]
#[error("struct2: {source}")]
struct SimpleError3 {
    #[from]
    source: anyhow::Error,
    // i: i32, // rustc: error: deriving From requires no fields other than source and backtrace
}

// 或者使用 newtype 类型
#[derive(thiserror::Error, Debug)]
#[error("struct3: {0}")]
struct SimpleError4(#[from] anyhow::Error);
```
