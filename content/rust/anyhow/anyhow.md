---
title: "anyhow"
author: ["zhangjun"]
lastmod: 2024-06-11T20:27:36+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 6
---

anyhow crate 包解析.

<!--more-->

<details>
<summary>变更历史</summary>
<div class="details">

2024-06-10 Mon 重新 review 和更新内容.

2024-06-08 Fri 首次创建
</div>
</details>

sturct [anyhow::Error](https://docs.rs/anyhow/latest/anyhow/) 是各种动态 error 类型的封装, 类似与 `Box<dyn std::error::Error>`, 具有如下特点:

1.  anyhow::Error 实现了 Send/Sync/'static, 可用于多线程环境中;
2.  anyhow::Error 确保提供了 backtrace 的支持(需要运行时配置 RUST_BACKTRACE=1 环境变量);
3.  anyhow::Error 只占用一个 usize 大小, 而 Box trait object 占用两个 usize;

提供了三个抽象:

1.  struct anyhow::Error
2.  struct anyhow::Result&lt;T&gt; , 等效于 std::result::Result&lt;T, anyhow::Error&gt;;
3.  trait anyhow::Context, 自动为 Result/Options 实现了该 trait;

`andyhow::Error` 可以转换为 Box&lt;dyn StdError&gt;:

-   From&lt;Error&gt; for Box&lt;dyn StdError + 'static&gt;
-   From&lt;Error&gt; for Box&lt;dyn StdError + Send + 'static&gt;
-   From&lt;Error&gt; for Box&lt;dyn StdError + Send + Sync + 'static&gt;:

`anyhow::Error` 可以作为 std::error::Error 来使用:

-   Deref&lt;Target = dyn std::error::Error + Sync + Send&gt;

任何实现了 std::error::Error 的错误类型 `都可以被转换为 anyhow::Error`:

-   From&lt;E&gt; where E: StdError + Send + Sync + 'static

anyhow::Error 方法：

-   new(error): 从标准库 error 创建 anyhow::Error
-   msg(message): 从 message 创建 anyhow::Error
-   context(context): 为 error 添加 context 信息, 后续在打印错误时先打印 context 信息, 再是底层 error
    信息;

<!--listend-->

```rust
impl Error

// new 将 std Error 的错误转换为 anyhow::Error
pub fn new<E>(error: E) -> Self where E: StdError + Send + Sync + 'static, // StdError 为标准库 core::error::Error

// Create a new error object from a printable error message.
pub fn msg<M>(message: M) -> Self where M: Display + Debug + Send + Sync + 'static,
use anyhow::{Error, Result};
use futures::stream::{Stream, StreamExt, TryStreamExt};
async fn demo<S>(stream: S) -> Result<Vec<Output>> where S: Stream<Item = Input>,
 {
        stream
            .then(ffi::do_some_work) // returns Result<Output, &str>
            .map_err(Error::msg)
            .try_collect()
            .await
 }


// Wrap the error value with additional context.
pub fn context<C>(self, context: C) -> Self where C: Display + Send + Sync + 'static,
use anyhow::Result;
use std::fs::File;
use std::path::Path;
struct ParseError {
    line: usize,
    column: usize,
}
fn parse_impl(file: File) -> Result<T, ParseError> {
    //...
}
pub fn parse(path: impl AsRef<Path>) -> Result<T> {
    let file = File::open(&path)?;
    parse_impl(file).map_err(|error| {
        let context = format!(
            "only the first {} lines of {} are valid",
            error.line, path.as_ref().display(),
        );
        anyhow::Error::new(error).context(context)
    })
}

// Get the backtrace for this Error.
pub fn backtrace(&self) -> &Backtrace

// An iterator of the chain of source errors contained by this Error.  This iterator will visit
// every error in the cause chain of this error object, beginning with the error that this error
// object was created from.
pub fn chain(&self) -> Chain<'_>

use anyhow::Error;
use std::io;
pub fn underlying_io_error_kind(error: &Error) -> Option<io::ErrorKind> {
    for cause in error.chain() {
        if let Some(io_error) = cause.downcast_ref::<io::Error>() {
            return Some(io_error.kind());
        }
    }
    None
}


pub fn root_cause(&self) -> &(dyn StdError + 'static)

// Returns true if E is the type held by this error object.
pub fn is<E>(&self) -> bool where E: Display + Debug + Send + Sync + 'static,

// Attempt to downcast the error object to a concrete type.
pub fn downcast<E>(self) -> Result<E, Self> where E: Display + Debug + Send + Sync + 'static,

// Downcast this error object by reference.
pub fn downcast_ref<E>(&self) -> Option<&E> where E: Display + Debug + Send + Sync + 'static,
    // If the error was caused by redaction, then return a tombstone instead
    // of the content.
    match root_cause.downcast_ref::<DataStoreError>() {
        Some(DataStoreError::Censored(_)) => Ok(Poll::Ready(REDACTED_CONTENT)),
        None => Err(error),
}
```

anyhow::Error 自定义了 std::fmt 的 trait，通过 `{}/{:#}/{:?}/{:#?}` 来打印不同的错误信息：

1.  `{}` :只显示最外层的错误，如 Failed to read instrs from ./path/to/instrs.json
2.  `{:#}`: 显示所有错误，如 Failed to read instrs from ./path/to/instrs.json: No such file or
    directory (os error 2)
3.  `{:?}` 显示 backtrace
    ```rust

       Error: Failed to read instrs from ./path/to/instrs.json

       Caused by:
           No such file or directory (os error 2)

           // and if there is a backtrace available:

           Error: Failed to read instrs from ./path/to/instrs.json

       Caused by:
           No such file or directory (os error 2)

       Stack backtrace:
          0: <E as anyhow::context::ext::StdError>::ext_context
              at /git/anyhow/src/backtrace.rs:26
          1: core::result::Result<T,E>::map_err
              at /git/rustc/src/libcore/result.rs:596
          2: anyhow::context::<impl anyhow::Context<T,E> for core::result::Result<T,E>>::with_context
                    at /git/anyhow/src/context.rs:58
          3: testing::main
                    at src/main.rs:5
          4: std::rt::lang_start
              at /git/rustc/src/libstd/rt.rs:61
          5: main
          6: __libc_start_main
          7: _start
    ```
4.  `{:#?}`: 使用 struct 风格的错误表示：
    ```rust
       Error {
           context: "Failed to read instrs from ./path/to/instrs.json",
           source: Os {
               code: 2,
               kind: NotFound,
               message: "No such file or directory",
           },
       }
    ```

Rust &gt;= 1.65  版本支持捕获 backtrace, 需要启用 `std::backtrace` 中定义的如下环境变量:

1.  panic 和 error 都包含 backtrace: `RUST_BACKTRACE=1`
2.  只对 error 捕获 backtrace: `RUST_LIB_BACKTRACE=1`;
3.  只对 panic 捕获 backtrace: `RUST_BACKTRACE=1 and RUST_LIB_BACKTRACE=0`.

`anyhow::Result<T>` 是 `std::Result<T, anyhow::Error>` 类型别名，它使用的 anyhow::Error 在显示出错信息时包含 context 和 backtrace:

```rust
use anyhow::{Context, Result};

// 使用 anyhow::Result<()> 作为函数返回值, 它使用 anyhow::Error
fn main() -> Result<()> {
    //...
    it.detach().context("Failed to detach the important thing")?;

    let content = std::fs::read(path).with_context(|| format!("Failed to read instrs from {}", path))?;
    //...

    // 其它实现标准库 Error 的类型会被自动转换为 anyhow::Error
    let config = std::fs::read_to_string("cluster.json")?;
    let map: ClusterMap = serde_json::from_str(&config)?;
    println!("cluster info: {:#?}", map);
    Ok(())
}
```

`Trait anyhow::Context` 为 Result/Option 提供了 context 方法:

-   Result/Option 都实现了 anyhow::Context trait;
-   任何实现了 std::error::Error 的 error 类型都添加 Context() 方法, 所以 thiserror 宏定义的自定义
    error 类型都可以和 anyhow::Context 使用;
-   后续报错时, 会先返回 context()/with_context() 的上下文信息, 然后是 Caused by 开始的未加 context 前的错误.

<!--listend-->

```rust
pub trait Context<T, E>: Sealed {
    // Required methods
    fn context<C>(self, context: C) -> Result<T, Error> where C: Display + Send + Sync + 'static;
    fn with_context<C, F>(self, f: F) -> Result<T, Error> where C: Display + Send + Sync + 'static, F: FnOnce() -> C;
}

impl<T> Context<T, Infallible> for Option<T>
// 任何实现了 std::error::Error 的 error 类型都添加 Context() 方法, 所以 thiserror 宏定义的自定义
// error 类型,都可以和 anyhow::Context 使用;
impl<T, E> Context<T, E> for Result<T, E> where E: StdError + Send + Sync + 'static,

// 示例:
use anyhow::{Context, Result};
use std::fs;
use std::path::PathBuf;
pub struct ImportantThing {
    path: PathBuf,
}
impl ImportantThing {
    pub fn detach(&mut self) -> Result<()> {...}
}
pub fn do_it(mut it: ImportantThing) -> Result<Vec<u8>> {
    it.detach().context("Failed to detach the important thing")?; // 为 Result 自动添加 context() 方法
    let path = &it.path;
    let content = fs::read(path) // 为 Result 自动添加 with_context() 方法
        .with_context(|| format!("Failed to read instrs from {}", path.display()))?;
    Ok(content)
}

// 这样后续报错时, 会先返回 context()/with_context() 的内容, 然后是 Caused by 开始的未加 context 前的错误.

// Error: Failed to read instrs from ./path/to/instrs.json  // context() 信息

// Caused by:
//     No such file or directory (os error 2) // fs::read() 的报错信息
```

struct anyhow::Error 实现了 std::error::Error trait, 所以 std::result::Result&lt;T, anyhow::Error&gt; 也可以和 thiserror 的 context 结合使用:

```rust
use anyhow::{bail, Context, Result};
use std::fs;
use std::io::Read;
use thiserror::Error;

// 为自定义类型实现 thiserror::Error
#[derive(Clone, Debug, Eq, Error, PartialEq)]
#[error("Found no username in {0}")] // thiserror::Error 的 attribute macro
struct EmptyUsernameError(String);

fn read_username(path: &str) -> Result<String> { // 返回 anyhow::Result, 内部使用 anyhow::Error
    let mut username = String::with_capacity(100);
    fs::File::open(path)
        .with_context(|| format!("Failed to open {path}"))?
        .read_to_string(&mut username)
        .context("Failed to read")?; // anyhow::Context
    if username.is_empty() {
        bail!(EmptyUsernameError(path.to_string()));
    }
    Ok(username)
}

fn main() {
    //fs::write("config.dat", "").unwrap();
    match read_username("config.dat") {
        Ok(username) => println!("Username: {username}"),
        Err(err) => println!("Error: {err :?}"), // anyhow::Error 自定义了 Debug format.
    }
}
```

`anyhow::anyhow!()` 和老版本 `anyhow::bail!()` 宏提供了快速创建 anyhow::Error 类型的对象:

-   bail!() 创建和 return anyhow::Error;

<!--listend-->

```rust
use anyhow::{anyhow, Result};

fn lookup(key: &str) -> Result<V> {
    if key.len() != 16 {
        return Err(anyhow!("key length must be 16 characters, got {:?}", key));
    }
    // ...
}

if !has_permission(user, resource) {
    bail!("permission denied for accessing {}", resource);
}

#[derive(Error, Debug)]
enum ScienceError {
    #[error("recursion limit exceeded")]
    RecursionLimitExceeded,
    ...
}

if depth > MAX_DEPTH {
    bail!(ScienceError::RecursionLimitExceeded);
}
```
