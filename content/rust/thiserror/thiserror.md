---
title: "thiserror"
author: ["zhangjun"]
lastmod: 2024-06-08T21:57:42+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 5
---

[thiserror](https://google.github.io/comprehensive-rust/error-handling/thiserror-and-anyhow.html) 使用 derive macro 来创建自定义 Error 类型，消除实现 Display/Error/From 等标准 trait 的样板式代码。thiserror 可以使用 anyhow::Error. 而 anyhow 可以为自定义 Error 添加 context 信息。

```rust
use anyhow::{bail, Context, Result};
use std::fs;
use std::io::Read;
use thiserror::Error;

#[derive(Clone, Debug, Eq, Error, PartialEq)]
#[error("Found no username in {0}")]
struct EmptyUsernameError(String);

fn read_username(path: &str) -> Result<String> {
    let mut username = String::with_capacity(100);
    fs::File::open(path)
        .with_context(|| format!("Failed to open {path}"))?
        .read_to_string(&mut username)
        .context("Failed to read")?;
    if username.is_empty() {
        bail!(EmptyUsernameError(path.to_string()));
    }
    Ok(username)
}

fn main() {
    //fs::write("config.dat", "").unwrap();
    match read_username("config.dat") {
        Ok(username) => println!("Username: {username}"),
        Err(err) => println!("Error: {err:?}"),
    }
}

// https://github.com/ptillemans/rust-advent-of-code/blob/main/aoc-2018-1/src/main.rs
#[derive(thiserror::Error, Debug)]
pub enum AocError {
    #[error("Error parsing the input")]
    ParseError,
    #[error("No solution found")]
    NoSolutionFound,
}
```

<https://dev.to/seanchen1991/a-beginner-s-guide-to-handling-errors-in-rust-40k2>

常规的 Rust Error 定义方式：

1.  Result&lt;Config, &amp;'static str&gt; 的 Error 类型是字符串，缺点是需要解析内容才知道具体的错误类型和细节；
2.  Result&lt;(), Box&lt;dyn Error&gt;&gt; 是动态 Error，需要 downcase 来转换才知道错误类型；

<!--listend-->

```rust
impl Config {
    pub fn new(mut args: env::Args) -> Result<Config, &'static str> {
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

    pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
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

如果我们需要具体的错误类型，则一般需要通过自定义 Error 类型（如 enum）来实现：

```rust
// 在 src/error.rs 中几种定义错误类型
#[derive(Debug)]
pub enum AppError {
    MissingQuery,
    MissingFilename,
    ConfigLoad,
}
```

代码中返回错误类型：

```rust
- pub fn new(mut args: env::Args) -> Result<Config, &'static str>
+ pub fn new(mut args: env::Args) -> Result<Config, AppError>
    // --snip--

    let query = match args.next() {
        Some(arg) => arg,
-       None => return Err("Didn't get a query string"),
+       None => return Err(AppError::MissingQuery),
    };

    let filename = match args.next() {
        Some(arg) => arg,
-       None => return Err("Didn't get a file name"),
+       None => return Err(AppError::MissingFilename),
    };

    // --snip--


- pub fn run(config: Config) -> Result<(), Box<dyn Error>>
+ pub fn run(config: Config) -> Result<(), AppError>
```

但是 AppError 目前有些问题：

1.  缺少了具体的 error message；
2.  run() 中有可能返回 std::io:Error 并不能转换为 AppError，所以编译报错。

解决办法：使用 thiserror：

-   使用 #[error] 来为具体 variant 生成 Diplay message。
-   使用 #[from] 将其他类型 Error 作为 source 转换为 AppError。

<!--listend-->

```rust
+ use std::io;

+ use thiserror::Error;

- #[derive(Debug)]
+ #[derive(Debug, Error)]
pub enum AppError {
+   #[error("Didn't get a query string")]
    MissingQuery,
+   #[error("Didn't get a file name")]
    MissingFilename,
+   #[error("Could not load config")] // 提供 Display 实现，将显示 Error message
    ConfigLoad {
+       #[from]  // 提供将 io::Error 自动转换为 AppError 的实现
+       source: io::Error,
+   }
}
```

如果不使用 thiserror，完全手工来做：

```rust
use std::error::Error;
use std::fmt;

#[derive(Error)]
pub enum AppError {
    MissingQuery,
    MissingFilename,
    ConfigLoad {
        source: io::Error,
    }
}

// 为 AppError 实现错误信息展示的 Display，不同错误类型显示不同提示内容
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::MissingQuery => f.write_str("Didn't get a query string"),
            Self::MissingFilename => f.write_str("Didn't get a file name"),
            Self::ConfigLoad { source } => write!(f, "Could not load config: {}", source),
        }
    }
}

// 为 AppError 实现 std::error::Error trait
//
// 目的是提供 source of error，这里只是 ConfigLoad 会暴露更底层的 source 信息，其他类型没有和不需要
// 暴露source 信息。
impl error::Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            Self::ConfigLoad { source } => Some(source),
            _ => None,
        }
    }
}

// 将 std::io::Error 转换为 AppError
// 因为大量的标准库类型实现了 std::io::Error trait，所以实现该 From trait 后，它们都可以转换为 AppError
impl From<io::Error> for AppError {
    fn from(source: io::Error) -> Self {
        Self::ConfigLoad { source }
    }
}
```

最后使用 AppError：

```rust
// https://github.com/seanchen1991/error-handling-examples/blob/main/examples/minigrep/src/lib.rs
mod error;

use std::env;
use std::fs;
use error::AppError;

pub struct Config {
    query: String,
    filename: String,
    case_sensitive: bool,
}

impl Config {
    pub fn new(mut args: env::Args) -> Result<Config, AppError> {
        args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err(AppError::MissingQuery),
        };

        let filename = match args.next() {
            Some(arg) => arg,
            None => return Err(AppError::MissingFilename),
        };

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config {
            query,
            filename,
            case_sensitive,
        })
    }
}

pub fn run(config: Config) -> Result<(), AppError> {
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

另一个实际的例子：

```rust
// https://betterprogramming.pub/a-simple-guide-to-using-thiserror-crate-in-rust-eee6e442409b

use std::{error::Error, fmt::Debug};

enum CustomError {
    FileReadError(std::io::Error),
    RequestError(reqwest::Error),
    FileDeleteError(std::io::Error),
}

// 更好的方式是实现 From<std::io::Error>
impl From<reqwest::Error> for CustomError {
    fn from(e: reqwest::Error) -> Self {
        CustomError::RequestError(e)
    }
}

impl std::error::Error for CustomError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            CustomError::FileReadError(s) => Some(s),
            CustomError::RequestError(s) => Some(s),
            CustomError::FileDeleteError(s) => Some(s),
        }
    }
}

impl Debug for CustomError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        writeln!(f, "{}", self)?;
        if let Some(source) = self.source() {
            writeln!(f, "Caused by:\n\t{}", source)?;
        }
        Ok(())
    }
}

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

可见，需要给自定义 CustomError 定义许多样板式代码。

使用 thiserror 来重写上面的例子：

-   消除了实现 std::error::Error, std::fmt::Display 和 From&lt;reqwest::Error&gt;；
-   自定义错误消息可以引用错误信息（如字段值）；
-   可以使用 cargo-expand 来展开 derive 宏；

<!--listend-->

```rust
// https://betterprogramming.pub/a-simple-guide-to-using-thiserror-crate-in-rust-eee6e442409b
use std::{error::Error, fmt::Debug};

#[derive(thiserror::Error)]
enum CustomError {
    #[error("failed to read the key file")] // error attr 实现 Display trait
    FileReadError(#[source] std::io::Error),

    #[error("failed to send the api request")]
    RequestError(#[from] reqwest::Error),   // from 实现 impl From<reqwest::Error> for CustomeError

    #[error("failed to delete the key file")]
    FileDeleteError(#[source] std::io::Error), // source 实现 impl std::error::Error for CustomeError
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

// 宏被展开后

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

// 所有的 #[from] 都被实现为 From trait
#[allow(unused_qualifications)]
impl ::core::convert::From<reqwest::Error> for CustomError {
    #[allow(deprecated)]
    fn from(source: reqwest::Error) -> Self {
        CustomError::RequestError { 0: source }
    }
}


// 使用
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

注：

1.  任何 error type 实现 std::error::Error or dereferences to dyn std::error::Error will work as a
    source. 例如 anyhow::Error 类型可以作为 source。
2.  \#[error(transparent)] 表示将 forward the source and Display methods straight through to an
    underlying error without adding an additional message. ；
