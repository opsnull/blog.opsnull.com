---
title: "tracing"
author: ["zhangjun"]
lastmod: 2024-04-22T15:46:25+08:00
tags: ["rust", "tracing"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 2
---

<https://docs.rs/tracing/latest/tracing/index.html>

A scoped, structured logging and diagnostics system.

核心概念：

1.  Span：To record the flow of execution through a program, tracing introduces the concept of
    spans. Unlike a log line that represents a moment in time, a span represents `a period of time`
    with a beginning and an end. When a program begins executing in a context or performing a unit of
    work, it `enters that context’s span`, and when it stops executing in that context, it `exits the
          span`. The span in which a thread is currently executing is referred to as that thread’s `current
          span`.
    ```rust
       use tracing::{span, Level};
       let span = span!(Level::TRACE, "my_span"); // Level 和 Span id（name）

       // `enter` returns a RAII guard which, when dropped, exits the span. this
       // indicates that we are in the span for the current lexical scope.
       let _enter = span.enter();
       // perform some work in the context of `my_span`...
    ```

2.  Events：An Event represents a moment in time. It signifies something that happened while a trace
    was being recorded. Events are comparable to `the log records` emitted by unstructured logging
    code, but unlike a typical log line, `an Event may occur within the context of a span.`
    ```rust
       use tracing::{event, span, Level};

       // records an event outside of any span context:
       event!(Level::INFO, "something happened"); // Event 也可以位于 span context 之外

       let span = span!(Level::INFO, "my_span");
       let _guard = span.enter();
       // records an event within "my_span".
       event!(Level::DEBUG, "something happened inside my_span");
    ```
    Event 和 Span 都具有级别信息。

3.  Subscribers： As Spans and Events occur, they `are recorded or aggregated by implementations of
          the Subscriber trait`. Subscribers `are notified` when an Event takes place and when a Span is
    entered or exited. These notifications are represented by the following Subscriber trait methods:
    -   event, called when an Event takes place,
    -   enter, called when execution enters a Span,
    -   exit, called when execution exits a Span

In addition, subscribers may implement `the enabled function` to filter the notifications they receive
based on `metadata` describing each Span or Event. If a call to `Subscriber::enabled` returns false for
a given set of metadata, that Subscriber will `not be notified` about the corresponding Span or
Event. For performance reasons, if no currently active subscribers express interest in a given set
of metadata by returning true, then the corresponding Span or Event will `never be constructed`.

Subscriber::enabeld() 可以基于传入的 Span 或 Event 的 metadata 来判断是否需要记录它们.


## <span class="section-num">1</span> span {#span}

使用 span!() 宏来创建特定级别和 id 的 span，然后调用他的 enter() 方法来创建一个 span context，后续在该 span 被 drop 前，所有 event 都属于该 span。

-   span 可以嵌套 enter() 实现 span 嵌套, 这样后续子 span 下打印 event 后会自动关联 父子 span 关系;

<!--listend-->

```rust
use tracing::{span, Level};

// Construct a new span named "my span" with trace log level.
let span = span!(Level::TRACE, "my span");

// Enter the span, returning a guard object.
let _enter = span.enter();

// Any trace events that occur before the guard is dropped will occur within the span.

// Dropping the guard will exit the span.
```

event 也可以使用 target: "span_name" 或 parent: &amp;span 来只指定父 span.

```rust
    // 对于 span, 必须在 name/id 后添加 k=v
    tracing::error_span!("myerrorspan", ?s);
    // tracing::error_span!(?s, "myerrorspan"); // 报错

    // 对于 event, 分两种情况:
    // 1. 如果有 mesage, 自定义 field 都必须位于 message 之前;
    // 2. 如果没有 message, 则可以使用 field=value 形式来定义任意 field.
    tracing::error!(target: "myerrorspan", ?s, s.field_a, "just a debug message2");
```

对于自定义函数，可以使用 #[instrument] 宏来简化 span 的创建：

-   使用函数名作为 span name, 函数参数将作为 span 的 field;

<!--listend-->

```rust
use tracing::{Level, event, instrument};

#[instrument]
pub fn my_function(my_arg: usize) {
    // This event will be recorded inside a span named `my_function` with the field `my_arg`.
    event!(Level::INFO, "inside my_function!");
    // ...
}

// instrument 支持丰富的配置参数
use tracing::instrument;
impl Handler {
    /// Process a single connection.
    #[instrument(
        name = "Handler::run",
        skip(self),
        fields(
            // `%` serializes the peer IP addr with `Display`
            peer_addr = %self.connection.peer_addr().unwrap()
        ),
    )]
    async fn run(&mut self) -> mini_redis::Result<()> {
        ...
    }
}
```

对于不能使用 #[instrument] 的第三方函数，可以使用 info_span:

```rust
use tracing::info_span;

let json = info_span!("json.parse").in_scope(|| serde_json::from_slice(&buf))?; // wrap synchonous code in a span
```


## <span class="section-num">2</span> events {#events}

使用 event!() 宏来记录 event：

```rust
use tracing::{event, Level};

event!(Level::INFO, "something has happened!"); // level 和 message, 字符串字面量默认为 message
// 等效为
event!(Level::INFO, message = "something has happened!")
```


## <span class="section-num">3</span> span和 event 宏 {#span和-event-宏}

span 和 event 都需要指定 Level, span name/id, event message，可选的指定:

1.  target（默认为 module path）;
2.  parent span（默认为current span） 等属性;
3.  name 默认为 event <line>;

注意 Level, message 是有先后关系的:

```rust
let span = span!(Level::TRACE, "my span"); // span 必选: Level 和 span name/id
event!(Level::INFO, "something has happened!");  // Event 必选: event Level 和 message
event!(parent: &span, Level::INFO, "something has happened!");  // 可选的指定 parent span

span!(target: "app_spans", Level::TRACE, "my span");  // 可选的 target
event!(target: "app_events", Level::INFO, "something has happened!");

span!(Level::TRACE, "my span");
event!(name: "some_info", Level::INFO, "something has happened!"); // name 默认为 event file:line, 可以被重写
```

span 和 event 还可以使用逗号分割的 field_name=field_value 来指定自定义属性：

-   对于 span: 只能在 span name/id 字符串后添加 k=v;
-   对于 event:
    -   如果使用了 message, 则所有 field_name=field_value 都必须在 message 字符串前面定义.
    -   如果没有 message, 则可以在 level 后定义所有 kv;

<!--listend-->

```rust
// records an event with two fields:
//  - "answer", with the value 42
//  - "question", with the value "life, the universe and everything"
// 没有 message, level 后指定 k=v
event!(Level::INFO, answer = 42, question = "life, the universe, and everything");
// 有 message, 必须在 message 前定义 k=v
tracing::error!(target: "myerrorspan", ?s, s.field_a, "just a debug message2");


// 对于 span, 必须在 name/id 后添加 k=v
tracing::error_span!("myerrorspan", ?s);
// tracing::error_span!(?s, "myerrorspan"); // 报错

// 对于 event, 分两种情况:
// 1. 如果有 mesage, 自定义 field 都必须位于 message 之前;
// 2. 如果没有 message, 则可以使用 field=value 形式来定义任意 field.
tracing::error!(target: "myerrorspan", ?s, s.field_a, "just a debug message2"); // 有 message
tracing::error!(target: "myerrorspan", ?s, s.field_a, a = "b");  // 没有 message

// 在 enter _span drop 前, 后续的 event 都自动关联该 span
let _span = tracing::error_span!("my_enter_span", ?s).entered();
tracing::error!(?s, s.field_a, a = "c");


let user = "ferris";
span!(Level::TRACE, "login", user);
// is equivalent to:
span!(Level::TRACE, "login", user = user);

let user = "ferris";
let email = "ferris@rust-lang.org";
// Field names can include dots, but should not be terminated by them:
span!(Level::TRACE, "login", user, user.email = email);


let user = User {
    name: "ferris",
    email: "ferris@rust-lang.org",
};
// the span will have the fields `user.name = "ferris"` and `user.email = "ferris@rust-lang.org"`.
span!(Level::TRACE, "login", user.name, user.email);


// records an event with fields whose names are not Rust identifiers
//  - "guid:x-request-id", containing a `:`, with the value "abcdef"
//  - "type", which is a reserved word, with the value "request"
span!(Level::TRACE, "api", "guid:x-request-id" = "abcdef", "type" = "request");
```

可以使用 ？和 % 来使用 Display 或 Debug 实现：

```rust
#[derive(Debug)]
struct MyStruct {
    field: &'static str,
}

let my_struct = MyStruct {
    field: "Hello world!"
};

// ？ 使用 Debug
// `my_struct` will be recorded using its `fmt::Debug` implementation.
event!(Level::TRACE, greeting = ?my_struct);
// is equivalent to:
event!(Level::TRACE, greeting = tracing::field::debug(&my_struct));


// % 使用 Display
// `my_struct.field` will be recorded using its `fmt::Display` implementation.
event!(Level::TRACE, greeting = %my_struct.field);
// is equivalent to:
event!(Level::TRACE, greeting = tracing::field::display(&my_struct.field)

// ？ 和 % 也可以用在变量名前
// `my_struct.field` will be recorded using its `fmt::Display` implementation.
event!(Level::TRACE, %my_struct.field);
```

如果为 field 指定特殊的 Empty 值，则后续可以再设置：

```rust
use tracing::{trace_span, field};

// 创建一个 span, Create a span with two fields: `greeting`, with the value "hello world", and
// `parting`, without a value.
let span = trace_span!("my_span", greeting = "hello world", parting = field::Empty);

// ...

// 在 span context 中创建一个 record，为 Empty field 指定具体的值.  Now, record a value for parting
// as well.
span.record("parting", &"goodbye world!");
```

span 和 event 还可以使用 format 字符串：

```rust
let question = "the ultimate question of life, the universe, and everything";
let answer = 42;
// records an event with the following fields:
// - `question.answer` with the value 42,
// - `question.tricky` with the value `true`,
// - "message", with the value "the answer to the ultimate question of life, the
//    universe, and everything is 42."
event!(
    Level::DEBUG,
    question.answer = answer,
    question.tricky = true,
    "the answer to {} is {}.", question, answer  // message
);
```

为了方便创建指定 Level 的 span 和 event, 还可以使用带 level 的特殊宏, 如 trace!/debug! 等:

-   event!:  trace!, debug!, info!, warn!, and error! behave similarly to the event!
-   span!: trace_span!, debug_span!, info_span!, warn_span!, and error_span! macros are the same, but
    for the span! macro.


## <span class="section-num">4</span> event/span metadata {#event-span-metadata}

All spans and events have the following metadata:

-   A name, represented as a static string.
-   A target, a string that categorizes part of the system where the span or event occurred. The tracing macros default to using the module path where the span or event originated as the target, but it may be overridden.
-   A verbosity level. This determines how verbose a given span or event is, and allows enabling or disabling more verbose diagnostics situationally. See the documentation for the Level type for details.
-   The `names of the fields` defined by the span or event.  自定义 field=value
-   Whether the metadata corresponds to a span or event.

In addition, the following optional metadata describing the source code location where the span or
event originated may be provided:

-   The file name
-   The line number
-   The module path


## <span class="section-num">5</span> log 互操作性 {#log-互操作性}

创建 Event 相关的 trace!, debug!, info! 等宏名称, 和 log crate 提供的记录日志的宏名称相同, 可以直接替换使用, tracing 的 Event 包含了更丰富的结构化信息:

1.  tracing crate 可以 emit log crate 消费的 log records；
2.  Subscribers 也可以将 log create 的 log records 当作 tracing Event 来消费(需要使用 tracing-log
    crate)；

Emitting log Records: This crate provides two feature flags, `“log” and “log-always”`, which will
cause `spans` and `events` to `emit log records`.

-   log feature: 在没有激活 tracing Subscriber 的情况下将 tracing event/span 转换为 log record;
-   log-always feature: 即使激活了 tracing Subscriber, 也将 tracing event/span 转换为 log record;
    ```rust
      use std::{error::Error, io};
      use tracing::{debug, error, info, span, warn, Level};

      // the `#[tracing::instrument]` attribute creates and enters a span
      // every time the instrumented function is called. The span is named after the
      // the function or method. Parameters passed to the function are recorded as fields.
      #[tracing::instrument]
      pub fn shave(yak: usize) -> Result<(), Box<dyn Error + 'static>> {
          // this creates an event at the DEBUG level with two fields:
          // - `excitement`, with the key "excitement" and the value "yay!"
          // - `message`, with the key "message" and the value "hello! I'm gonna shave a yak."
          //
          // unlike other fields, `message`'s shorthand initialization is just the string itself.
          debug!(excitement = "yay!", "hello! I'm gonna shave a yak.");
          if yak == 3 {
              warn!("could not locate yak!");
              // note that this is intended to demonstrate `tracing`'s features, not idiomatic
              // error handling! in a library or application, you should consider returning
              // a dedicated `YakError`. libraries like snafu or thiserror make this easy.
              return Err(io::Error::new(io::ErrorKind::Other, "shaving yak failed!").into());
          } else {
              debug!("yak shaved successfully");
          }
          Ok(())
      }

      pub fn shave_all(yaks: usize) -> usize {
          // Constructs a new span named "shaving_yaks" at the TRACE level,
          // and a field whose key is "yaks". This is equivalent to writing:
          //
          // let span = span!(Level::TRACE, "shaving_yaks", yaks = yaks);
          //
          // local variables (`yaks`) can be used as field values
          // without an assignment, similar to struct initializers.
          let _span = span!(Level::TRACE, "shaving_yaks", yaks).entered();

          info!("shaving yaks");

          let mut yaks_shaved = 0;
          for yak in 1..=yaks {
              let res = shave(yak);
              debug!(yak, shaved = res.is_ok());

              if let Err(ref error) = res {
                  // Like spans, events can also use the field initialization shorthand.
                  // In this instance, `yak` is the field being initalized.
                  error!(yak, error = error.as_ref(), "failed to shave yak!");
              } else {
                  yaks_shaved += 1;
              }
              debug!(yaks_shaved);
          }

          yaks_shaved
      }
    ```

When the “log” feature is enabled, if `no tracing Subscriber is active`, invoking an event macro or
creating a span with fields will `emit a log record`. This is intended primarily for use in libraries
which wish to `emit diagnostics` that can be consumed by applications using `tracing or log`, without
paying the additional overhead of emitting both forms of diagnostics when tracing is in use.

-   相比直接的 log record，通过 tracing 发送的 event 或 span 转换为 log record 后，会多一些 structured
    diagnostic data。

Enabling the “log-always” feature will cause log records to be emitted even if a tracing Subscriber
is set. This is intended to be used in applications where `a log Logger is being used` to record a
textual log, and `tracing is used only to record other forms of diagnostics` (such as metrics,
profiling, or distributed tracing data).

Consuming log Records

The `tracing-log crate` provides a compatibility layer which allows `a tracing Subscriber to consume
log records` as though they were `tracing events`. This allows applications using tracing to record the
logs emitted by dependencies `using log as events within the context` of the application’s trace
tree. See that crate’s documentation for details.


## <span class="section-num">6</span> subscriber {#subscriber}

In order to `record` trace events, executables have to use `a Subscriber implementation` compatible with
tracing. A Subscriber implements a way of collecting trace data, such as by logging it to standard
output.

This library does not contain any Subscriber implementations; these are provided by `other crates`.

The simplest way to use a subscriber is to call the `set_global_default` function:

```rust
// 全局 Subsriber
extern crate tracing;
// FooSubscriber 是其他 crate 提供的 Subscriber 实现
let my_subscriber = FooSubscriber::new();
tracing::subscriber::set_global_default(my_subscriber).expect("setting tracing default failed");

// 局部闭包 Subscribe
let my_subscriber = FooSubscriber::new();
tracing::subscriber::with_default(my_subscriber, || {
    // Any trace events generated in this closure or by functions it calls will be collected by
    // `my_subscriber`.
})
```

示例：

```rust
// https://tokio.rs/tokio/topics/tracing

#[tokio::main]
pub async fn main() -> mini_redis::Result<()> {
    // construct a subscriber that prints formatted traces to stdout
    let subscriber = tracing_subscriber::FmtSubscriber::new();
    // use that subscriber to process traces emitted after this point
    tracing::subscriber::set_global_default(subscriber)?;
    ...
}


// 对 subscriber 进行更精细化的配置.
// Start configuring a `fmt` subscriber
let subscriber = tracing_subscriber::fmt()
    // Use a more compact, abbreviated log format
    .compact()
    // Display source code file paths
    .with_file(true)
    // Display source code line numbers
    .with_line_number(true)
    // Display the thread ID an event was recorded on
    .with_thread_ids(true)
    // Don't display the event's target (module path)
    .with_target(true)
    .with_ansi(true)
    .with_env_filter("tracing=trace,tokio=trace,runtime=trace") // tracing=trace 指定运行的 --binary tracing 的日志级别
    //.pretty()
    // Build the subscriber
    .finish();
```


## <span class="section-num">7</span> 相关 crate {#相关-crate}

tracing-futures
: provides a compatibility layer with the futures crate, allowing spans to be
    attached to Futures, Streams, and Executors.

tracing-subscriber
: provides Subscriber implementations and utilities for working with
    Subscribers. This includes a `FmtSubscriber` FmtSubscriber for logging formatted trace data to
    stdout, with similar filtering and formatting to the env_logger crate.

tracing-log
: provides `a compatibility layer` with the log crate, allowing log messages to be recorded as tracing Events within the trace tree. This is useful when a project using tracing have dependencies which use log. Note that if you’re using tracing-subscriber’s FmtSubscriber, you don’t need to depend on tracing-log directly.

tracing-appender
: provides utilities for `outputting tracing data`, including a file appender and
    non blocking writer.


## <span class="section-num">8</span> tracing console {#tracing-console}

Cargo.toml

```toml
[dependencies]
# ...
tokio = { version = "1.15", features = ["full", "tracing"] }
```

.cargo/config.toml

```toml
[build]
rustflags = ["--cfg", "tokio_unstable"]
```

The tokio and runtime tracing targets must be enabled at `the TRACE level`.

初始化 console subcribe:

```text
console_subscriber::init();
```

然后编译运行程序时, 自动在 6669 端口暴露 tokio-console 可以读取的数据.

启动 tokio-console 来链接 6669 端口:

```text
tokio-console
```


## <span class="section-num">9</span> 例子 {#例子}

<https://www.shuttle.rs/blog/2024/01/09/getting-started-tracing-rust>

<https://tokio.rs/tokio/topics/tracing>

<https://burgers.io/custom-logging-in-rust-using-tracing>

```rust
#![allow(dead_code)]

#[tracing::instrument] // instructment 包含一些可以配置的参数.
fn trace_me(a: u32, b: u32) -> u32 {
    tracing::info!("trace_me info message");
    a + b
}

#[derive(Debug)]
struct MyStruct {
    field_a: u8,
    field_b: String,
}

#[tokio::main]
pub async fn main() {
    // construct a subscriber that prints formatted traces to stdout
    //let subscriber = tracing_subscriber::FmtSubscriber::new();

// Start configuring a `fmt` subscriber
let subscriber = tracing_subscriber::fmt()
    // Use a more compact, abbreviated log format
    .compact()
    // Display source code file paths
    .with_file(true)
    // Display source code line numbers
    .with_line_number(true)
    // Display the thread ID an event was recorded on
    .with_thread_ids(true)
    // Don't display the event's target (module path)
    .with_target(true)
    .with_ansi(true)
    .with_env_filter("tracing=trace,tokio=trace,runtime=trace") // tracing=trace 指定运行的 --binary tracing 的日志级别
    //.pretty()
    // Build the subscriber
    .finish();

    // use that subscriber to process traces emitted after this point
    tracing::subscriber::set_global_default(subscriber).unwrap();

    //console_subscriber::init();

    tracing::warn!("just a test");

    trace_me(2, 3);

    let s = MyStruct{field_a: 8, field_b: "fieldB value".to_string()};

    // 对于 span, 必须在 name/id 后添加 k=v
    tracing::error_span!("myerrorspan", ?s);
    // tracing::error_span!(?s, "myerrorspan"); // 报错

    // 对于 event, 分两种情况:
    // 1. 如果有 mesage, 自定义 field 都必须位于 message 之前;
    // 2. 如果没有 message, 则可以使用 field=value 形式来定义任意 field.
    tracing::error!(target: "myerrorspan", ?s, s.field_a, "just a debug message2"); // 有 message
    tracing::error!(target: "myerrorspan", ?s, s.field_a, a = "b");  // 没有 message

    // 在 enter _span drop 前, 后续的 event 都自动关联该 span
    let _span = tracing::error_span!("my_enter_span", ?s).entered();
    tracing::error!(?s, s.field_a, a = "c");

    // 子 span, 打印 event 时会自动先后打印所属的 span
    let _span = tracing::error_span!("my_enter_span2", ?s).entered();
    tracing::error!(?s, s.field_a, a = "d");
    // 2024-03-07T11:54:16.136844Z ERROR ThreadId(01) my_enter_span:my_enter_span2: tracing: src/bin/tracing.rs:59: s=MyStruct { field_a: 8, field_b: "fieldB value" } s.field_a=8 a="d" s=MyStruct { field_a: 8, field_b: "fieldB value" } s=MyStruct { field_a: 8, field_b: "fieldB value" }

    //std::thread::sleep_ms(200000);
}
```

输出日志:

```rust
zj@a:~/codes/rust/mydemo$ cargo run --bin tracing
   Compiling mydemo v0.1.0 (/Users/zhangjun/codes/rust/mydemo)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.82s
     Running `target/debug/tracing`
2024-03-07T11:54:16.136269Z  WARN ThreadId(01) tracing: src/bin/tracing.rs:42: just a test
2024-03-07T11:54:16.136417Z  INFO ThreadId(01) trace_me: tracing: src/bin/tracing.rs:5: trace_me info message a=2 b=3
2024-03-07T11:54:16.136475Z ERROR ThreadId(01) tracing: src/bin/tracing.rs:49: just a debug message1 s=MyStruct { field_a: 8, field_b: "fieldB value" }
2024-03-07T11:54:16.136558Z ERROR ThreadId(01) myerrorspan: src/bin/tracing.rs:52: just a debug message2 s=MyStruct { field_a: 8, field_b: "fieldB value" } s.field_a=8
2024-03-07T11:54:16.136585Z ERROR ThreadId(01) myerrorspan: src/bin/tracing.rs:53: s=MyStruct { field_a: 8, field_b: "fieldB value" } s.field_a=8 a="b"
2024-03-07T11:54:16.136791Z ERROR ThreadId(01) my_enter_span: tracing: src/bin/tracing.rs:56: s=MyStruct { field_a: 8, field_b: "fieldB value" } s.field_a=8 a="c" s=MyStruct { field_a: 8, field_b: "fieldB value" }
2024-03-07T11:54:16.136844Z ERROR ThreadId(01) my_enter_span:my_enter_span2: tracing: src/bin/tracing.rs:59: s=MyStruct { field_a: 8, field_b: "fieldB value" } s.field_a=8 a="d" s=MyStruct { field_a: 8, field_b: "fieldB value" } s=MyStruct { field_a: 8, field_b: "fieldB value" }
```
