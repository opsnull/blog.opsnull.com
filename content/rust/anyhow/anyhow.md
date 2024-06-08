---
title: "anyhow"
author: ["zhangjun"]
lastmod: 2024-06-08T21:57:42+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 6
---

[anyhow::Error](https://docs.rs/anyhow/latest/anyhow/) 实现了 From&lt;E&gt; where E: StdError + Send + Sync + 'static，所有可以使用 ? 将标准 Error
转换为 anyhow::Error.

同时 anyhow::Error 也实现了其他 trait，可以将自身转换为标准 Error；

-   Deref&lt;Target = dyn std::Error + Sync + Send&gt; ，所以可以作为标准 Error 来使用。
-   From&lt;Error&gt; for Box&lt;dyn StdError + 'static&gt;
-   From&lt;Error&gt; for Box&lt;dyn StdError + Send + 'static&gt;
-   From&lt;Error&gt; for Box&lt;dyn StdError + Send + Sync + 'static&gt;:

anyhow::Error 方法：

-   new(error): 从标准库 error 创建 anyhow::Error
-   msg(message): 从 message 创建 anyhow::Error
-   context(context): 为 error 添加 context 信息

<!--listend-->

```rust

impl Error

// new 将 std Error 的错误转换为 anyhow::Error
pub fn new<E>(error: E) -> Self where E: StdError + Send + Sync + 'static, // StdError 为标准库 core::error::Error

// Create a new error object from a printable error message.
pub fn msg<M>(message: M) -> Self where    M: Display + Debug + Send + Sync + 'static,
use anyhow::{Error, Result};
use futures::stream::{Stream, StreamExt, TryStreamExt};
async fn demo<S>(stream: S) -> Result<Vec<Output>>
    where
        S: Stream<Item = Input>,
    {
        stream
            .then(ffi::do_some_work) // returns Result<Output, &str>
            .map_err(Error::msg)
            .try_collect()
            .await
    }


// Wrap the error value with additional context.
pub fn context<C>(self, context: C) -> Self where C: Display + Send + Sync + 'static,
// 示例
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

anyhow::Result&lt;T&gt; 是 std::Result&lt;T, anyhow::Error&gt; 的类型别名，它使用的 anyhow::Error 在显示出错信息时包含 context 和 backtrace 。

```rust
use anyhow::{Context, Result};

// Result<()> 使用默认的 anyhow::Error
fn main() -> Result<()> {
    //...
    it.detach().context("Failed to detach the important thing")?;

    let content = std::fs::read(path).with_context(|| format!("Failed to read instrs from {}", path))?;
    //...

    // 其他实现标准库 Error 的类型会被自动转换为 anyhow::Error
    let config = std::fs::read_to_string("cluster.json")?;
    let map: ClusterMap = serde_json::from_str(&config)?;
    println!("cluster info: {:#?}", map);
    Ok(())
}
```

This library provides `anyhow::Error`, a trait object based error type for easy idiomatic error
handling in Rust applications.

Use `Result<T, anyhow::Error>`, or equivalently `anyhow::Result<T>`, as the return type of any fallible
function.

Within the function, use ? to easily propagate any error that implements the `std::error::Error`
trait.

```rust
use anyhow::Result;

fn get_cluster_info() -> Result<ClusterMap> { // 使用 anyhow::Result 作为函数返回值
    let config = std::fs::read_to_string("cluster.json")?;
    let map: ClusterMap = serde_json::from_str(&config)?;
    Ok(map)
}
```

Attach `context` to help the person troubleshooting the error understand where things went wrong. A
low-level error like “No such file or directory” can be annoying to debug without more context about
what higher level step the application was in the middle of.

-   anyhow 默认为 Result 和 Option 实现了 Context trait

<!--listend-->

```rust
use anyhow::{Context, Result};

fn main() -> Result<()> {
    //...
    it.detach().context("Failed to detach the important thing")?;

    let content = std::fs::read(path).with_context(|| format!("Failed to read instrs from {}", path))?;
    //...
}
```

出错示例：

```rust
    Error: Failed to read instrs from ./path/to/instrs.json

    Caused by:
        No such file or directory (os error 2)
```

`Downcasting` is supported and can be by value, by shared reference, or by mutable reference as
needed.

```rust
// If the error was caused by redaction, then return a tombstone instead of the content.
match root_cause.downcast_ref::<DataStoreError>() {
    Some(DataStoreError::Censored(_)) => Ok(Poll::Ready(REDACTED_CONTENT)),
    None => Err(error),
}
```

If using Rust ≥ 1.65, a `backtrace` is captured and printed with the error if the underlying error
type does not already provide its own. In order to see backtraces, they must be enabled through the
environment variables described in std::backtrace:

-   If you want panics and errors to both have backtraces, set RUST_BACKTRACE=1;
-   If you want only errors to have backtraces, set RUST_LIB_BACKTRACE=1;
-   If you want only panics to have backtraces, set RUST_BACKTRACE=1 and RUST_LIB_BACKTRACE=0.

Anyhow works with any error type that has an impl of `std::error::Error`, including ones defined in
your crate. We do not bundle a derive(Error) macro but you can write the impls yourself or use a
standalone macro like `thiserror`.

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum FormatError {
    #[error("Invalid header (expected {expected:?}, got {found:?})")]
    InvalidHeader {
        expected: String,
        found: String,
    },
    #[error("Missing attribute: {0}")]
    MissingAttribute(String),
}
```

One-off error messages can be constructed using the anyhow! macro, which supports string
interpolation and produces an anyhow::Error.

```text
return Err(anyhow!("Missing attribute: {}", missing));
```

A bail! macro is provided as a shorthand for the same early return.

```text
bail!("Missing attribute: {}", missing);
```
