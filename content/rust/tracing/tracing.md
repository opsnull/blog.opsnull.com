---
title: "tracing"
author: ["zhangjun"]
lastmod: 2024-04-23T22:14:25+08:00
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
    Event 和 Span 都具有 Level 信息。

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

-   Subscriber::enabeld() 可以基于传入的 Span 或 Event 的 metadata 来判断是否需要记录它们.


## <span class="section-num">1</span> span 和 event 宏 {#span-和-event-宏}

使用 span!() 宏来创建特定 Level 和 id 的 span，然后调用他的 enter() 方法来创建一个 span context，后续在该 span 被 drop 前，所有 event 都属于该 span。

-   span 可以 enter() 实现 span 嵌套, 这样后续子 span 下打印 event 后会自动关联父 span 关系;

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

event 可以使用 target: "span_name" 或 parent: &amp;span 来指定父 span.

```rust
// 对于 span, 必须在 name/id 后添加 k=v
tracing::error_span!("myerrorspan", ?s);
// tracing::error_span!(?s, "myerrorspan"); // 错误

// 对于 event, 分两种情况:
// 1. 如果有 message, 自定义 field 都必须位于 message 之前;
// 2. 如果没有 message, 则可以使用 field=value 形式来定义任意 field.
tracing::error!(target: "myerrorspan", ?s, s.field_a, "just a debug message2");
```

对于自定义函数，可以使用 #[instrument] 宏来简化 span 的创建，他使用函数名作为 span name, 函数参数将作为 span 的 field;

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
        //...
    }
}
```

对于不能使用 #[instrument] 的第三方函数，可以使用 span 的 in_scope():

```rust
use tracing::info_span;

let json = info_span!("json.parse").in_scope(|| serde_json::from_slice(&buf))?; // wrap synchonous code in a span
```

使用 event!() 宏来记录 event：

```rust
use tracing::{event, Level};

event!(Level::INFO, "something has happened!"); // level 和 message, 字符串字面量默认为 message
// 等效为
event!(Level::INFO, message = "something has happened!")
```

span 和 event 都需要指定 Level, span name/id, event message，可选的指定:

1.  name 默认为 event <line>;
2.  target（默认为 module path）;
3.  parent span（默认为 current span） 等属性;

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

可以使用 ？和 % 来使用 Debug 或 Display 实现：

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

event!
: trace!, debug!, info!, warn!, and error! behave similarly to the event!

span!
: trace_span!, debug_span!, info_span!, warn_span!, and error_span! macros are the same,
    but for the span! macro.


## <span class="section-num">2</span> event/span metadata {#event-span-metadata}

All spans and events have the following metadata:

-   A `name`, represented as a static string.
-   A `target`, a string that categorizes part of the system where the span or event occurred. The
    tracing macros default to using `the module path` where the span or event originated as the target,
    but it may be overridden.
-   A verbosity `level`. This determines how verbose a given span or event is, and allows enabling or
    disabling more verbose diagnostics situationally. See the documentation for the Level type for
    details.
-   The `names of the fields` defined by the span or event. 自定义 field=value
-   Whether the metadata corresponds to a span or event.

In addition, the following optional metadata describing the source code location where the span or
event originated may be provided:

-   The file name
-   The line number
-   The module path

Metadata::new() 创建 Metadata 对象：

-   Kind 可选值为 EVENT，SPAN，HINT

<!--listend-->

```rust
impl<'a> Metadata<'a>
pub const fn new(
    name: &'static str,
    target: &'a str,
    level: Level,
    file: Option<&'a str>,
    line: Option<u32>,
    module_path: Option<&'a str>,
    fields: FieldSet,
    kind: Kind
) -> Metadata<'a>
```

subscribe 可以使用这些 metadata 来对 span/event 进行过滤（enable() 方法）。


## <span class="section-num">3</span> log 互操作性 {#log-互操作性}

创建 Event 的 trace!, debug!, info! 等宏名称和 log crate 提供的记录日志的宏名称相同, 可以直接替换使用, tracing 的 Event 包含了更丰富的结构化信息。

另外 tracing 也提供了 log crate 的互操作性：

1.  tracing crate 可以 emit log crate 消费的 log records；
2.  Subscribers 也可以将 log crate 的 log records 当作 tracing Event 来消费(需要使用 tracing-log
    crate)；

Emitting log Records: This crate provides two feature flags, `“log” and “log-always”`, which will
cause `spans` and `events` to `emit log records`.

-   log feature: 在没有激活 tracing Subscriber 的情况下将 tracing event/span 转换为 log record;
-   log-always feature: 即使激活了 tracing Subscriber, 也将 tracing event/span 转换为 log record;

生成的 log record 包含 span/event 的 fileds 和 metadata（如 target，level，module path，file，line
number 等）。而且 span 的 entered/exited/close 也会创建 log record，他们的 log target 为
tracing::span， 遮掩gkeyi使用 log 的开启或关闭机制来操作他们。

Consuming log Records： The `tracing-log crate` provides a compatibility layer which allows `a tracing
Subscriber to consume log records` as though they were `tracing events`. This allows applications using
tracing to record the logs emitted by dependencies `using log as events within the context` of the
application’s trace tree. See that crate’s documentation for details.

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


## <span class="section-num">4</span> subscriber {#subscriber}

In order to `record` trace events, executables have to use `a Subscriber implementation` compatible with
tracing. A Subscriber implements a way of collecting trace data, such as by logging it to standard
output. This library does not contain any Subscriber implementations; these are provided by `other
crates`.

tracing crate 定义的 `Subscriber trait` 代表需要收集 trace/event 数据的函数接口，tracing crate 并没有提供该 trait 的实现，但其他 crate，如 tracing-subscriber crate 的 Registry/fmt::Subscriber struct 类型都实现了 tracing::Subscriber trait, 故可以用于
tracing::subscriber::set_default()/set_global_default()/with_default() 的参数。

-   set_default()：为当前线程设置缺省的 Subscribe 实现；
-   set_global_default()：为程序所有线程设置缺省的 Subscribe 实现；
-   with_default() ：为闭包代码指定使用的缺省 Subscribe 实现；

<!--listend-->

```rust
// 全局 Subsriber
extern crate tracing;
// FooSubscriber 是其他 crate 提供的 Subscriber 实现
let my_subscriber = FooSubscriber::new();
tracing::subscriber::set_global_default(my_subscriber).expect("setting tracing default failed");

// 局部闭包 Subscribe，可以按需创建多个 subscriber，分别来使用。
let my_subscriber = FooSubscriber::new();
tracing::subscriber::with_default(my_subscriber, || {
    // Any trace events generated in this closure or by functions it calls will be collected by
    // `my_subscriber`.
})
```

set_global_default() 的底层是用指定的 Subscribe trait 实现来创建一个 tracing::dispatcher::Dispatch
对象，然后设置它为全局 dispatcher：

```rust
pub fn set_global_default<S>(subscriber: S) -> Result<(), SetGlobalDefaultError>
where
    S: Subscriber + Send + Sync + 'static,
{
    crate::dispatcher::set_global_default(crate::Dispatch::new(subscriber))
}
```

tracing::dispatcher::Dispatch 类型和设置方法， `Dispatch 对象负责发送 trace 数据给 Subscriber 实现` ：

-   set_default()：为当前线程设置缺省的 Dispatch；
-   set_global_default()：为程序所有线程设置缺省的 Dispatch；
-   with_default() ：为闭包代码指定使用的缺省 Dispatch；
-   get_default()：返回当前线程使用的 Dispatch；

<!--listend-->

```rust
pub struct Dispatch { /* private fields */ }
// 从 Subscriber 实现来创建 Dispatch
pub fn new<S>(subscriber: S) -> Dispatch where S: Subscriber + Send + Sync + 'static,

// 创建一个 dispatch
use dispatcher::Dispatch;
let my_subscriber = FooSubscriber::new();
let my_dispatch = Dispatch::new(my_subscriber);
// 使用方式1: 为闭包函数设置缺省 Subscribe
dispatcher::with_default(&my_dispatch, || {
    // my_subscriber is the default
});

// 使用方式2: 为全局所有线程设置缺省 Subscribe
dispatcher::set_global_default(my_dispatch)
    // `set_global_default` will return an error if the global default
    // subscriber has already been set.
    .expect("global default was already set!");
// `my_subscriber` is now the default

// 使用方式3: 为当前线程设置缺省 Subscribe
dispatcher::set_default(my_dispatch)
    .expect("default was already set!");
```

tracing::subscriber::Subscriber 定义了收集 trace data 的函数接口：

-   enabled() : 根据 Metadata 来判断是否要记录该 record；
-   new_span(): 根据 Attributes 返回一个 span ID；
-   enter(): 开启一个新的 span；
-   exit(): 退出（完成）一个 span；

<!--listend-->

```rust
pub trait Subscriber: 'static {
    // Required methods
    fn enabled(&self, metadata: &Metadata<'_>) -> bool;
    fn new_span(&self, span: &Attributes<'_>) -> Id;
    fn record(&self, span: &Id, values: &Record<'_>);
    fn record_follows_from(&self, span: &Id, follows: &Id);
    fn event(&self, event: &Event<'_>);
    fn enter(&self, span: &Id);
    fn exit(&self, span: &Id);

    // Provided methods
    fn on_register_dispatch(&self, subscriber: &Dispatch) { ... }
    fn register_callsite(&self, metadata: &'static Metadata<'static> ) -> Interest { ... }
    fn max_level_hint(&self) -> Option<LevelFilter> { ... }
    fn event_enabled(&self, event: &Event<'_>) -> bool { ... }
    fn clone_span(&self, id: &Id) -> Id { ... }
    fn drop_span(&self, _id: Id) { ... }
    fn try_close(&self, id: Id) -> bool { ... }
    fn current_span(&self) -> Current { ... }
    unsafe fn downcast_raw(&self, id: TypeId) -> Option<*const ()> { ... }
}
```


## <span class="section-num">5</span> tracing-subscriber {#tracing-subscriber}

tracing crate 定义的 Subscriber trait 代表需要收集 trace/event 数据的函数接口。

tracing-subscriber crate 的 Registry/fmt::Subscriber struct 类型都实现了 tracing::Subscriber trait,
故可以用于 tracing::subscriber::set_default()/set_global_default()/with_default() 的参数。

-   tracing-subscriber::fmt::Subscriber struct 实现了 tracing::Subscriber trait，可以用作 tracing 的全局 Subscribe；
    -   tracing-subscriber::fmt() 返回的 tracing_subscriber::fmt::SubscriberBuilder 的 .with_writer()方法可以指定一个实现 std::io::Writer 的参数，从而实现自定义的终端、文件写入。（可以使用
        tracing-appender crate 来生成这两种 writer）。
-   Registry 可以通过 .with(Layer) 方式来自定义 trace 数据的过滤、格式化和写入，非常适合于复杂自定义场景，如和 opentelemetry 集成；

<!--listend-->

```rust
// https://tokio.rs/tokio/topics/tracing
#[tokio::main]
pub async fn main() -> mini_redis::Result<()> {
    // construct a subscriber that prints formatted traces to stdout
    let subscriber = tracing_subscriber::FmtSubscriber::new();
    // use that subscriber to process traces emitted after this point
    tracing::subscriber::set_global_default(subscriber)?;
    //...
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
// use that subscriber to process traces emitted after this point
tracing::subscriber::set_global_default(subscriber).unwrap();
```

tracing_subscriber::fmt::Subscriber struct 实现了 tracing::Subscriber trait：

-   支持通过环境变量来配置 EnvFilter，如 RUST_LOG=debug,my_crate=trace;
-   fmt() 返回一个 SubscriberBuilder 对象，然后进行详细配置，最后的 .finish() 返回一个
    tracing_subscriber::fmt::Subscriber 对象;

<!--listend-->

```rust
use tracing_subscriber;
tracing_subscriber::fmt::init(); // 创建 fmt::Subscriber 并设置为 tracing crate 的 global Subscriber

let subscriber = tracing_subscriber::fmt()
    // ... add configuration
    .finish();
tracing::subscriber::set_global_default(subscriber).unwrap(); // 设置为 tracing crate 的 global Subscriber

let subscriber = tracing_subscriber::fmt()
    // ... add configuration
    .init() // 内部调用 builder.finish() 然后设置为 tracing crate 的 global Subscriber
```

tracing_subscriber::fmt::format()：用于定义 event formatter, 可以通过 .with_XX() 方法来设置是否输出对应内容:

```rust
pub fn format() -> Format
// 返回的 Struct tracing_subscriber::fmt::format::Format 定义:
pub struct Format<F = Full, T = SystemTime> { /* private fields */ }

let format = tracing_subscriber::fmt::format()
    .without_time()         // Don't include timestamps
    .with_target(false)     // Don't include event targets.
    .with_level(false)      // Don't include event levels.
    .compact();             // Use a more compact, abbreviated format.

// Use the configured formatter when building a new subscriber.
tracing_subscriber::fmt()
    .event_format(format)
    .init();
```

tracing_subscriber::fmt::layer(): 返回一个 tracing_subscriber::fmt::Layer 对象, 用来组合生成
Subscriber:

-   tracing_subscriber::fmt::Layer 的方法和 tracing_subscriber::fmt::format::Format 类似, 可以通过
    .with_XX() 方法来设置是否输出对应内容:
-   tracing_subscriber::fmt::Layer 类型实现了 tracing_subscriber::layer::Layer trait, 该 trait 是
    tracing_subscriber::registry::Registry.with(layer) 的输入类型;
-   tracing_subscriber::fmt::Layer 的 .with_writer() 方法, 可以用于自定义 event writer;

<!--listend-->

```rust
pub fn layer<S>() -> Layer<S>
// 返回的 Struct tracing_subscriber::fmt::Layer 定义:
pub struct Layer<S, N = DefaultFields, E = Format<Full>, W = fn() -> Stdout> { /* private fields */ }

use tracing_subscriber::{fmt, Registry};
use tracing_subscriber::fmt::{self, format, time};
use tracing_subscriber::prelude::*;

let subscriber = Registry::default()
    .with(fmt::Layer::default()); // 创建和使用的默认的 Layer 对象配置

let fmt = format().with_timer(time::Uptime::default());
let fmt_layer = fmt::layer() // 自定义 Layer 对象配置
   .with_target(false) // don't include event targets when logging
   .with_level(false) // don't include event levels when logging
   .event_format(fmt);
// 使用多个 Layer 来创建一个灵活的实现 tracing::subscriber::Subscriber 的 Registry 对象
let subscriber = Registry::default().with(fmt_layer);


use std::io;
use tracing_subscriber::fmt;
let layer = fmt::layer()
    .with_writer(io::stderr);
```

tracing_subscriber::layer::Layer trait 用于处理 traicing event, 用于
tracing_subscriber::registry::Registry.with(layer) 的输入, 用于构建 tracing::Subscriber:

-   tracing_subscriber::fmt::Layer 类型实现了 tracing_subscriber::layer::Layer trait;

<!--listend-->

```rust
pub trait Layer<S>
where
    S: Subscriber,
    Self: 'static,
{
    // Provided methods
    fn on_register_dispatch(&self, subscriber: &Dispatch) { ... }
    fn on_layer(&mut self, subscriber: &mut S) { ... }
    fn register_callsite(
        &self,
        metadata: &'static Metadata<'static>
    ) -> Interest { ... }
    fn enabled(&self, metadata: &Metadata<'_>, ctx: Context<'_, S>) -> bool { ... }
    fn on_new_span(&self, attrs: &Attributes<'_>, id: &Id, ctx: Context<'_, S>) { ... }
    fn on_record(&self, _span: &Id, _values: &Record<'_>, _ctx: Context<'_, S>) { ... }
    fn on_follows_from(&self, _span: &Id, _follows: &Id, _ctx: Context<'_, S>) { ... }
    fn event_enabled(&self, _event: &Event<'_>, _ctx: Context<'_, S>) -> bool { ... }
    fn on_event(&self, _event: &Event<'_>, _ctx: Context<'_, S>) { ... }
    fn on_enter(&self, _id: &Id, _ctx: Context<'_, S>) { ... }
    fn on_exit(&self, _id: &Id, _ctx: Context<'_, S>) { ... }
    fn on_close(&self, _id: Id, _ctx: Context<'_, S>) { ... }
    fn on_id_change(&self, _old: &Id, _new: &Id, _ctx: Context<'_, S>) { ... }
    fn and_then<L>(self, layer: L) -> Layered<L, Self, S> ⓘ
       where L: Layer<S>,
             Self: Sized { ... }
    fn with_subscriber(self, inner: S) -> Layered<Self, S> ⓘ
       where Self: Sized { ... }
    fn with_filter<F>(self, filter: F) -> Filtered<Self, F, S> ⓘ
       where Self: Sized,
             F: Filter<S> { ... }
    fn boxed(self) -> Box<dyn Layer<S> + Send + Sync + 'static>
       where Self: Sized + Layer<S> + Send + Sync + 'static,
             S: Subscriber { ... }
}
```

tracing_subscriber::registry::Registry struct 类型实现了 tracing::subscriber::Subscriber, 可以和多个
Layer 结合起来, 实现自定义 Subscriber:

-   tracing_subscriber::registry() 返回 Registry 对象;
-   Registry 的核心功能是生成 span ID;
-   Registry 实现了 SubscriberExt trait 和 SubscriberInitExt trait, 前者的 `with() 方法是配置 Registry
        的核心方法`. 而后者提供的 set_default()/init() 用来将 Registry 作为 tracing crate 的全局
    Subscriber;
    -   with() 的输入是实现 tracing_subscriber::layer::Layer trait 的对象, 如
        tracing_subscriber::fmt::Layer 类型, tracing_opentelemetry::Layer 类型;

<!--listend-->

```rust
use tracing_subscriber::{registry::Registry, Layer, prelude::*};
let subscriber = Registry::default()
    .with(FooLayer::new())
    .with(BarLayer::new());

impl<S> SubscriberExt for S where S: Subscriber,
// Wraps self with the provided layer.
fn with<L>(self, layer: L) -> Layered<L, Self> where L: Layer<Self>, Self: Sized,

impl<T> SubscriberInitExt for T where T: Into<Dispatch>,
fn set_default(self) -> DefaultGuard
fn try_init(self) -> Result<(), TryInitError>
fn init(self)


use tracing_subscriber::{fmt, Registry};
use tracing_subscriber::fmt::{self, format, time};
use tracing_subscriber::prelude::*;
Registry::default().with(fmt::Layer::default()).init()
```


## <span class="section-num">6</span> tracing_opentelemetry {#tracing-opentelemetry}

将 opentelemetry 和 tracing 对接，使用 tracing 的 API 来提供了一个 subscriber，将多个 span 汇聚成一个 trace，然后发送给 opentelemetry 兼容的后端系统。

基于 Layer 和 Registry, tracing_opentelemetry crate 提供了 tracing crate 与 OpenTelemetry 集成协作的功能:

1.  创建一个 TracerProvider;
2.  从 TracerProvider 创建一个 tracer;
3.  使用 tracer 创建一个实现 tracing_subscriber::layer::Layer trait 的 Layer
4.  使用该 Layer 来创建一个实现 tracing::Subscriber trait 的 Registry 对象;
5.  使用该 Subscriber 对象来记录 span/event;

<!--listend-->

```rust
use opentelemetry_sdk::trace::TracerProvider;
use opentelemetry::trace::{Tracer, TracerProvider as _};
use tracing::{error, span};
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::Registry;

// Create a new OpenTelemetry trace pipeline that prints to stdout
let provider = TracerProvider::builder()
    .with_simple_exporter(opentelemetry_stdout::SpanExporter::default())
    .build();
let tracer = provider.tracer("readme_example");

// Create a tracing layer with the configured tracer
let telemetry = tracing_opentelemetry::layer().with_tracer(tracer);

// Use the tracing subscriber `Registry`, or any other subscriber
// that impls `LookupSpan`
let subscriber = Registry::default().with(telemetry);

// Trace executed code
tracing::subscriber::with_default(subscriber, || {
    // Spans will be sent to the configured OpenTelemetry exporter
    let root = span!(tracing::Level::TRACE, "app_start", work_units = 2);
    let _enter = root.enter();

    error!("This event will be logged in the root span.");
});
```

另一个例子:

```rust
// https://github.com/tokio-rs/mini-redis/blob/master/src/bin/server.rs#L59

#[cfg(feature = "otel")]
// To be able to set the XrayPropagator
use opentelemetry::global;
#[cfg(feature = "otel")]
// To configure certain options such as sampling rate
use opentelemetry::sdk::trace as sdktrace;
#[cfg(feature = "otel")]
// For passing along the same XrayId across services
use opentelemetry_aws::trace::XrayPropagator;
#[cfg(feature = "otel")]
// The `Ext` traits are to allow the Registry to accept the
// OpenTelemetry-specific types (such as `OpenTelemetryLayer`)
use tracing_subscriber::{
    fmt, layer::SubscriberExt, util::SubscriberInitExt, util::TryInitError, EnvFilter,
};

#[cfg(feature = "otel")]
fn set_up_logging() -> Result<(), TryInitError> {
    // Set the global propagator to X-Ray propagator
    // Note: If you need to pass the x-amzn-trace-id across services in the same trace,
    // you will need this line. However, this requires additional code not pictured here.
    // For a full example using hyper, see:
    // https://github.com/open-telemetry/opentelemetry-rust/blob/v0.19.0/examples/aws-xray/src/server.rs#L14-L26
    global::set_text_map_propagator(XrayPropagator::default());

    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(opentelemetry_otlp::new_exporter().tonic())
        .with_trace_config(
            sdktrace::config()
                .with_sampler(sdktrace::Sampler::AlwaysOn)
                // Needed in order to convert the trace IDs into an Xray-compatible format
                .with_id_generator(sdktrace::XrayIdGenerator::default()),
        )
        .install_simple()
        .expect("Unable to initialize OtlpPipeline");

    // Create a tracing layer with the configured tracer
    let opentelemetry = tracing_opentelemetry::layer().with_tracer(tracer);

    // Parse an `EnvFilter` configuration from the `RUST_LOG`
    // environment variable.
    let filter = EnvFilter::from_default_env();

    // Use the tracing subscriber `Registry`, or any other subscriber
    // that impls `LookupSpan`
    tracing_subscriber::registry()
        .with(opentelemetry)
        .with(filter)
        .with(fmt::Layer::default())
        .try_init()
}
```

tracing_opentelemetry crate 的核心是提供了一个实现 tracing_subscriber::layer::Layer trait 的
OpenTelemetryLayer 类型:

-   tracing_opentelemetry::layer() 返回该 OpenTelemetryLayer 对象;

<!--listend-->

```rust
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::Registry;
// Use the tracing subscriber `Registry`, or any other subscriber
// that impls `LookupSpan`
let subscriber = Registry::default().with(tracing_opentelemetry::layer());

// pub struct OpenTelemetryLayer<S, T> { /* private fields */ }
```

OpenTelemetryLayer::new(Tracer) 函数返回一个 OpenTelemetryLayer:

-   Trait opentelemetry::trace::Tracer 定义了 Tracer 函数接口;

<!--listend-->

```rust
use tracing_opentelemetry::OpenTelemetryLayer;
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::Registry;

// Create a jaeger exporter pipeline for a `trace_demo` service.
let tracer = opentelemetry_jaeger::new_agent_pipeline()
    .with_service_name("trace_demo")
    .install_simple()
    .expect("Error initializing Jaeger exporter");

// Create a layer with the configured tracer
let otel_layer = OpenTelemetryLayer::new(tracer);
// 或者:
// let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);

// Use the tracing subscriber `Registry`, or any other subscriber
// that impls `LookupSpan`
let subscriber = Registry::default().with(otel_layer);
```


## <span class="section-num">7</span> opentelemetry {#opentelemetry}

[open-telemetry/opentelemetry-rust](https://github.com/open-telemetry/opentelemetry-rust) 项目提供了如下 crate：

-   opentelemetry
-   opentelemetry_sdk
-   opentelemetry-otlp
-   opentelemetry-stdout
-   opentelemetry-http

opentelemetry 提供 global/logs/metrics/trace/context/propagation 等 module:

-   logs module: 提供了 Logger 和 LoggerProvider trait 定义, 同时定义了 LogRecord/LogRdcordBuilder
    struct 类型;
-   metrics module: 提供了 Counter/Guage/Histogram/UpDownCounter/Observer 和 MeterProvider trait 定义,
    同时也定义了 Meter/Counter/Guage/Histogram/UpDownCounter/Observer struct 类型;
-   trace module: 提供了 Span/Tracer trait 和 TracerProvider 定义, 同时定义了
    SpanBuilder/SpanId/TraceId struct 类型;
-   context module：提供了execution-scoped collection of values.
    -   Context struct 类型为 execution-scoped collection of values 提供了封装和传递机制；
    -   Context::with_value() 来写入数据，get() 来获得对应类型的数据，一般使用应用定义的自定义类型来区分不同的数据；
    -   Context::attch() 方法为当前 thread 设置一个默认的 Context。Context 是可以嵌套的，通过 drop 返回的 ContextGuard，可以恢复上一个 Context；通过 Context::current() 返回当前 Context；
-   propagation module：提供了跨服务的上下文分布式追踪功能：
    -   Propagator：从应用间交换的 message 中读取和写入 Context data；
    -   propagation module 提供了名为 Trait
        opentelemetry::propagation::text_map_propagator::TextMapPropagator 的 Propagator，他提供了
        inject() 和 extract() 方法，用来写入和提取 Context；
    -   Struct opentelemetry::propagation::composite::TextMapCompositePropagator 实现了
        TextMapPropagator trait，他可以将多个实现 TextMapPropagator trait 的对象组合为一个。
    -   opentelemetry_sdk crate 的 propagation::BaggagePropagator 和 TraceContextPropagator 实现了
        TextMapPropagator；
-   global module:
    1.  定义了 GlobalLoggerProvider/GlobalTracerProvider/GlobalMeterProvider struct 类型, 他们分别实现了 LoggerProvider/TracerProvider/MeterProvider;
    2.  设置/获取 GlobalLoggerProvider/GlobalTracerProvider/GlobalMeterProvider 对象的函数:
        1.  设置全局对象: set_logger_provider()/set_meter_provider()/set_tracer_provider()
        2.  获取全局对象: logger_provider()/meter_provider()/tracer_provider()
        3.  获取指定 name 的 Logger/Meter/Tracer 的实现对象: logger(name)/meter(name)/tracer(name);
    3.  获取和设置全局 TextMapPropagator propagator：
        1.  获取：执行一个闭包 FnMut(&amp;dyn TextMapPropagator) -&gt; T, 返回 T；
        2.  设置：set_text_map_propagator() 设置全局 TextMapPropagator propagator；

global module 提供了 logs/metrics/trace 的全局 API 对象, 这样 app/lib 就可以更方便的从全局对象创建实现 Logger/Meter/Tracer trait 的对象, 更方便使用.

global trace module 提供了 global trace API，他底层使用配置的实现 [TracerProvider trait](https://docs.rs/opentelemetry/latest/opentelemetry/trace/trait.TracerProvider.html) 的 Provider。这样应用代码就不需要使用从 Open Telemetry SDK 创建的 Trace/Metric/Logs 对象，并来回传递他们引用。

```rust
// 应用示例：需要在 main 或 app 其他启动过程中定义全局 tracer provider
use opentelemetry::trace::{Tracer, noop::NoopTracerProvider};
use opentelemetry::global;
fn init_tracer() {
    // Swap this no-op provider for your tracing service of choice (jaeger, zipkin, etc)
    let provider = NoopTracerProvider::new();
    // Configure the global `TracerProvider` singleton when your app starts
    // (there is a no-op default if this is not set by your application)
    let _ = global::set_tracer_provider(provider);
}
fn do_something_tracked() {
    // Then you can get a named tracer instance anywhere in your codebase.
    let tracer = global::tracer("my-component"); // 从全局 trace provider 中获得一个 tracer
    tracer.in_span("doing_work", |cx| {
        // Traced app logic here...
    });
}
// in main or other app start
init_tracer();
do_something_tracked();


// lib 示例：不需要创建全局 tracer provider，而是使用 global::tracer_provider() 获得所在应用定义的 provider。
use opentelemetry::trace::{Tracer, TracerProvider};
use opentelemetry::global;
pub fn my_traced_library_function() {
    // End users of your library will configure their global tracer provider so you can use the
    // global tracer without any setup
    let tracer = global::tracer_provider().versioned_tracer(
        "my-library-name",
        Some(env!("CARGO_PKG_VERSION")),
        Some("https://opentelemetry.io/schemas/1.17.0"),
        None,
    );
    tracer.in_span("doing_library_work", |cx| {
        // Traced library logic here...
    });
}
```

global metrics module 提供了全局可以访问的 global metrics APIs，他底层使用配置的实现[MeterProvider 的
  Provider](https://docs.rs/opentelemetry/latest/opentelemetry/metrics/trait.MeterProvider.html)。这样应用代码就不需要使用从 Open Telemetry SDK 创建的 Metrics 对象，并来回传递他们引用。

```rust
// 应用示例：需要在 main 或 app 其他启动过程中定义全局 metrics provider
use opentelemetry::metrics::{Meter, noop::NoopMeterProvider};
use opentelemetry::{global, KeyValue};

fn init_meter() {
    let provider = NoopMeterProvider::new();

    // Configure the global `MeterProvider` singleton when your app starts
    // (there is a no-op default if this is not set by your application)
    global::set_meter_provider(provider)
}

fn do_something_instrumented() {
    // Then you can get a named tracer instance anywhere in your codebase.
    let meter = global::meter("my-component");
    let counter = meter.u64_counter("my_counter").init();

    // record metrics
    counter.add(1, &[KeyValue::new("mykey", "myvalue")]);
}

// in main or other app start
init_meter();
do_something_instrumented();


// lib 示例
use opentelemetry::{global, KeyValue};
pub fn my_traced_library_function() {
    // End users of your library will configure their global meter provider
    // so you can use the global meter without any setup
    let tracer = global::meter("my-library-name");
    let counter = tracer.u64_counter("my_counter").init();

    // record metrics
    counter.add(1, &[KeyValue::new("mykey", "myvalue")]);
}
```

Context 示例：

```rust
use opentelemetry::Context;

// 自定义的各种类型作为 Context 的 value 类型
// Application-specific `a` and `b` values
#[derive(Debug, PartialEq)]
struct ValueA(&'static str);
#[derive(Debug, PartialEq)]
struct ValueB(u64);

let _outer_guard = Context::new().with_value(ValueA("a")).attach(); // 为当前 thread 设置缺省 Context

// Only value a has been set
let current = Context::current();
assert_eq!(current.get::<ValueA>(), Some(&ValueA("a"))); // 获得 ValueA 类型对应的值
assert_eq!(current.get::<ValueB>(), None);

{
    let _inner_guard = Context::current_with_value(ValueB(42)).attach();
    // Both values are set in inner context
    let current = Context::current();
    assert_eq!(current.get::<ValueA>(), Some(&ValueA("a")));
    assert_eq!(current.get::<ValueB>(), Some(&ValueB(42)));
    // drop _inner_guard 后恢复以前的 Context
}

// Resets to only the `a` value when inner guard is dropped
let current = Context::current();
assert_eq!(current.get::<ValueA>(), Some(&ValueA("a")));
assert_eq!(current.get::<ValueB>(), None);
```

Context 实现了 BaggageExt trait 和 opentelemetry::trace::TraceContextExt：

-   BaggageExt：可以将 Baggage 中的数据合并到 Context 中；
-   TraceContextExt：从 Span 来创建一个新的 Context；

<!--listend-->

```rust
impl BaggageExt for Context

// Returns a clone of the given context with the included name/value pairs.
fn with_baggage<T: IntoIterator<Item = I>, I: Into<KeyValueMetadata>>(
    &self,
    baggage: T
) -> Self
// Returns a clone of the current context with the included name/value pairs. Read more
fn current_with_baggage<T: IntoIterator<Item = I>, I: Into<KeyValueMetadata>>(
    kvs: T
) -> Self
// Returns a clone of the given context with no baggage. Read more
fn with_cleared_baggage(&self) -> Self
// Returns a reference to this context’s baggage, or the default empty baggage if none has been set.
fn baggage(&self) -> &Baggage


impl TraceContextExt for Context
// Returns a clone of the current context with the included Span. Read more
fn current_with_span<T: Span + Send + Sync + 'static>(span: T) -> Self
// Returns a clone of this context with the included span. Read more
fn with_span<T: Span + Send + Sync + 'static>(&self, span: T) -> Self
// Returns a reference to this context’s span, or the default no-op span if none has been set. Read more
fn span(&self) -> SpanRef<'_>
// Returns whether or not an active span has been set. Read more
fn has_active_span(&self) -> bool
// Returns a copy of this context with the span context included. Read more
fn with_remote_span_context(&self, span_context: SpanContext) -> Self
```

opentelemetry::trace::Tracer trait 的部分方法使用 Context 来创建 Span：

```rust
pub trait Tracer {
    type Span: Span;

    // Required method
    fn build_with_context(
        &self,
        builder: SpanBuilder,
        parent_cx: &Context
    ) -> Self::Span;

    // Provided methods
    fn start<T>(&self, name: T) -> Self::Span where T: Into<Cow<'static, str>> { ... }
    fn start_with_context<T>(&self, name: T, parent_cx: &Context) -> Self::Span where T: Into<Cow<'static, str>> { ... }
    fn span_builder<T>(&self, name: T) -> SpanBuilder where T: Into<Cow<'static, str>> { ... }
    fn build(&self, builder: SpanBuilder) -> Self::Span { ... }
    fn in_span<T, F, N>(&self, name: N, f: F) -> T
       where F: FnOnce(Context) -> T,
             N: Into<Cow<'static, str>>,
             Self::Span: Send + Sync + 'static { ... }
}

// 示例
use opentelemetry::{global, trace::{Span, Tracer, TraceContextExt}, Context};
let tracer = global::tracer("my-component");
let parent = tracer.start("foo");
let parent_cx = Context::current_with_span(parent);
let mut child = tracer.start_with_context("bar", &parent_cx);
// ...
child.end(); // explicitly end
drop(parent_cx) // or implicitly end on drop
```

Propagator 示例：

-   Propagator 使用 inject_context() 将传入的 Context 信息注入 injector 如 HashMash；
-   global::get_text_map_propagator() 的参数是一个闭包，可以从传入的 Propagator 来提取出 Context；

<!--listend-->

```rust
use opentelemetry::{
    baggage::BaggageExt,
    propagation::{TextMapPropagator, TextMapCompositePropagator},
    trace::{TraceContextExt, Tracer, TracerProvider},
    Context, KeyValue,
};
use opentelemetry_sdk::propagation::{BaggagePropagator, TraceContextPropagator,};
use opentelemetry_sdk::trace as sdktrace;
use std::collections::HashMap;

// First create 1 or more propagators
// BaggagePropagator 和 TraceContextPropagator 均实现了 opentelemetry::TextMapPropagator
let baggage_propagator = BaggagePropagator::new();
let trace_context_propagator = TraceContextPropagator::new();

// Then create a composite propagator
// 将两个 propagators 组合为一个 propagators
let composite_propagator = TextMapCompositePropagator::new(vec![
    Box::new(baggage_propagator),
    Box::new(trace_context_propagator),
]);

// Then for a given implementation of `Injector`
let mut injector = HashMap::new();

// And a given span
let example_span = sdktrace::TracerProvider::default()
    .tracer("example-component")
    .start("span-name");

// with the current context, call inject to add the headers
composite_propagator.inject_context(
    &Context::current_with_span(example_span)
        .with_baggage(vec![KeyValue::new("test", "example")]),
    &mut injector,
);

// The injector now has both `baggage` and `traceparent` headers
assert!(injector.get("baggage").is_some());
assert!(injector.get("traceparent").is_some());



// 另一个例子，使用 global::get_text_map_propagator() 来提取和注入信息。

// Ensure context propagation optional
// Context propagation is particularly important when network calls (for example, REST) are involved.
// Method to extract the parent context from the request

// 从 reque 中提取 Context 信息
fn get_parent_context(req: Request<Body>) -> Context {
    global::get_text_map_propagator(|propagator| {
        propagator.extract(&HeaderExtractor(req.headers()))
    })
}

async fn incoming_request(req: Request<Body>) -> Result<Response<Body>, Infallible> {
    let parent_cx = get_parent_context(req);

    let mut span = global::tracer("manual-server")
        .start_with_context("my-server-span", &parent_cx); //TODO Replace with the name of your span

    span.set_attribute(KeyValue::new("my-server-key-1", "my-server-value-1")); //TODO Add attributes

    // TODO Your incoming_request code goes here
}
// 向发出的 hyper 请求注入 context
async fn outgoing_request(
    context: Context,
) -> std::result::Result<(), Box<dyn std::error::Error + Send + Sync + 'static>> {
    let client = Client::new();
    let span = global::tracer("manual-client")
        .start_with_context("my-client-span", &context); //TODO Replace with the name of your span
    let cx = Context::current_with_span(span);

    let mut req = hyper::Request::builder().uri("<HTTP_URL>");

    //Method to inject the current context in the request
    global::get_text_map_propagator(|propagator| {
        propagator.inject_context(&cx, &mut HeaderInjector(&mut req.headers_mut().unwrap()))
    });

    cx.span()
        .set_attribute(KeyValue::new("my-client-key-1", "my-client-value-1")); //TODO Add attributes

    // TODO Your outgoing_request code goes here
}
```

opentelemetry 的 logs/metrics/trace module 定义了 LoggerProvider/MeterProvider/TracerProvider trait,
而且 global module 的 set_logger_provider()/set_meter_provider()/set_tracer_provider() 都需要传入这些 Provider trait 的实现, 但是 opentelemetry crate `并没有提供他们的实现` , 而  opentelemetry_sdk
crate 提供了各种 Provider 的实现.

opentelemetry_sdk crate 提供了各种 Provider 的实现:

-   logs module: 定义了 Logger/LoggerProvider struct 类型, 以及用于配置 Logger 的
    BatchConfig/BatchConfigBuilder, Config/Builder struct 类型;
    -   Logger struct 实现了 Trait opentelemetry::logs::Logger;
    -   LoggerProvider struct 实现了 Trait opentelemetry::logs::LoggerProvider;
-   metrics module: 定义了 SdkMeter/SdkMeterProvider struct 类型, 以及用于配置他们的
    ManualReaderBuilder/MeterProviderBuilder struct 类型;
    -   SdkMeterProvider struct 实现了 Trait opentelemetry::metrics::MeterProvider;
    -   MeterProviderBuilder 配置和创建 SdkMeterProvider, 主要方法:
        .with_resource()/.with_reader()/.with_view().
-   trace module: 定义了 Span/Tracer/TracerProvider 类型, 以及用于配置他们的
    Config/Builder/BatchConfig/BatchConfigBuilder:
    -   Tracer struct 实现了 Trait opentelemetry::trace::Tracer;
    -   TracerProvider struct 实现了 Trait opentelemetry::trace::TracerProvider;
    -   Struct opentelemetry_sdk::trace::Builder 提供的方法用于设置 TracerProvider, 如
        with_simple_exporter()/with_batch_exporter()/with_span_processor()/with_config()
-   propagation module：提供了 BaggagePropagator 和 TraceContextPropagator struct：
    1.  BaggagePropagator 遵从 W3C Baggage format 规范；TraceContextPropagator 遵从 W3C TraceContext
        格式规范。
    2.  他们都实现了 Trait opentelemetry::propagation::text_map_propagator::TextMapPropagator，

opentelemetry_sdk metrics:

```rust
use opentelemetry::{global, Context};
use opentelemetry_sdk::metrics::SdkMeterProvider;

fn init_metrics() -> SdkMeterProvider {
    // Setup metric pipelines with readers + views, default has no
    // readers so nothing is exported.
    let provider = SdkMeterProvider::default(); // 缺省配置
    // let meterProviderBuilder = SdkMeterProvider::builder(); // 使用 Builder 来个性化配置

    // Set provider to be used as global meter provider
    let _ = global::set_meter_provider(provider.clone());

    provider
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let provider = init_metrics();

    // create instruments + record measurements

    // force all instruments to flush
    provider.force_flush()?;

    // record more measurements..

    // shutdown ensures any cleanup required by the provider is done,
    // and also invokes shutdown on the readers.
    provider.shutdown()?;

    Ok(())
}
```

opentelemetry_sdk tracer:

```rust
use opentelemetry::global;
use opentelemetry_sdk::trace::TracerProvider;

fn init_tracing() -> TracerProvider {
    let provider = TracerProvider::default(); // 缺省配置
    // let provider = TracerProvider::builder(); // 使用 Builder 来个性化配置

    // Set provider to be used as global tracer provider
    let _ = global::set_tracer_provider(provider.clone());

    provider
}

fn main() {
    let provider = init_tracing();

    // create spans..

    // force all spans to flush
    for result in provider.force_flush() {
        if let Err(err) = result {
            // .. handle flush error
        }
    }

    // create more spans..

    // dropping provider and shutting down global provider ensure all
    // remaining spans are exported
    drop(provider);
    global::shutdown_tracer_provider();
}

// 其他例子
// https://docs.rs/opentelemetry_sdk/latest/opentelemetry_sdk/trace/struct.TracerProvider.html
use opentelemetry::global;
use opentelemetry_sdk::trace::TracerProvider;

fn init_tracing() -> TracerProvider {
    let provider = TracerProvider::default();
    // Set provider to be used as global tracer provider
    let _ = global::set_tracer_provider(provider.clone());
    provider
}

fn main() {
    let provider = init_tracing();
    // create spans..
    // force all spans to flush
    for result in provider.force_flush() {
        if let Err(err) = result {
            // .. handle flush error
        }
    }
    // create more spans..
    // dropping provider and shutting down global provider ensure all
    // remaining spans are exported
    drop(provider);
    global::shutdown_tracer_provider();
}


// 或者直接使用创建的 trace provider
use opentelemetry::{global, trace::{Tracer, TracerProvider as _}};
use opentelemetry_sdk::trace::TracerProvider;
fn main() {
    // Choose an exporter like `opentelemetry_stdout::SpanExporter`
    let exporter = new_exporter();

    // Create a new trace pipeline that prints to stdout
    let provider = TracerProvider::builder()
        .with_simple_exporter(exporter)
        .build();
    let tracer = provider.tracer("readme_example");

    tracer.in_span("doing_work", |cx| {
        // Traced app logic here...
    });

    // Shutdown trace pipeline
    global::shutdown_tracer_provider();
}
```

opentelemetry_sdk propagation：

-   BaggagePropagator 是类似与 k=v 格式的集合；
-   TraceContextPropagator 将 SpanContext 使用 W3C TraceContex 格式的 traceparent 和 tracestatte
    header 来传递。

<!--listend-->

```rust
use opentelemetry::{baggage::BaggageExt, Key, propagation::TextMapPropagator};
use opentelemetry_sdk::propagation::BaggagePropagator;
use std::collections::HashMap;

// Example baggage value passed in externally via http headers
let mut headers = HashMap::new();
headers.insert("baggage".to_string(), "user_id=1".to_string());

let propagator = BaggagePropagator::new();
// can extract from any type that impls `Extractor`, usually an HTTP header map
let cx = propagator.extract(&headers);

// Iterate over extracted name-value pairs
for (name, value) in cx.baggage() {
    // ...
}

// Add new baggage
let cx_with_additions = cx.with_baggage(vec![Key::new("server_id").i64(42)]);

// Inject baggage into http request
propagator.inject_context(&cx_with_additions, &mut headers);

let header_value = headers.get("baggage").expect("header is injected");
assert!(header_value.contains("user_id=1"), "still contains previous name-value");
assert!(header_value.contains("server_id=42"), "contains new name-value pair");
```

Crate opentelemetry_otlp 的各种 Pipeline 的 install_simple()/install_batch() 方法返回实现
opentelemetry::logs::Logger, Trait opentelemetry::trace::Tracer 和 Struct
opentelemetry_sdk::metrics::SdkMeterProvider 对象;

1.  OtlpLogPipeline: 通过 with_log_config()/with_batch_config()/with_exporter() 来配置
    OtlpLogPipeline, 然后通过 install_simple()/install_batch() 返回实现 opentelemetry::logs::Logger
    trait 的对象，同时设置global::set_tracer_provider(), 这样该 tracer 会作为 opentelemetry 的 global
    tracer
2.  OtlpMetricPipeline
3.  OtlpTracePipeline
4.  OtlpPipeline: 他的 tracing()/logging()/metrics() 方法分别返回
    OtlpTracePipeline/OtlpLogPipeline/OtlpMetricPipeline;

当前 opentelemetry_otlp crate 发送 OTLP 格式的 tracing/metrics 数据, 使用 grpc 或 http.

-   grpc: 使用 tonic 作为 grpc layer;
-   http: 使用 http-proto 和 reqwest 实现;

opentelemetry_otlp 定义了 opentelemetry_otlp::OtlpExporterPipeline, 用于指定 tonic/http 协议的
expoter builder:

```rust
impl OtlpExporterPipeline
pub fn tonic(self) -> TonicExporterBuilder
pub fn http(self) -> HttpExporterBuilder
```

由于 opentelemetry_otlp crate 的 OtlpTracePipeline 实现了 Trait opentelemetry::trace::Tracer, 所以可以作为 tracing_opentelemetry::layer().with_tracer(tracer) 来创建一个实现 Trait
tracing_subscriber::layer::Layer trait 的对象, 然后作为 tracing_subscriber::registry().with(layer)的参数来创建一个 Subscriber:

-   opentelemetry_otlp::new_pipeline().tracing().install_simple()/install_batch() 返回一个 tracer 的同时设置 global::set_tracer_provider(), 这样该 tracer 会作为 opentelemetry 的 global tracer；

<!--listend-->

```rust
use opentelemetry::trace::Tracer;
fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync + 'static>> {
    // First, create a OTLP exporter builder. Configure it as you need.
    let otlp_exporter = opentelemetry_otlp::new_exporter().tonic();
    // Then pass it into pipeline builder
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(otlp_exporter)
        .install_simple()?;  // tracer 被设置为 global::set_tracer_provider(), 这样该 tracer 会作为
                             // opentelemetry 的 global tracer

    tracer.in_span("doing_work", |cx| {
        // Traced app logic here...
    });

    Ok(())
}

// 和 tracing 结合使用
let tracer = opentelemetry_otlp::new_pipeline()
    .tracing()
    .with_exporter(opentelemetry_otlp::new_exporter().tonic())
    .with_trace_config(
        sdktrace::config()
            .with_sampler(sdktrace::Sampler::AlwaysOn)
            // Needed in order to convert the trace IDs into an Xray-compatible format
            .with_id_generator(sdktrace::XrayIdGenerator::default()),
    )
    .install_simple() // tracer 被设置为 global::set_tracer_provider(), 这样该 tracer 会作为
    // opentelemetry 的 global tracer
    .expect("Unable to initialize OtlpPipeline");

// Create a tracing layer with the configured tracer
let opentelemetry = tracing_opentelemetry::layer().with_tracer(tracer);

// Parse an `EnvFilter` configuration from the `RUST_LOG`
// environment variable.
let filter = EnvFilter::from_default_env();

// Use the tracing subscriber `Registry`, or any other subscriber
// that impls `LookupSpan`
tracing_subscriber::registry()
    .with(opentelemetry)
    .with(filter)
    .with(fmt::Layer::default())
    .try_init()

// 后续就可以使用 tracing 库的 span/event 宏了
#[tracing::instrument]
async fn hello_world() -> &'static str {
     info!("Received a request!");
     "Hello world!"
}

// 另一个和 tracing 结合使用的例子：https://www.shuttle.rs/blog/2024/04/10/using-opentelemetry-rust
// note that here, localhost:4318 is the default HTTP address
// for a local OpenTelemetry collector
let tracer = opentelemetry_otlp
    ::new_pipeline()
    .tracing()
    .with_exporter(opentelemetry_otlp::new_exporter().http().with_endpoint("localhost:4318"))
    .install_batch(Tokio)
    .unwrap();

// log level filtering here
let filter_layer = EnvFilter::try_from_default_env()
    .or_else(|_| EnvFilter::try_new("info"))
    .unwrap();

// fmt layer - printing out logs
let fmt_layer = fmt::layer().compact();

// turn our OTLP pipeline into a tracing layer
let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);

// initialise our subscriber
subscriber
    .with(filter_layer)
    .with(fmt_layer)
    .with(otel_layer)
    // The error layer needs to go after the otel_layer, because it needs access to the
    // otel_data extension that is set on the span in the otel_layer.
    .with(ErrorTracingLayer::new())
    .init();
```

直接使用 opentelemetry/opentelemetry_otlp 的例子:

-   opentelemetry_otlp::new_exporter() 返回一个 Struct opentelemetry_otlp::OtlpExporterPipeline 对象,
    他的 tonic() 方法返回 TonicExporterBuilder 对象, http() 返回 HttpExporterBuilder 对象.
-   pub struct TonicExporterBuilder 提供了 build_log_exporter()/build_metrics_exporter()/
    build_span_exporter() 来分别返回 LogExporter/MetricsExporter/SpanExporter;
-   TonicExporterBuilder 实现了 WithExportConfig trait, 提供了
    with_endpoint()/with_protocol()/with_timeout()/with_export_config() 配置方法; 这些参数也可以通过
    OTEL_EXPORTER_OTLP_XX 环境变量来配置(优先级低), 如:
    -   OTEL_EXPORTER_OTLP_ENDPOINT
    -   OTEL_EXPORTER_OTLP_TIMEOUT

全量 opentelemetry_otlp 配置:

```rust
use opentelemetry::{KeyValue, trace::Tracer};
use opentelemetry_sdk::{trace::{self, RandomIdGenerator, Sampler}, Resource};
use opentelemetry_sdk::metrics::reader::{DefaultAggregationSelector, DefaultTemporalitySelector};
use opentelemetry_otlp::{Protocol, WithExportConfig, ExportConfig};
use std::time::Duration;
use tonic::metadata::*;

fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync + 'static>> {
    let mut map = MetadataMap::with_capacity(3);

    map.insert("x-host", "example.com".parse().unwrap());
    map.insert("x-number", "123".parse().unwrap());
    map.insert_bin("trace-proto-bin", MetadataValue::from_bytes(b"[binary data]"));

    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter() // OtlpExporterPipeline
            .tonic() // TonicExporterBuilder，实现了 WithExportConfig trait，调用他的 with_xx() 方法
            .with_endpoint("http://localhost:4317") // 如果没有设置, 则会读取环境变量 OTEL_EXPORTER_OTLP_XX
            .with_timeout(Duration::from_secs(3))
            .with_metadata(map)
         )
        .with_trace_config(
            trace::config()
                .with_sampler(Sampler::AlwaysOn)
                .with_id_generator(RandomIdGenerator::default())
                .with_max_events_per_span(64)
                .with_max_attributes_per_span(16)
                .with_max_events_per_span(16)
                .with_resource(Resource::new(vec![KeyValue::new("service.name", "example")])),
        )
        // 会返回一个 tracer, 同时设置 global::set_tracer_provider(), 这样该 tracer 会作为
        // opentelemetry 的 global tracer
        .install_batch(opentelemetry_sdk::runtime::Tokio)?;

    let export_config = ExportConfig {
        endpoint: "http://localhost:4317".to_string(),
        timeout: Duration::from_secs(3),
        protocol: Protocol::Grpc
    };

    let meter = opentelemetry_otlp::new_pipeline()
        .metrics(opentelemetry_sdk::runtime::Tokio)
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .tonic()
                .with_export_config(export_config),
                // can also config it using with_* functions like the tracing part above.
        )
        .with_resource(Resource::new(vec![KeyValue::new("service.name", "example")]))
        .with_period(Duration::from_secs(3))
        .with_timeout(Duration::from_secs(10))
        .with_aggregation_selector(DefaultAggregationSelector::new())
        .with_temporality_selector(DefaultTemporalitySelector::new())
        .build();

    tracer.in_span("doing_work", |cx| {
        // Traced app logic here...
    });

    Ok(())
}
```

```rust
use opentelemetry_sdk::metrics::reader::{
    DefaultAggregationSelector, DefaultTemporalitySelector,
};
// Create a span exporter you can use to when configuring tracer providers
let span_exporter = opentelemetry_otlp::new_exporter().tonic().build_span_exporter()?;
// Create a metrics exporter you can use when configuring meter providers
let metrics_exporter = opentelemetry_otlp::new_exporter()
    .tonic()
    .build_metrics_exporter(
        Box::new(DefaultAggregationSelector::new()),
        Box::new(DefaultTemporalitySelector::new()),
    )?;

// Create a log exporter you can use when configuring logger providers
let log_exporter = opentelemetry_otlp::new_exporter().tonic().build_log_exporter()?;


// 例子1:
use opentelemetry::trace::Tracer;
fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync + 'static>> {
    // First, create a OTLP exporter builder. Configure it as you need.
    let otlp_exporter = opentelemetry_otlp::new_exporter().tonic();
    // Then pass it into pipeline builder
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(otlp_exporter)
        .install_simple()?;

    tracer.in_span("doing_work", |cx| {
        // Traced app logic here...
    });
    Ok(())
}


use opentelemetry::sdk::Resource;
use opentelemetry::trace::TraceError;
use opentelemetry::{global, sdk::trace as sdktrace};
use opentelemetry::{trace::Tracer};
use opentelemetry_otlp::WithExportConfig;
fn init_tracer() -> Result<sdktrace::Tracer, TraceError> {
    opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(opentelemetry_otlp::new_exporter().tonic()) // 默认从环境变量获取 exporter 配置
        .with_trace_config(
            sdktrace::config().with_resource(Resource::default()),
        )
        .install_batch(opentelemetry::runtime::Tokio)
}
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error + Send + Sync + 'static>> {
    let tracer = init_tracer()?;
    let parent_cx = global::get_text_map_propagator(|propagator| {
        propagator.extract(&HeaderExtractor(req.headers()))
    });
    tracer.start_with_context("fibonacci", &parent_cx);
    //...
}

// 后续可以使用环境变量来设置 RESOURCE 和 ENDPOINT
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 OTEL_RESOURCE_ATTRIBUTES=service.name=rust-app cargo run
```

从 http request headers 中提取 text map propagator，从而实现分布式 trace context：

Context propagation – The mechanism that allows us to correlate events across distributed
services. Context is referred to as the metadata we collect and transfer. Propagation is how the
context is packaged and transferred across services, often via HTTP headers. Context propagation is
one of the areas where OpenTelemetry shines.

opentelemetry-http crate 提供了从 HTTP Header 中提取 propagating/extracing context 的能力，从而实现跨服务的分布式追踪：

{{< figure src="/images/tracing/2024-04-23_18-41-10_screenshot.png" width="400" >}}

```rust
// https://github.com/SigNoz/sample-rust-app/blob/main/src/main.rs
#![warn(rust_2018_idioms)]

use opentelemetry::global::shutdown_tracer_provider;
use opentelemetry::sdk::Resource;
use opentelemetry::trace::TraceError;
use opentelemetry::{global, sdk::trace as sdktrace};
use opentelemetry::{trace::Tracer};
use opentelemetry_otlp::WithExportConfig;
use std::error::Error;

use hyper::{body::Body, Method, Request, Response, Server, StatusCode};

use hyper::service::{make_service_fn, service_fn};

use opentelemetry_http::HeaderExtractor;
use std::collections::HashMap;
use url::form_urlencoded;

static INDEX: &[u8] = b"<html><body><form action=\"post\" method=\"post\">Name: <input type=\"text\" name=\"name\"><br>Number: <input type=\"text\" name=\"number\"><br><input type=\"submit\"></body></html>";
static MISSING: &[u8] = b"Missing field";
static NOTNUMERIC: &[u8] = b"Number field is not numeric";

fn fibonacci(n: u8) -> u64 {
    match n {
        0 => 1,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

// Using service_fn, we can turn this function into a `Service`.
async fn handle(req: Request<Body>) -> Result<Response<Body>, hyper::Error> {
    let tracer = global::tracer("global_tracer");

    match (req.method(), req.uri().path()) {
        (&Method::GET, "/") | (&Method::GET, "/post") => Ok(Response::new(INDEX.into())),
        (&Method::POST, "/post") => {
            // Extract the incoming context
            let parent_cx = global::get_text_map_propagator(|propagator| {
                propagator.extract(&HeaderExtractor(req.headers()))
            });
            tracer.start_with_context("fibonacci", &parent_cx);

            // Concatenate the body...
            let b = hyper::body::to_bytes(req).await?;
            // Parse the request body. form_urlencoded::parse
            // always succeeds, but in general parsing may
            // fail (for example, an invalid post of json), so
            // returning early with BadRequest may be
            // necessary.
            //
            // Warning: this is a simplified use case. In
            // principle names can appear multiple times in a
            // form, and the values should be rolled up into a
            // HashMap<String, Vec<String>>. However in this
            // example the simpler approach is sufficient.
            let params = form_urlencoded::parse(b.as_ref())
                .into_owned()
                .collect::<HashMap<String, String>>();

            // Validate the request parameters, returning
            // early if an invalid input is detected.
            let name = if let Some(n) = params.get("name") {
                n
            } else {
                return Ok(Response::builder()
                    .status(StatusCode::UNPROCESSABLE_ENTITY)
                    .body(MISSING.into())
                    .unwrap());
            };
            let number = if let Some(n) = params.get("number") {
                if let Ok(v) = n.parse::<u8>() {
                    v
                } else {
                    return Ok(Response::builder()
                        .status(StatusCode::UNPROCESSABLE_ENTITY)
                        .body(NOTNUMERIC.into())
                        .unwrap());
                }
            } else {
                return Ok(Response::builder()
                    .status(StatusCode::UNPROCESSABLE_ENTITY)
                    .body(MISSING.into())
                    .unwrap());
            };

            let nth_fib = fibonacci(number);

            // Render the response. This will often involve
            // calls to a database or web service, which will
            // require creating a new stream for the response
            // body. Since those may fail, other error
            // responses such as InternalServiceError may be
            // needed here, too.

            let body = format!(
                "Hello {}, {}th fibonacci number is {}",
                name, number, nth_fib
            );
            Ok(Response::new(body.into()))
        }
        (&Method::GET, "/get") => {
            let query = if let Some(q) = req.uri().query() {
                q
            } else {
                return Ok(Response::builder()
                    .status(StatusCode::UNPROCESSABLE_ENTITY)
                    .body(MISSING.into())
                    .unwrap());
            };
            let params = form_urlencoded::parse(query.as_bytes())
                .into_owned()
                .collect::<HashMap<String, String>>();
            let page = if let Some(p) = params.get("page") {
                p
            } else {
                return Ok(Response::builder()
                    .status(StatusCode::UNPROCESSABLE_ENTITY)
                    .body(MISSING.into())
                    .unwrap());
            };
            let body = format!("You requested {}", page);
            Ok(Response::new(body.into()))
        }
        _ => Ok(Response::builder()
            .status(StatusCode::NOT_FOUND)
            .body(Body::empty())
            .unwrap()),
    }
}

fn init_tracer() -> Result<sdktrace::Tracer, TraceError> {
    opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(opentelemetry_otlp::new_exporter().tonic().with_env())
        .with_trace_config(
            sdktrace::config().with_resource(Resource::default()),
        )
        .install_batch(opentelemetry::runtime::Tokio)
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error + Send + Sync + 'static>> {
    let _ = init_tracer()?;

    let addr = ([127, 0, 0, 1], 1337).into();

    let server = Server::bind(&addr).serve(make_service_fn(|_| async {
        Ok::<_, hyper::Error>(service_fn(handle))
    }));

    println!("Listening on {}", addr);
    if let Err(e) = server.await {
        eprintln!("server error: {}", e);
    }

    shutdown_tracer_provider();

    Ok(())
}
```

另一个分布式追踪例子：

```rust
// https://docs.dynatrace.com/docs/extend-dynatrace/opentelemetry/walkthroughs/rust

use std::collections::HashMap;
use std::io::{BufRead, BufReader, Read};

use opentelemetry::{
    global,
    trace::{Span, TraceContextExt, TraceError, Tracer},
    Context, KeyValue,
};

use opentelemetry_sdk::{runtime, trace as sdktrace, Resource};
use opentelemetry_http::{HeaderExtractor, HeaderInjector};
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_semantic_conventions as semcov;

fn init_opentelemetry() -> Result<sdktrace::Tracer, TraceError>  {
    // Helper function to read potentially available OneAgent data
    fn read_dt_metadata() -> Resource {
        fn read_single(path: &str, metadata: &mut Vec<KeyValue>) -> std::io::Result<()> {
            let mut file = std::fs::File::open(path)?;
            if path.starts_with("dt_metadata") {
                let mut name = String::new();
                file.read_to_string(&mut name)?;
                file = std::fs::File::open(name)?;
            }
            for line in BufReader::new(file).lines() {
                if let Some((k, v)) = line?.split_once('=') {
                    metadata.push(KeyValue::new(k.to_string(), v.to_string()))
                }
            }
            Ok(())
        }
        let mut metadata = Vec::new();
        for name in [
            "dt_metadata_e617c525669e072eebe3d0f08212e8f2.properties",
            "/var/lib/dynatrace/enrichment/dt_metadata.properties",
            "/var/lib/dynatrace/enrichment/dt_host_metadata.properties"
        ] {
            let _ = read_single(name, &mut metadata);
        }
        Resource::new(metadata)
    }

    // ===== GENERAL SETUP =====
    let DT_API_TOKEN = env::var("DT_API_TOKEN").unwrap(); // TODO: change
    let DT_API_URL = env::var("DT_API_URL").unwrap();

    let mut map = HashMap::new();
    map.insert("Authorization".to_string(), format!("Api-Token {}", DT_API_TOKEN));
    let mut resource = Resource::new([
    KeyValue::new(semcov::resource::SERVICE_NAME, "rust-app") //TODO Replace with the name of your application
    ]);
    resource = resource.merge(&read_dt_metadata());

    // ===== TRACING SETUP =====
    global::set_text_map_propagator(TraceContextPropagator::new());

    opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .http()
                .with_endpoint(DT_API_URL)
                .with_headers(map),
        )
        .with_trace_config(
            sdktrace::config()
                .with_resource(resource)
                .with_sampler(sdktrace::Sampler::AlwaysOn),
        )
        .install_batch(runtime::Tokio) // 设置 opentelemetry global 的 trace provider
}

init_opentelemetry()

// 后续，在需要的位置，从 global 获取命名的 tracer
let tracer = global::tracer("my-tracer");

// 从 tracer 创建 span
let mut span = tracer.start("Call to /myendpoint");
span.set_attribute(KeyValue::new("http.method", "GET"));
span.set_attribute(KeyValue::new("net.protocol.version", "1.1"));
// TODO: Your code goes here
span.end();


// Ensure context propagation optional
// Context propagation is particularly important when network calls (for example, REST) are involved.

//Method to extract the parent context from the request
fn get_parent_context(req: Request<Body>) -> Context {
    global::get_text_map_propagator(|propagator| {
        propagator.extract(&HeaderExtractor(req.headers()))
    })
}

async fn incoming_request(req: Request<Body>) -> Result<Response<Body>, Infallible> {
    let parent_cx = get_parent_context(req);

    let mut span = global::tracer("manual-server")
        .start_with_context("my-server-span", &parent_cx); //TODO Replace with the name of your span

    span.set_attribute(KeyValue::new("my-server-key-1", "my-server-value-1")); //TODO Add attributes

    // TODO Your incoming_request code goes here
}

// 向发出的 hyper 请求注入 context
async fn outgoing_request(
    context: Context,
) -> std::result::Result<(), Box<dyn std::error::Error + Send + Sync + 'static>> {
    let client = Client::new();
    let span = global::tracer("manual-client")
        .start_with_context("my-client-span", &context); //TODO Replace with the name of your span
    let cx = Context::current_with_span(span);

    let mut req = hyper::Request::builder().uri("<HTTP_URL>");

    //Method to inject the current context in the request
    global::get_text_map_propagator(|propagator| {
        propagator.inject_context(&cx, &mut HeaderInjector(&mut req.headers_mut().unwrap()))
    });

    cx.span()
        .set_attribute(KeyValue::new("my-client-key-1", "my-client-value-1")); //TODO Add attributes

    // TODO Your outgoing_request code goes here
}
```

aliyun sls 的例子：

```rust
// https://www.alibabacloud.com/help/en/sls/user-guide/import-trace-data-from-rust-applications-to-log-service-by-using-opentelemetry-sdk-for-rust
use opentelemetry::global::shutdown_tracer_provider;
use opentelemetry::sdk::Resource;
use opentelemetry::trace::TraceError;
use opentelemetry::{
    baggage::BaggageExt,
    trace::{TraceContextExt, Tracer},
    Context, Key, KeyValue,
};
use opentelemetry::{global, sdk::trace as sdktrace};
use opentelemetry_otlp::WithExportConfig;
use std::error::Error;
use std::time::Duration;
use tonic::metadata::MetadataMap;
use tonic::transport::ClientTlsConfig;
use url::Url;
static ENDPOINT: &str = "https://${endpoint}";
static PROJECT: &str = "${project}";
static INSTANCE_ID: &str = "${instance}";
static AK_ID: &str = "${access-key-id}";
static AK_SECRET: &str = "${access-key-secret}";
static SERVICE_VERSION: &str = "${version}";
static SERVICE_NAME: &str = "${service}";
static SERVICE_NAMESPACE: &str = "${service.namespace}";
static HOST_NAME: &str = "${host}";

static SLS_PROJECT_HEADER: &str = "x-sls-otel-project";
static SLS_INSTANCE_ID_HEADER: &str = "x-sls-otel-instance-id";
static SLS_AK_ID_HEADER: &str = "x-sls-otel-ak-id";
static SLS_AK_SECRET_HEADER: &str = "x-sls-otel-ak-secret";
static SLS_SERVICE_VERSION: &str = "service.version";
static SLS_SERVICE_NAME: &str = "service.name";
static SLS_SERVICE_NAMESPACE: &str = "service.namespace";
static SLS_HOST_NAME: &str = "host.name";

fn init_tracer() -> Result<sdktrace::Tracer, TraceError> {
    let mut metadata_map = MetadataMap::with_capacity(4);
    metadata_map.insert(SLS_PROJECT_HEADER, PROJECT.parse().unwrap());
    metadata_map.insert(SLS_INSTANCE_ID_HEADER, INSTANCE_ID.parse().unwrap());
    metadata_map.insert(SLS_AK_ID_HEADER, AK_ID.parse().unwrap());
    metadata_map.insert(SLS_AK_SECRET_HEADER, AK_SECRET.parse().unwrap());

    let endpoint = ENDPOINT;
    let endpoint = Url::parse(&endpoint).expect("endpoint is not a valid url");
    let resource = vec![
        KeyValue::new(SLS_SERVICE_VERSION, SERVICE_VERSION),
        KeyValue::new(SLS_HOST_NAME, HOST_NAME),
        KeyValue::new(SLS_SERVICE_NAMESPACE, SERVICE_NAMESPACE),
        KeyValue::new(SLS_SERVICE_NAME, SERVICE_NAME),
    ];

    opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .tonic()
                .with_endpoint(endpoint.as_str())
                .with_metadata(dbg!(metadata_map))
                .with_tls_config(
                    ClientTlsConfig::new().domain_name(
                        endpoint
                            .host_str()
                            .expect("the specified endpoint should have a valid host"),
                    ),
                ),
        )
        .with_trace_config(sdktrace::config().with_resource(Resource::new(resource)))
        .install_batch(opentelemetry::runtime::Tokio)
}

const FOO_KEY: Key = Key::from_static_str("ex.com/foo");
const BAR_KEY: Key = Key::from_static_str("ex.com/bar");
const LEMONS_KEY: Key = Key::from_static_str("lemons");
const ANOTHER_KEY: Key = Key::from_static_str("ex.com/another");

lazy_static::lazy_static! {
    static ref COMMON_ATTRIBUTES: [KeyValue; 4] = [
        LEMONS_KEY.i64(10),
        KeyValue::new("A", "1"),
        KeyValue::new("B", "2"),
        KeyValue::new("C", "3"),
    ];
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error + Send + Sync + 'static>> {
    let _ = init_tracer()?;
    let tracer = global::tracer("ex.com/basic");
    let _baggage =
        Context::current_with_baggage(vec![FOO_KEY.string("foo1"), BAR_KEY.string("bar1")])
            .attach();

    tracer.in_span("operation", cx {
        let span = cx.span();
        span.add_event(
            "Nice operation!".to_string(),
            vec![Key::new("bogons").i64(100)],
        );
        span.set_attribute(ANOTHER_KEY.string("yes"));

        tracer.in_span("Sub operation...", cx {
            let span = cx.span();
            span.set_attribute(LEMONS_KEY.string("five"));

            span.add_event("Sub span event".to_string(), vec![]);
        });
    });

    tokio::time::sleep(Duration::from_secs(60)).await;
    shutdown_tracer_provider();
    Ok(())
}
```


## <span class="section-num">8</span> tracing-appender {#tracing-appender}

该 crate 提供了终端和文件输出的 writer，同时文件支持轮转，需要和
tracing_subscriber::fmt().with_writer(xxx) 来配合使用。

-   按时间周期，按文件大小轮转；
-   tracing_appender::non_blocking() 非阻塞模式，会创建一个特定的 worker thread 来接受 log line 并写入
    writer。log line 先被 enqueue，然后在被写入。

<!--listend-->

```rust
// File Appender
// file_appender 实现了 std::io::Write
let file_appender = tracing_appender::rolling::hourly("/some/directory", "prefix.log");
tracing_subscriber::fmt()
    .with_writer(file_appender)
    .init();

// Non-Blocking Writer
// 1. stdout
let (non_blocking, _guard) = tracing_appender::non_blocking(std::io::stdout());
tracing_subscriber::fmt()
    .with_writer(non_blocking)
    .init();

// 2. 自定义 writer
use std::io::Error;
struct TestWriter;
impl std::io::Write for TestWriter {
    fn write(&mut self, buf: &[u8]) -> std::io::Result<usize> {
        let buf_len = buf.len();
        println!("{:?}", buf);
        Ok(buf_len)
    }

    fn flush(&mut self) -> std::io::Result<()> {
        Ok(())
    }
}
let (non_blocking, _guard) = tracing_appender::non_blocking(TestWriter);
tracing_subscriber::fmt()
    .with_writer(non_blocking)
    .init();

// Non-Blocking Rolling File Appender
let file_appender = tracing_appender::rolling::hourly("/some/directory", "prefix.log");
let (non_blocking, _guard) = tracing_appender::non_blocking(file_appender);
tracing_subscriber::fmt()
    .with_writer(non_blocking)
    .init();
```


## <span class="section-num">9</span> tracing-flame {#tracing-flame}

tracing 数据也可以生成火焰图。

先在项目中添加 tracing-frame crate：cargo add tracing-flame

tracing_flame::FlameLayer() 提供了一个 Layer 实现，可用于配置 Registry 来作为全局 Subscribe：

```rust
use std::{fs::File, io::BufWriter};
use tracing_flame::FlameLayer;
use tracing_subscriber::{registry::Registry, prelude::*, fmt};

fn main()  {
    let fmt_layer = fmt::Layer::default();

    let (flame_layer, _guard) = FlameLayer::with_file("./tracing.folded").unwrap();

    tracing_subscriber::registry()
    .with(fmt::layer())
    .with(flame_layer)
    .init();
    // ... the rest of your code
}
```

tracing 数据会被保存到 ./tracing.folded 文件，然后使用 inferno crate 来转换成 flamegraph：

```shell
cargo install inferno

# flamegraph
cat tracing.folded | inferno-flamegraph > tracing-flamegraph.svg
# flamechart
cat tracing.folded | inferno-flamegraph --flamechart > tracing-flamechart.svg
```


## <span class="section-num">10</span> tracing console {#tracing-console}

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


## <span class="section-num">11</span> 例子 {#例子}

<https://www.shuttle.rs/blog/2024/01/09/getting-started-tracing-rust>

<https://tokio.rs/tokio/topics/tracing>

<https://burgers.io/custom-logging-in-rust-using-tracing>

<https://signoz.io/blog/opentelemetry-rust/>

<https://docs.dynatrace.com/docs/extend-dynatrace/opentelemetry/walkthroughs/rust>

<https://www.alibabacloud.com/help/en/sls/user-guide/import-trace-data-from-rust-applications-to-log-service-by-using-opentelemetry-sdk-for-rust>

```rust
#![allow(dead_code)]

#[tracing::instrument]
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
