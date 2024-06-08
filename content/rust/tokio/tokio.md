---
title: "tokio"
author: ["zhangjun"]
lastmod: 2024-06-08T21:57:41+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 1
---

Tokio provides multiple variations of the runtime. Everything from `a multi-threaded, work-stealing
runtime` to a light-weight, single-threaded runtime


## <span class="section-num">1</span> features {#features}

最简单的是使用 full feature：

```toml
tokio = { version = "1", features = ["full"] }

# 或单独启用 features
tokio = { version = "1", features = ["rt", "net"] }
```

full
: Enables all features listed below except test-util and tracing.

rt
: Enables tokio::spawn, the `current-thread scheduler`, and non-scheduler utilities.

rt-multi-thread
: Enables the heavier, `multi-threaded, work-stealing scheduler`.

io-util
: Enables the IO based `Ext traits`.

io-std
: Enable Stdout, Stdin and Stderr types.

net: Enables tokio::net types such as TcpStream, UnixStream and UdpSocket, as well as (on
    Unix-like systems) AsyncFd and (on FreeBSD) PollAio.

time: Enables tokio::time types and allows the schedulers to enable the built in timer.

process: Enables tokio::process types.

macros
: Enables `#[tokio::main] and #[tokio::test]` macros.

sync: Enables all tokio::sync types.

signal: Enables all tokio::signal types.

fs: Enables tokio::fs types.

test-util: Enables testing based infrastructure for the Tokio runtime.

parking_lot: As a potential optimization, use the <span class="underline">parking_lot</span> crate’s synchronization primitives
    internally. Also, this dependency is necessary to construct some of our primitives in a const
    context. MSRV may increase according to the <span class="underline">parking_lot</span> release in use.

还有一些 unstable features，需要单独启用：

-   tracing: Enables tracing events.
-   task::Builder 注意不是 runtime::Builder
-   Some methods on task::JoinSet
-   runtime::RuntimeMetrics
-   runtime::Builder::unhandled_panic
-   task::Id

<!--listend-->

```toml
[build]
rustflags = ["--cfg", "tokio_unstable"]
```


## <span class="section-num">2</span> macros {#macros}

`#[main]`: 用于设置一个 Runtime，如果要设置复杂的 Runtime 参数可以使用 Builder;

-   可以用于任何 async fn，但一般是 async main fn，否则每次调用该 async fn 时都新建一个 Runtime 来运行。
-   默认是 Multi-threaded runtime，为每个 CPU Core 创建一个 thread 的 `worker thread pool` 来调度执行
    spawn() 产生的 Future, 支持 work-stealing strategy.
    -   对于 spawn_blocking() 传入的同步函数/闭包，也是立即在一个新的 thread 中执行，这些 thread 形成另一个 `blocking thread pool` 。当这些 thread 空闲一段时间后，可能会被 drop 回收。

<!--listend-->

```rust
// Multi-threaded runtime
#[tokio::main(flavor = "multi_thread", worker_threads = 10)]
#[tokio::main(worker_threads = 2)]

// Current thread runtime
#[tokio::main(flavor = "current_thread")]
#[tokio::main(flavor = "current_thread", start_paused = true)]

// 示例
#[tokio::main]
async fn main() {
    println!("Hello world");
}
// 等效为
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            println!("Hello world");
        })
}
```

`join!()` 并发执行传入的所有 async fn，直到他们都完成：

1.  join!() 必须在 async context 中运行，比如 async fn/block/closure；
2.  tokio 使用一个 `单线程` 来执行这些 async task，所以如果一个 task 可能 blocking 其他 task 的执行；如果要真正并发执行，需要使用 spawn();
3.  如果 async task 都返回 Result，join!() 也是等待他们都完成，如果要在遇到第一个 Error 时停止执行，可以使用 `try_join!()`

<!--listend-->

```rust
async fn do_stuff_async() {
    // async work
}
async fn more_async_work() {
    // more here
}

#[tokio::main]
async fn main() {
    let (first, second) = tokio::join!(
        do_stuff_async(),
        more_async_work());
    // do something with the values
}
```

`pin!()` : 拥有传入的 async task 的 Future 对象，返回一个 Pin，可以确保该 Future 对象的栈内存不发生移动：

```rust
use tokio::{pin, select};
use tokio_stream::{self as stream, StreamExt};

async fn my_async_fn() {
    // async logic here
}

#[tokio::main]
async fn main() {
    let mut stream = stream::iter(vec![1, 2, 3, 4]);

    let future = my_async_fn();
    pin!(future);

    loop {
        select! {
            _ = &mut future => {
                // Stop looping `future` will be polled after completion
                break;
            }
            Some(val) = stream.next() => {
                println!("got value = {}", val);
            }
        }
    }
}

// 同时创建多一个 Future Pin
use tokio::{pin, select};
async fn my_async_fn() {
    // async logic here
}
#[tokio::main]
async fn main() {
    pin! {
        let future1 = my_async_fn();
        let future2 = my_async_fn();
    }

    select! {
        _ = &mut future1 => {}
        _ = &mut future2 => {}
    }
}
```

`select!()` : 并发执行多个 async expression，对于第一个匹配 pattern 的 branch，执行对应的 handler，同时 drop 未返回的其他 branch 对象：

```text
<pattern> = <async expression> (, if <precondition>)? => <handler>,
```

```rust
async fn do_stuff_async() {
    // async work
}

async fn more_async_work() {
    // more here
}

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = do_stuff_async() => {
            println!("do_stuff_async() completed first")
        }
        _ = more_async_work() => {
            println!("more_async_work() completed first")
        }
    };
}
```

`task_local!()`: 对于传给 spwan() 的 async fn，必须实现 Send + Sync + 'static, 它有可能被调度到不同的线程上运行，所以不能使用 thread_local 变量，而是使用 task local 变量。该宏生成一个
tokio::task::LocalKey 使用的 local key：

```rust
task_local! {
    pub static ONE: u32;

    #[allow(unused)]
    static TWO: f32;
}

// 使用示例：
tokio::task_local! {
    static NUMBER: u32;
}

NUMBER.scope(1, async move {
    assert_eq!(NUMBER.get(), 1);
}).await;

NUMBER.scope(2, async move {
    assert_eq!(NUMBER.get(), 2);

    NUMBER.scope(3, async move {
        assert_eq!(NUMBER.get(), 3);
    }).await;
}).await;
```


## <span class="section-num">3</span> runtime {#runtime}

tokio Runtime 包含三部分：

1.  An I/O event loop, called the driver, which drives I/O resources and dispatches I/O events to
    tasks that depend on them.
2.  A scheduler to execute tasks that use these I/O resources.
3.  A timer for scheduling work to run after a set period of time.

一般不需要手动创建 Runtime，而是使用 #[tokio::main] 属性来标识 async fn main() 函数。但是如果要精细控制 Runtime 的参数，可以使用 tokio::runtime::Builder:

-   在 Runtime 上下文中，可以使用 tokio::spawn()/spawn_local() 来执行异步任务，而使用 spwan_blocking()
    来创建一个临时 thread 执行同步任务；

<!--listend-->

```rust
// 示例
#[tokio::main]
async fn main() {
    println!("Hello world");
}
// 等效为
fn main() {
    tokio::runtime::Builder::new_multi_thread() // 多线程模式
        .enable_all() // 启用所有 resource driver
        .build()
        .unwrap()
        .block_on(async {
            println!("Hello world");
        })
}

// 另一个例子，手动创建 Runtime
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::runtime::Runtime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the runtime
    let rt  = Runtime::new()?;

    // Spawn the root task
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await?;

        loop {
            let (mut socket, _) = listener.accept().await?;

            tokio::spawn(async move {
                let mut buf = [0; 1024];

                // In a loop, read data from the socket and write the data back.
                loop {
                    let n = match socket.read(&mut buf).await {
                        // socket closed
                        Ok(n) if n == 0 => return,
                        Ok(n) => n,
                        Err(e) => {
                            println!("failed to read from socket; err = {:?}", e);
                            return;
                        }
                    };

                    // Write the data back
                    if let Err(e) = socket.write_all(&buf[0..n]).await {
                        println!("failed to write to socket; err = {:?}", e);
                        return;
                    }
                }
            });
        }
    })
}
```

tokio Runtime 有两种类型（默认使用的是多线程版本）：

-   Multi-Thread Scheduler：使用每个 CPU 一个 thread 的 worker thread pool, using a work-stealing
    strategy 来执行 spawn() 提交的 Future task；
-   Current-Thread Scheduler：provides `a single-threaded` future executor. All tasks will be created
    and executed on `the current thread`

tokio Runtime 除了管理 Scheduler 以外，还管理各种 IO/Timer 等 resource driver，需要单独启用：

-   `Builder::enable_io()`
-   `Builder::enable_time`
-   As a shorthand, `Builder::enable_all()`

Runtime 在调度任务的间隙，周期检查这些 IO/Timer 是否 Ready，默认当没有任务可以调度，或者调度的任务数超过 61 时检查 IO/Timer，这个次数可以使用 event_interval 来设置。

对于 worker thread pool 是在 Runtime 创建时为每个 CPU 创建一个 thread，tokio 还有一个 blocking
thread pool，是在调用 spawn_blocking() 时临时创建的线程，这个 pool 中的线程数量不固定，而且 idle 一段时间后会自动被 tokio Runtime 清理。当 Runtime 被 Drop 时，所有的 thread 都会被终止（terminated），但是 unstoppable spawned work are not guaranteed to have been terminated。

---

tokio Runtime 确保所有 task 都是公平调度，防止个别 task 一直可以调度的情况下，其他 task 得不到运行。

-   There is some number MAX_TASKS such that the total number of tasks on the runtime at any specific
    point in time never exceeds MAX_TASKS.
-   There is some number MAX_SCHEDULE such that calling poll on any task spawned on the runtime
    returns within MAX_SCHEDULE time units.
-   Then, there is some number MAX_DELAY such that when a task is woken, it will be scheduled by the
    runtime within MAX_DELAY time units.

(Here, MAX_TASKS and MAX_SCHEDULE can be any number and the user of the runtime may choose them. The
MAX_DELAY number is controlled by the runtime, and depends on the value of MAX_TASKS and
MAX_SCHEDULE.) Other than the above fairness guarantee, there is no guarantee about the order in
which tasks are scheduled.

除了调度 task，runtime 还周期检查 IO/timer event，来唤醒对应的 task：

Beyond just scheduling tasks, the runtime must also `manage IO resources and timers`. It does this by
`periodically checking` whether there are any IO resources or timers that are ready, and waking the
relevant task so that it will be scheduled.

tokio 有两种 Runtime：

1.  The `multi-thread scheduler` executes futures on a thread pool, using a work-stealing strategy. By
    default, it will start a worker thread for each CPU core available on the system.
2.  The `current-thread scheduler` provides a single-threaded future executor. All tasks will be
    created and executed on the current thread.

这两个 Runtime 都有两个 queue：The current thread runtime maintains two FIFO queues of tasks that
are ready to be scheduled: `the global queue and the local queue` 。

-   先从 local queue 获取 task，如果为空，再从 global queue 获取任务。或者从 local queu 获取
    `global_queue_interval` （默认 31）个任务后，从 global queue 获取任务；
-   当没有任务可以调度，或者调度任务超过 `event_interval` 次后（默认 61）后，检查 IO 或 timer event；
-   current-thread scheduler 默认启用 lifo slot optimazition，当有新 task 被 wake 时，添加到 local
    queue；

对于 multi-thread scheduler，有一个固定 thread 数量的 thread pool，它在创建 Runtime 时创建。有一个
global queue，和每个 thread 一个的 local queue，local queue 初始容量是 256 个 tasks，超过的会被移动到 global queue。默认先从 local queue 获取 task，然后是 global queue，如果都为空，则从其他 thread 的
local queue `steal tasks` 。

The multi thread runtime `uses the lifo slot optimization`: Whenever a task wakes up another task, the
other task is added to the `worker thread’s lifo slot` instead of being added to a queue.  If there
was already a task in the lifo slot when this happened, then the lifo slot `is replaced`, and the task
that used to be in the lifo slot is placed in the thread’s `local queue`. When the runtime finishes
scheduling a task, it will `schedule the task in the lifo slot immediately`, if any。When the lifo
slot is used, `the coop budget is not reset`. Furthermore, if a worker thread uses the lifo slot three
times in a row, it is `temporarily disabled` until the worker thread has scheduled a task that didn’t
come from the lifo slot. The lifo slot can be disabled using the `disable_lifo_slot setting`.

The lifo slot is separate from the local queue, so other worker threads `cannot steal the task` in the
lifo slot. When a task is woken from a thread that is not a worker thread, then the task is placed
in the global queue.


### <span class="section-num">3.1</span> Runtime {#runtime}

The runtime provides an I/O driver, task scheduler, timer, and blocking pool, necessary for running
asynchronous tasks.

Shutting down the runtime is done by `dropping` the value, or calling `shutdown_background or
shutdown_timeout`.

对于 Drop runtime，默认是 `wait forver` 直到所有 spawned work has been stopped：

1.  Tasks spawned through `Runtime::spawn` keep running until they `yield. Then they are dropped`. They
    are not guaranteed to run to completion, but might do so if they do not yield until completion.
    -   只有 yield 时（例如会被 blocking 时下一次 `.await 位置` ）才会被 dropped，否则就一直运行直到结束；
2.  Blocking functions spawned through Runtime::spawn_blocking `keep running until they return`.
    -   一直运行直到结束；

通过使用 Runtime 的 shutdown_background and shutdown_timeout 方法，可以 `避免这种 waiting forerver` 的等待。

-   当 timeout 时如果 task 没有结束，则运行他的 thread 会被泄露（leak），task 会一直运行直到结束；

Runtime 方法：

-   new() 方法创建一个多线程的调度器 Runtime，启用所有默认 resource driver，并自动调用 enter()。一般使用 #[tokio::main] 来自动调用。
-   blocl_on() 在 Runtime 中执行传入Future task，只能在同步上下文中调用该方法，他是同步代码和异步代码的结合点，如果在异步上下文中执行则会 panic。
    -   在当前线程中执行 Future，block 当前线程直到 Future 返回。如果要并行执行 task，则需要在 Future 内部调用 spawn() 来提交 task；
-   spawn() 和 spawn_blocking() 返回的 JoinHandle 对象实现了 Future，可以 .await 获得结果；
-   enter() 会设置 thread local Runtime，后续 tokio::spawn() 等会感知该 Runtime。一般用于手动创建
    Runtime 的场景。
-   handler() 返回一个可以在该 Runtime 上 spawn() task 的 Handler 对象，它基于引用计数实现了 Clone()
    方法，可以将 Runtime 在多个 thread 间共享。

<!--listend-->

```rust
impl Runtime

// Available on crate feature rt-multi-thread only. 创建一个多线程的调度器 Runtime，启用所有默认
// resource driver，并自动调用 enter()。一般使用 #[tokio::main] 来自动调用。
pub fn new() -> Result<Runtime>

pub fn handle(&self) -> &Handle // 返回一个实现了引用计数的 Handle，可以 spawn() 任务

// 在 worker thread pool 中立即运行 task，而不管是否 await 返回的 JoinHandle
pub fn spawn<F>(&self, future: F) -> JoinHandle<F::Output>  // 提交异步任务 Future
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,

pub fn spawn_blocking<F, R>(&self, func: F) -> JoinHandle<R> // 提交同步任务
where
    F: FnOnce() -> R + Send + 'static,
    R: Send + 'static,

// 执行 future，直到结束。对于多线程 Runtime，当 block_on 返回时，spawn() 的 task 可能还在运行，如果
// 要确保这些 task完成，可以进行 .await.
pub fn block_on<F: Future>(&self, future: F) -> F::Output

pub fn enter(&self) -> EnterGuard<'_> // 设置 thread local Runtime，后续可以使用 tokio::spawn() 等函数

pub fn shutdown_timeout(self, duration: Duration) // 关闭 Runtime，不能再提交任务和 IO/Timer
pub fn shutdown_background(self)

impl Runtime
// Available on tokio_unstable only.
pub fn metrics(&self) -> RuntimeMetrics
```

可以使用 Runtime::enter() 或 Handle::enter() 进入 Runtime context，他会创建一个 thread local 变量来标识当前的 Runtime，然后就可以使用 tokio::spawn() 等方法在该 Runtime context 中提交异步任务了，一般用于手动创建 Runtime 的场景：

```rust
use tokio::runtime::Runtime;
use tokio::task::JoinHandle;

fn function_that_spawns(msg: String) -> JoinHandle<()> {
    // Had we not used `rt.enter` below, this would panic.
    tokio::spawn(async move {
        println!("{}", msg);
    })
}

fn main() {
    let rt = Runtime::new().unwrap();

    let s = "Hello World!".to_string();

    // By entering the context, we tie `tokio::spawn` to this executor.
    let _guard = rt.enter();
    let handle = function_that_spawns(s);

    // Wait for the task before we end the test.
    rt.block_on(handle).unwrap();
}
```

runtime.shutdown_timeout() 和 shutdown_background() 关闭 Runtime，而不等待所有 spawn 的 task 结束：

-   Drop Runtime 时默认会等待所有 task 结束，则可能导致无限期等待；
-   这两个方法不会无限期等待，task 还会在后台运行，对应的 thread 可能会被泄露；

<!--listend-->

```rust
use tokio::runtime::Runtime;
use tokio::task;

use std::thread;
use std::time::Duration;

fn main() {
    let runtime = Runtime::new().unwrap();

    runtime.block_on(async move {
        task::spawn_blocking(move || {
            thread::sleep(Duration::from_secs(10_000));
        });
    });

    runtime.shutdown_timeout(Duration::from_millis(100));
}
```

Runtime 对象本身可以通过多方方式进行 Sharing：

1.  Using an Arc&lt;Runtime&gt;.
2.  Using a Handle.
3.  Entering the runtime context.

Arc&lt;Runtime&gt; 和 Handle（可以使用 Runtime.handle() 方法返回 ） 都基于引用计数实现了 Clone() 方法，可以将 Runtime 在多个 thread 间共享。

-   Handler::current() 返回当前 Runtime 的 Handle，可以 spawn() 并发异步任务；

<!--listend-->

```rust
use tokio::runtime::Handle;

#[tokio::main]
async fn main () {
    let handle = Handle::current();

    std::thread::spawn(move || {
        // Using Handle::block_on to run async code in the new thread.
        handle.block_on(async {
            println!("hello");
        });
    });
}
```


### <span class="section-num">3.2</span> Builder {#builder}

Builds Tokio Runtime with `custom configuration values`.

```rust
use tokio::runtime::Builder;

fn main() {
    // build runtime
    let runtime = Builder::new_multi_thread()
        .worker_threads(4)
        .thread_name("my-custom-name")
        .thread_stack_size(3 * 1024 * 1024)
        .build()
        .unwrap();

    // use runtime ...
}
```

Builder 方法：

-   new_current_thread() ： 单线程调度器；
-   new_multi_thread() : 多线程调度器

<!--listend-->

```rust
impl Builder
pub fn new_current_thread() -> Builder
// Available on crate feature rt-multi-thread only.
pub fn new_multi_thread() -> Builder
// Available on tokio_unstable and crate feature rt-multi-thread only.
pub fn new_multi_thread_alt() -> Builder

// 启用所有 resource，如 io/net/timer 等
pub fn enable_all(&mut self) -> &mut Self

pub fn worker_threads(&mut self, val: usize) -> &mut Self
pub fn max_blocking_threads(&mut self, val: usize) -> &mut Self
pub fn thread_name(&mut self, val: impl Into<String>) -> &mut Self
pub fn thread_name_fn<F>(&mut self, f: F) -> &mut Self
where
    F: Fn() -> String + Send + Sync + 'static,

pub fn thread_stack_size(&mut self, val: usize) -> &mut Self

pub fn on_thread_start<F>(&mut self, f: F) -> &mut Self
where
    F: Fn() + Send + Sync + 'static,
pub fn on_thread_stop<F>(&mut self, f: F) -> &mut Self
where
    F: Fn() + Send + Sync + 'static,
pub fn on_thread_park<F>(&mut self, f: F) -> &mut Self
where
    F: Fn() + Send + Sync + 'static,
pub fn on_thread_unpark<F>(&mut self, f: F) -> &mut Self
where
    F: Fn() + Send + Sync + 'static,

// 构建生成 Runtime
pub fn build(&mut self) -> Result<Runtime>
// blocking 线程池中线程空闲时间
pub fn thread_keep_alive(&mut self, duration: Duration) -> &mut Self
// 检查 global queue event 的数量
pub fn global_queue_interval(&mut self, val: u32) -> &mut Self
// 检查 event 的调度事件数量
pub fn event_interval(&mut self, val: u32) -> &mut Self

// Available on tokio_unstable only.
pub fn unhandled_panic(&mut self, behavior: UnhandledPanic) -> &mut Self
// Available on tokio_unstable only.
pub fn disable_lifo_slot(&mut self) -> &mut Self
// Available on tokio_unstable only.
pub fn rng_seed(&mut self, seed: RngSeed) -> &mut Self
// Available on tokio_unstable only.
pub fn enable_metrics_poll_count_histogram(&mut self) -> &mut Self

pub fn metrics_poll_count_histogram_scale(
    &mut self,
    histogram_scale: HistogramScale
) -> &mut Self
pub fn metrics_poll_count_histogram_resolution(
    &mut self,
    resolution: Duration
) -> &mut Self
pub fn metrics_poll_count_histogram_buckets(
    &mut self,
    buckets: usize
) -> &mut Self

impl Builder
// Available on crate feature net, or Unix and crate feature process, or Unix and crate feature signal only.
pub fn enable_io(&mut self) -> &mut Self
// Available on crate feature net, or Unix and crate feature process, or Unix and crate feature signal only.
pub fn max_io_events_per_tick(&mut self, capacity: usize) -> &mut Self

impl Builder
// Available on crate feature time only.
pub fn enable_time(&mut self) -> &mut Self

impl Builder
pub fn start_paused(&mut self, start_paused: bool) -> &mut Self
```


## <span class="section-num">4</span> task {#task}

Asynchronous green-threads.

A task is `a light weight, non-blocking unit of execution`. A task is similar to an OS thread, but
rather than being managed by the OS scheduler, they are managed by the `Tokio runtime`. Another name
for this general pattern is `green threads`. If you are familiar with Go’s goroutines, Kotlin’s
coroutines, or Erlang’s processes, you can think of Tokio’s tasks as something similar.

三大特点：

1.  轻量级：scheduled by the Tokio runtime rather than the operating system；
2.  协作式调度：OS thread 一般是抢占式调度；协作式调度 cooperatively 表示 task 会一直运行直到
    yield（例如数据没有准备好，.await 位置会 yield，同时 tokio API lib 中默认强制插入了一些 yield
    point，确保即使 task 没有 yield，底层的 lib 也会周期 yield），这时 Tokio Runtime 会调度其他 task
    来运行；
3.  非阻塞：需要使用 tokio crate 提供的非阻塞 IO 来进行 IO/Net/Fs 等到 async context 的 APIs 操作，这些 API 不会阻塞线程，而是 yield，这样 tokio Runtime 可以执行其他 task；

async task 可以使用 aysnc fn 或 async block 来构造，他们都是返回实现 Funture trait 的语法糖。

-   async move block 更常用；

<!--listend-->

```rust
use tokio::task;

// 需要位于 Runtime Context， spawn 的 task 立即在后台执行，而不管是否 .await
let join = task::spawn(async {
    // ...
    "hello world!"
});

// ...

// Await the result of the spawned task.
let result = join.await?;
assert_eq!(result, "hello world!");

// 如果 task panic，则 await 返回  JoinErr
use tokio::task;

let join = task::spawn(async {
    panic!("something bad happened!")
});

// The returned result indicates that the task failed.
assert!(join.await.is_err());
```

spawn 的 task 可以通过 `JoinHandle.abort()` 方法来取消（Cancelled）, 对应的 task 会在 `下一次 .await 点被 yield` 时终止。另外当 shutdown Runtime 时，如 #[tokio::main] 修饰的 async fn 返回时，这些 task 也将被终止。这时，JoinHandle 的 .await 结果是 JoinErr，他的 is_cancelled() 为 true；

-   abort task 并不代表 task 一定以 JoinErr 结束，因为有些任务可能在 yield 前正常结束，这时 .await 返回正常结果。
-   调用 JoinHandle.abort() 只是给运行的 task 发送了信号，但是并不等待 task 都结束才返回。
-   可以使用 JoinHandle .await 来确保 task 都返回。

如果使用的是不是 tokio crate 的 APIs，如标准库的 IO APIs，则可能会阻塞 tokio Runtime。对于可能会引起阻塞的任务，tokio 提供了在 async context 中运行的 `task::spawn_blocking() 和 task::block_in_place()`
函数, 前者是在单独的 blocing thread pool 中运行同步任务（clouse 来标识），从而避免阻塞运行 aysnc
task 的线程。

```rust
let join = task::spawn_blocking(|| { // 在单独的线程中运行可能会阻塞 tokio 的代码
    // do some compute-heavy work or call synchronous code
    "blocking completed"
});

let result = join.await?;
assert_eq!(result, "blocking completed");
```

如果使用的多线程 runtime，则 `task::block_in_place()` 也是可用的，和 task::spawn_blocking() 类似，他也是在 async context 中运行可能 blocking 当前线程的代码，但是他是将 Runtime 的 worker thread 转换为
blocking thread 来实现的，这样可以避免上下文切换来提升性能：

```rust
use tokio::task;

let result = task::block_in_place(|| {
    // do some compute-heavy work or call synchronous code
    "blocking completed"
});

assert_eq!(result, "blocking completed");
```

调用 async fn `task::yield_now()` 函数类似于 std::thread::yield_now(), 调用并且 .await 该函数将会使当前 task yield to tokio Runtime 调度器，让其他 task 被调度执行。

```rust
use tokio::task;

async {
    task::spawn(async {
        // ...
        println!("spawned task done!")
    });

    // Yield, allowing the newly-spawned task to execute first.
    task::yield_now().await; // 必须 .await
    println!("main task done!");
}
```

协作式调度：tokio Runtime 没有使用 OS thread 的抢占式调度，而是使用协作式调度，可以避免一个 task 执行时长时间占有 CPU 而影响其他 task 的执行。这是通过在 tokio libray 中 `强制插入一些 yield point` ，从而强制实现 task 周期返回 executor，从而可以调度其他 task 运行。

`task::unconstrained` 可以对 task 规避 tokio 协作式调度，使用他包裹的 Future task 不会 forced to yield
to Tokio：

```rust
use tokio::{task, sync::mpsc};

let fut = async {
    let (tx, mut rx) = mpsc::unbounded_channel();

    for i in 0..1000 {
        let _ = tx.send(());
        // This will always be ready. If coop was in effect, this code would be forced to yield
        // periodically. However, if left unconstrained, then this code will never yield.
        rx.recv().await;
    }
};

task::unconstrained(fut).await;
```


### <span class="section-num">4.1</span> JoinSet/JoinHandle/AbortHandle {#joinset-joinhandle-aborthandle}

在 tokion runtime 上 spawn 一批 task，等待一些或全部执行完成，task 按照完成的顺序返回。

-   所有任务的返回类型 T 必须相同；
-   如果 JoinSet 被 Drop，则其中所有 task 都会立即被 aborted；

<!--listend-->

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..10 {
        set.spawn(async move { i });
    }

    let mut seen = [false; 10];
    while let Some(res) = set.join_next().await { // join_next() 返回下一个返回的 task 结果
        let idx = res.unwrap();
        seen[idx] = true;
    }

    for i in 0..10 {
        assert!(seen[i]);
    }
}
```

JoinSet 的方法：

```rust
impl<T> JoinSet<T>
// 创建一个 JoinSet
pub fn new() -> Self
// 返回 JoinSet 中 task 数量
pub fn len(&self) -> usize
pub fn is_empty(&self) -> bool

impl<T: 'static> JoinSet<T>
// 使用 Builder 来构造一个 task（只能设置 然后再 ），name spawn
pub fn build_task(&mut self) -> Builder<'_, T>
use tokio::task::JoinSet;
#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut set = JoinSet::new();
    // Use the builder to configure a task's name before spawning it.
    set.build_task()
        .name("my_task")
        .spawn(async { /* ... */ })?;
    Ok(())
}

// 在 JoinSet 中 spwan 一个 task，该 task 会立即运行
pub fn spawn<F>(&mut self, task: F) -> AbortHandle
where
    F: Future<Output = T> + Send + 'static,
    T: Send,

// 在指定的 Handler 对应的 Runtime 上 spawn task
pub fn spawn_on<F>(&mut self, task: F, handle: &Handle) -> AbortHandle
where
    F: Future<Output = T> + Send + 'static,
    T: Send,

pub fn spawn_local<F>(&mut self, task: F) -> AbortHandle
where
    F: Future<Output = T> + 'static,
pub fn spawn_local_on<F>(
    &mut self,
    task: F,
    local_set: &LocalSet
) -> AbortHandle
where
    F: Future<Output = T> + 'static,

pub fn spawn_blocking<F>(&mut self, f: F) -> AbortHandle
where
    F: FnOnce() -> T + Send + 'static,
    T: Send,

use tokio::task::JoinSet;
#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();
    for i in 0..10 {
        set.spawn_blocking(move || { i });
    }
    let mut seen = [false; 10];
    while let Some(res) = set.join_next().await {
        let idx = res.unwrap();
        seen[idx] = true;
    }
    for i in 0..10 {
        assert!(seen[i]);
    }
}

pub fn spawn_blocking_on<F>(&mut self, f: F, handle: &Handle) -> AbortHandle
where
    F: FnOnce() -> T + Send + 'static,
    T: Send,

// 等待直到其中一个 task 完成，返回他的 output，注意是异步函数。
// 当 JoinSet 为空时，返回 None，可用于 while let 中。
pub async fn join_next(&mut self) -> Option<Result<T, JoinError>>

pub async fn join_next_with_id(&mut self) -> Option<Result<(Id, T), JoinError>>
pub fn try_join_next(&mut self) -> Option<Result<T, JoinError>>
pub fn try_join_next_with_id(&mut self) -> Option<Result<(Id, T), JoinError>>

// Aborts all tasks and waits for them to finish shutting down. Calling this method is equivalent to
// calling abort_all and then calling join_next in a loop until it returns None.
pub async fn shutdown(&mut self)

// Aborts all tasks on this JoinSet. This does not remove the tasks from the JoinSet. To wait for
// the tasks to complete cancellation, you should call join_next in a loop until the JoinSet is
// empty.
pub fn abort_all(&mut self)

// Removes all tasks from this JoinSet without aborting them.
// The tasks removed by this call will continue to run in the background even if the JoinSet is dropped.
pub fn detach_all(&mut self)

pub fn poll_join_next(
    &mut self,
    cx: &mut Context<'_>
) -> Poll<Option<Result<T, JoinError>>>

pub fn poll_join_next_with_id(
    &mut self,
    cx: &mut Context<'_>
) -> Poll<Option<Result<(Id, T), JoinError>>>
```

JoinSet 的个 spawn() 都返回 `AbortHandle` 对象，通过该对象可以 abort 对应的 spawned task，而不等待他运行结束。

-   不像 JoinHandle 那样 await task 结束，AbortHandle 只用于 terminate task 而不等待他结束；
-   Drop AbortHandle 表示释放了终止任务的权利，并不会终止任务。

<!--listend-->

```rust
impl AbortHandle
// 终止 task。JoinHandle 可能返回 JoinError 错误（也可能正常结束）
pub fn abort(&self)
// abort/canceld task 是否完成
pub fn is_finished(&self) -> bool

pub fn id(&self) -> Id
```

另外，task::spwan() 和 task::spawn_blocking() 返回的 `JoinHandle` 类型的 abort_handle() 方法也可以返回该 task 的AbortHandle：

-   JoinHandle 实现了 Future，可以进行 .await 结果；

<!--listend-->

```rust
use tokio::{time, task};

let mut handles = Vec::new();

handles.push(tokio::spawn(async {
   time::sleep(time::Duration::from_secs(10)).await;
   true
}));

handles.push(tokio::spawn(async {
   time::sleep(time::Duration::from_secs(10)).await;
   false
}));

let abort_handles: Vec<task::AbortHandle> = handles.iter().map(|h| h.abort_handle()).collect();

for handle in abort_handles {
    handle.abort();
}

for handle in handles {
    assert!(handle.await.unwrap_err().is_cancelled());
}
```

JoinHandle 实现了 Future，他的 Output = Result&lt;T, JoinError&gt;， `JoinError`  类型可以判断 task 是否被
cancelled/panic 等出错原因：

```rust
§
impl JoinError
pub fn is_cancelled(&self) -> bool
pub fn is_panic(&self) -> bool

use std::panic;
#[tokio::main]
async fn main() {
    let err = tokio::spawn(async {
        panic!("boom");
    }).await.unwrap_err();
    assert!(err.is_panic());
}

pub fn into_panic(self) -> Box<dyn Any + Send + 'static>
pub fn try_into_panic(self) -> Result<Box<dyn Any + Send + 'static>, JoinError>
pub fn id(&self) -> Id
```


### <span class="section-num">4.2</span> LocalSet/spawn_local() {#localset-spawn-local}

在同一个 thread 上运行一批异步 task，可以避免 tokio::spawn() 等要求的 Future task 必须实现 Send 的要求。

1.  先使用 task::LocalSet::new() 创建一个 LocalSet；
2.  在 LocalSet::run_until 来运行一个 async Future Block，该函数是一个 async fn，需要 .await 结果；
    -   当 LocalSet 中所有 task 都结束时，.await 返回；
    -   该 Future 在 `LocalSet context` 中执行，可以调用 tokio::task::spawn_local() 来提交 !Send task；
3.  在 Future 中使用 `LocalSet::spawn_local() 或者 tokio::task::spawn_local()` 来运行一个 !Send 的 task；
    -   只能使用这一个方法来提交任务，不能使用 task::spawn() 函数；

注意：

1.  LocalSet::run_until 只能在 #[tokio::main] /#[tokio::test] 或者 directly inside a call to

Runtime::block_on 中调用，不能在 tokio::spawn() 中调用。

1.  LocalSet 中 tokio::spawn_local() 提交的 !Send task 在 Runtime::block_on() 所在的单线程中调用。

<!--listend-->

```rust
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    let nonsend_data = Rc::new("my nonsend data...");
    let local = task::LocalSet::new();
    // Run the local task set.
    local.run_until(async move { // 指定调用 local.enter(), Future 位于该 LocalSet context 中
        let nonsend_data = nonsend_data.clone();
        // `spawn_local` ensures that the future is spawned on the local task set.

        // 在 run_until() 内部，使用 task::spawn_local() 提交任务。
        task::spawn_local(async move { // 可以调用多次
            println!("{}", nonsend_data);
            // ...
        }).await.unwrap(); // .await 在所有 spawn_local() 提交的 task 结束时返回
    }).await;
}

// 另一个例子
use tokio::{task, time};
use std::rc::Rc;

#[tokio::main]
async fn main() {
    let nonsend_data = Rc::new("world");
    let local = task::LocalSet::new();

    let nonsend_data2 = nonsend_data.clone();
    // 使用 LocalSet.spawn_local() 提交任务
    local.spawn_local(async move {
        // ...
        println!("hello {}", nonsend_data2)
    });

    local.spawn_local(async move {
        time::sleep(time::Duration::from_millis(100)).await;
        println!("goodbye {}", nonsend_data)
    });

    // ...

    local.await;
}
```

LocalSet 的方法：

```rust
source
impl LocalSet
// 创建一个 LocalSet
pub fn new() -> LocalSet
// 进入 LocalSet 的 context，这样后续使用 tokio::task::spawn_local() 提交 !Send task;
pub fn enter(&self) -> LocalEnterGuard
// 提交一个 !Send task， 返回可以 .await 的 JoinHandle
pub fn spawn_local<F>(&self, future: F) -> JoinHandle<F::Output> ⓘ
where
    F: Future + 'static,
    F::Output: 'static,
// 同步执行 future，直到返回
pub fn block_on<F>(&self, rt: &Runtime, future: F) -> F::Output
where
    F: Future,
// 执行一个 Future, 内部可以使用 tokio::task::spawn_local() 提交 !Send task, 运行直到这些 task 结束；
pub async fn run_until<F>(&self, future: F) -> F::Output
where
    F: Future,

impl LocalSet
pub fn unhandled_panic(&mut self, behavior: UnhandledPanic) -> &mut Self
pub fn id(&self) -> Id
```

示例：

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let local = task::LocalSet::new();

    // Spawn a future on the local set. This future will be run when
    // we call `run_until` to drive the task set.
    local.spawn_local(async {
       // ...
    });

    // Run the local task set.
    local.run_until(async move {
        // ...
    }).await;

    // When `run` finishes, we can spawn _more_ futures, which will
    // run in subsequent calls to `run_until`.
    local.spawn_local(async {
       // ...
    });

    local.run_until(async move {
        // ...
    }).await;
}
```


## <span class="section-num">5</span> spawn {#spawn}

concurrency/parallelism:

-   concurrency: 指的是同时在多个 task 间切换执行, 单 thread 也可以实现 concurrency;
-   parallelism: 指的是有个同时执行的实体, 如多 CPU Core, 多 thread;

非并发异步版本: 虽然使用了 aync, 但是每个连接都是在 loop 中串行执行的, 所以没有实现 concurrency;

```rust
use tokio::net::{TcpListener, TcpStream};
use mini_redis::{Connection, Frame};

#[tokio::main]
async fn main() {
    // Bind the listener to the address
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        // The second item contains the IP and port of the new connection.
        let (socket, _) = listener.accept().await.unwrap();
        process(socket).await;
    }
}

async fn process(socket: TcpStream) {
    // The `Connection` lets us read/write redis **frames** instead of
    // byte streams. The `Connection` type is defined by mini-redis.
    let mut connection = Connection::new(socket);

    if let Some(frame) = connection.read_frame().await.unwrap() {
        println!("GOT: {:?}", frame);

        // Respond with an error
        let response = Frame::Error("unimplemented".to_string());
        connection.write_frame(&response).await.unwrap();
    }
}
```

异步并发版本:

-   tokio Task 是非常轻量的 green-thread (相比数千字节的 thread stack, task 只占用 64 字节的内存开销),
    通过 async block 来进行创建.
-   tokio::spawn(async move {...}) 将异步任务提交给 tokio execution runtime 进行 concurrency 并发执行;

<!--listend-->

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // A new task is spawned for each inbound socket. The socket is moved to the new task and
        // processed there.
        tokio::spawn(async move {
            process(socket).await;
        });
    }
}
```

async task block 必须是 'static 生命周期的, 所以不能直接共享引用 stack 上的变量:

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];
    task::spawn(async {
        println!("Here's a vec: {:?}", v); // 错误
    });
}
```

报错: This happens because, by default, variables are `not moved into` async blocks. The v vector
remains `owned by the main function`. The println! line borrows v. The rust compiler helpfully
explains this to us and even suggests the fix! Changing line 7 to `task::spawn(async move {` will
instruct the compiler to move v into the spawned task. Now, `the task owns all of its data`, making it
'static.

```rust
error[E0373]: async block may outlive the current function, but
              it borrows `v`, which is owned by the current function
 --> src/main.rs:7:23
  |
7 |       task::spawn(async {
  |  _______________________^
8 | |         println!("Here's a vec: {:?}", v);
  | |                                        - `v` is borrowed here
9 | |     });
  | |_____^ may outlive borrowed value `v`
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:7:17
  |
7 |       task::spawn(async {
  |  _________________^
8 | |         println!("Here's a vector: {:?}", v);
9 | |     });
  | |_____^
help: to force the async block to take ownership of `v` (and any other
      referenced variables), use the `move` keyword
  |
7 |     task::spawn(async move {
8 |         println!("Here's a vec: {:?}", v);
9 |     });
  |
```

async task 的签名是 Future + Send + 'static, 所以:

1.  使用 move 来将引用的外部对象的所有权转移到 task, 从而满足 'static 的问题;
2.  task 会在多个 thread 间调度, 所以必须是可以 Send 的, 这意味着 task 内部的跨 await 的变量(因为在
    await 处生成 Future object, 它记录了上下文变量)必须实现了 Send;

<!--listend-->

```rust
// OK
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        // The scope forces `rc` to drop before `.await`.
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        }

        // `rc` is no longer used. It is **not** persisted when
        // the task yields to the scheduler
        yield_now().await;
    });
}

// Error
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");

        // `rc` is used after `.await`. It must be persisted to
        // the task's state.
        yield_now().await;

        println!("{}", rc);
    });
}
```

报错: 解决办法: 将 Rc 改成实现 Send 的 Arc:

```shell
error: future cannot be sent between threads safely
--> src/main.rs:6:5
	|
	6   |     tokio::spawn(async {
	|     ^^^^^^^^^^^^ future created by async block is not `Send`
	|
	::: [..]spawn.rs:127:21
	|
	127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in
    |                          `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait
    |       `std::marker::Send` is not  implemented for
    |       `std::rc::Rc<&str>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:10:9
    |
7   |         let rc = Rc::new("hello");
    |             -- has type `std::rc::Rc<&str>` which is not `Send`
...
10  |         yield_now().await;
    |         ^^^^^^^^^^^^^^^^^ await occurs here, with `rc` maybe
    |                           used later
11  |         println!("{}", rc);
12  |     });
    |     - `rc` is later dropped here
```


## <span class="section-num">6</span> mutex {#mutex}

异步任务中也可以使用 std::sync::Arc/Mutex 同步原语来对共享内存进行并发访问:

-   这里使用 std::sync::Mutex 而非 tokio::sync::Mutex,  `只有当 Mutex 跨 await 时才需要使用异步 Mutex`;
-   Mutex lock 阻塞时会阻塞当前 thread, 会导致其他 async task 也不能调度到该 thread 执行;
-   缺省情况下, tokio runtime 使用 multi-thread scheduler, task 被调度到那个 thread 是 tokio runtime
    决定的; tokio 还提供 current_thread runtime flavor, 它是一个轻量级, 单线程的 runtime.

<!--listend-->

```rust
use bytes::Bytes;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

type Db = Arc<Mutex<HashMap<String, Bytes>>>; // 定义类型别名

use tokio::net::TcpListener;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();
    println!("Listening");
    let db = Arc::new(Mutex::new(HashMap::new()));
    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // Clone the handle to the hash map.
        let db = db.clone();
        println!("Accepted");
        tokio::spawn(async move {
            process(socket, db).await;
        });
    }
}

use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};

async fn process(socket: TcpStream, db: Db) {
    use mini_redis::Command::{self, Get, Set};

    // Connection, provided by `mini-redis`, handles parsing frames from the socket
    let mut connection = Connection::new(socket);

    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                let mut db = db.lock().unwrap();  // db 是 MutexGuard 类型, 声明周期没有跨 await, 所以可以使用同步 Mutex;
                db.insert(cmd.key().to_string(), cmd.value().clone());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                let db = db.lock().unwrap();
                if let Some(value) = db.get(cmd.key()) {
                    Frame::Bulk(value.clone())
                } else {
                    Frame::Null
                }
            }
            cmd => panic!("unimplemented {:?}", cmd),
        };

        // Write the response to the client
        connection.write_frame(&response).await.unwrap();
    }
}
```

为了尽量降低 Mutex 竞争导致的 task/thread block, 可以:

1.  Switching to a dedicated task to manage state and use message passing.
2.  `Shard the mutex.`
3.  Restructure the code to avoid the mutex.

对于第二种情况的例子: 将单一 Mutex&lt;HashMap&lt;_, \__&gt;&gt; 实例,拆解为 N 个实例:

-   第三方 crate dashmap 提供了更复杂的 sharded hash map;

<!--listend-->

```rust
type ShardedDb = Arc<Vec<Mutex<HashMap<String, Vec<u8>>>>>;

fn new_sharded_db(num_shards: usize) -> ShardedDb {
    let mut db = Vec::with_capacity(num_shards);
    for _ in 0..num_shards {
        db.push(Mutex::new(HashMap::new()));
    }
    Arc::new(db)
}

// 后续查找 map 前先计算下 key 的 hash
let shard = db[hash(key) % db.len()].lock().unwrap();
shard.insert(key, value);
```

由于 MutexGuard 没有实现 Send, 在持有 MutexGuard 的情况下不能跨 .await:

-   Rust 在 .await 时有可能将当前 task 转移到其他 thread 运行, 所以需要 async task block 实现 Send.

<!--listend-->

```rust
use std::sync::{Mutex, MutexGuard};

async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;

    do_something_async().await;
} // lock goes out of scope here
```

编译报错: error: future cannot be sent between threads safely

```text
error: future cannot be sent between threads safely
   --> src/lib.rs:13:5
    |
13  |     tokio::spawn(async move {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-0.2.21/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/lib.rs:7:5
    |
4   |     let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    |         -------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
...
7   |     do_something_async().await;
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ await occurs here, with `mut lock` maybe used later
8   | }
    | - `mut lock` is later dropped here
```

解决办法: 将 MutexGuard 的 destructor 在 await 前运行:

```rust
// This works!
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    {
        let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
        *lock += 1;
    } // lock goes out of scope here

    do_something_async().await;
}

// This fails too.
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;
    drop(lock);

    do_something_async().await;
}
```

或者使用 struct 来封装 Mutex, 然后在同步函数中处理锁:

```rust
use std::sync::Mutex;

struct CanIncrement {
    mutex: Mutex<i32>,
}
impl CanIncrement {
    // This function is not marked async.
    fn increment(&self) {
        let mut lock = self.mutex.lock().unwrap();
        *lock += 1;
    }
}

async fn increment_and_do_stuff(can_incr: &CanIncrement) {
    can_incr.increment();
    do_something_async().await;
}
```

或者, 使用 tokio 的异步 Mutext, tokio::sync::Mutex 支持跨 .await, 但是性能会差一些:

```rust
use tokio::sync::Mutex; // note! This uses the Tokio mutex

// This compiles!
// (but restructuring the code would be better in this case)
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock = mutex.lock().await;
    *lock += 1;

    do_something_async().await;
} // lock goes out of scope here
```

综上, 对于 Mutext 和带来的 Send 问题:

1.  首选将 Mutext 封装到 Struct 和同步函数中;
2.  或者将 Mutext 在 .await 前解构(必须是 block 解构,而不是 drop());
3.  或者 spawn 一个 task 来专门管理 state, 其他 task 使用 message 来对它进行操作.


## <span class="section-num">7</span> channel {#channel}

tokio mpsc message passing:

1.  一个 tokio spawn task 作为 manager 角色, 通过 buffered mpsc channel 接收 message, 然后根据
    message 类型来操作有状态对象, 由于只有 manager 来串行操作该对象, 所以可以避免加锁.
2.  manager 通过 message 中的 response channel 来向发送者响应结果;
3.  tokio::sync::oneshot 是 a single-producer, single-consumer channel optimized for sending a single
    value.

<!--listend-->

```rust
use bytes::Bytes;
use mini_redis::client;
use tokio::sync::{mpsc, oneshot};

/// Multiple different commands are multiplexed over a single channel.
#[derive(Debug)]
enum Command {
    Get {
        key: String,
        resp: Responder<Option<Bytes>>, // 发送响应的 oneshot 类型 channel
    },
    Set {
        key: String,
        val: Bytes,
        resp: Responder<()>,
    },
}

/// Provided by the requester and used by the manager task to send the command
/// response back to the requester.
type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;

#[tokio::main]
async fn main() {
    // 创建一个 buffered 类型 channel, 当消费的慢时可以对发送端反压( send(...).await 将阻塞一段时间),从
    // 而降低内存和并发度, 防止消耗过多系统资源.
    let (tx, mut rx) = mpsc::channel(32);
    // Clone a `tx` handle for the second f
    let tx2 = tx.clone();

    let manager = tokio::spawn(async move {
        // Open a connection to the mini-redis address.
        let mut client = client::connect("127.0.0.1:6379").await.unwrap();

        while let Some(cmd) = rx.recv().await {
            match cmd {
                Command::Get { key, resp } => {
                    let res = client.get(&key).await;
                    // Ignore errors
                    let _ = resp.send(res); // 通过 oneshot channel 发送响应
                }
                Command::Set { key, val, resp } => {
                    let res = client.set(&key, val).await;
                    // Ignore errors
                    let _ = resp.send(res);
                }
            }
        }
    });

    // Spawn two tasks, one setting a value and other querying for key that was
    // set.
    let t1 = tokio::spawn(async move {
        let (resp_tx, resp_rx) = oneshot::channel(); // 构建响应 channel
        let cmd = Command::Get {
            key: "foo".to_string(),
            resp: resp_tx,
        };

        // Send the GET request
        if tx.send(cmd).await.is_err() {
            eprintln!("connection task shutdown");
            return;
        }

        // Await the response
        let res = resp_rx.await;
        println!("GOT (Get) = {:?}", res);
    });

    let t2 = tokio::spawn(async move {
        let (resp_tx, resp_rx) = oneshot::channel();
        let cmd = Command::Set {
            key: "foo".to_string(),
            val: "bar".into(),
            resp: resp_tx,
        };

        // Send the SET request
        if tx2.send(cmd).await.is_err() {
            eprintln!("connection task shutdown");
            return;
        }

        // Await the response
        let res = resp_rx.await;
        println!("GOT (Set) = {:?}", res);
    });

    t1.await.unwrap();
    t2.await.unwrap();
    manager.await.unwrap();
}
```

std:sync::mpsc 和 crossbeam::channel 都是同步 channel, 不能在 async func 中使用, 否则可能 block 当前线程和 task.


## <span class="section-num">8</span> async io {#async-io}

I/O in Tokio operates in much the same way as in std, but asynchronously. There is a trait for
`reading (AsyncRead)` and a trait for `writing (AsyncWrite)`. Specific types implement these traits as
appropriate (TcpStream, File, Stdout). AsyncRead and AsyncWrite are also implemented by a number of
data structures, such as Vec&lt;u8&gt; and &amp;[u8]. This allows using byte arrays where a reader or writer
is expected. The methods on these traits are typically not called directly, similar to how you don't
manually call `the poll method from the Future trait`. Instead, you will use them through the utility
methods provided by `AsyncReadExt and AsyncWriteExt`.

AsyncReadExt trait 提供的方法返回的对象都实现了 Future:

```rust
pub trait AsyncReadExt: AsyncRead {
    // Provided methods
    fn chain<R>(self, next: R) -> Chain<Self, R> where Self: Sized, R: AsyncRead { ... }
    fn read<'a>(&'a mut self, buf: &'a mut [u8]) -> Read<'a, Self> where Self: Unpin { ... }
    fn read_buf<'a, B>(&'a mut self, buf: &'a mut B) -> ReadBuf<'a, Self, B> where Self: Unpin, B: BufMut + ?Sized { ... }
    fn read_exact<'a>(&'a mut self, buf: &'a mut [u8]) -> ReadExact<'a, Self> where Self: Unpin { ... }
    fn read_u8(&mut self) -> ReadU8<&mut Self> where Self: Unpin { ... }
```

例如 read() 方法返回的 Read 对象实现了 Future, Poll ready 时返回 io::Result&lt;usize&gt;, 所以, 即使这些方法没有使用 async fn 但由于返回的对象实现了 Future, 它们也是异步函数;

```rust
// https://docs.rs/tokio/latest/src/tokio/io/util/read.rs.html#43
impl<R> Future for Read<'_, R>
where
    R: AsyncRead + Unpin + ?Sized,
{
    type Output = io::Result<usize>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<usize>> {
        let me = self.project();
        let mut buf = ReadBuf::new(me.buf);
        ready!(Pin::new(me.reader).poll_read(cx, &mut buf))?;
        Poll::Ready(Ok(buf.filled().len()))
    }
}
```

使用示例:

```rust
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt};

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer[..]).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}

use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = Vec::new();

    // read the whole file
    f.read_to_end(&mut buffer).await?;
    Ok(())
}
```

async write:

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    // Writes some prefix of the byte string, but not necessarily all of it.
    let n = file.write(b"some bytes").await?;

    println!("Wrote the first {} bytes of 'some bytes'.", n);
    Ok(())
}

use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    file.write_all(b"some bytes").await?;
    Ok(())
}
```

echo server client:

-   使用 tokio::io::split() 将一个实现 read/write 的对象拆分为 read 和 write 两个对象;

<!--listend-->

```rust
// https://tokio.rs/tokio/tutorial/io

use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> io::Result<()> {
    let socket = TcpStream::connect("127.0.0.1:6142").await?;
    let (mut rd, mut wr) = io::split(socket);

    // Write data in the background
    tokio::spawn(async move {
        wr.write_all(b"hello\r\n").await?;
        wr.write_all(b"world\r\n").await?;

        // Sometimes, the rust type inferencer needs
        // a little help
        Ok::<_, io::Error>(())
    });

    let mut buf = vec![0; 128];

    loop {
        let n = rd.read(&mut buf).await?;

        if n == 0 {
            break;
        }

        println!("GOT {:?}", &buf[..n]);
    }

    Ok(())
}
```

也可以使用 tokio::io::copy() 来实现 copy:

```rust
tokio::spawn(async move {
    let (mut rd, mut wr) = socket.split();

    if io::copy(&mut rd, &mut wr).await.is_err() {
        eprintln!("failed to copy");
    }
});
```

echo server:

-   尽量避免 stack buffer, 因为跨 .await 的上下文变量会随者 task Future 对象一起被保存, 如果使用较大的
    stack buffer 变量, 则自动生成的 task Future 对象就比较大, buffer size 一般是 page sized 对齐的, 这导致task 的大小大概是 $page-size + a-few-bytes, 从而比较浪费内存.

<!--listend-->

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = vec![0; 1024]; // 在堆中分配 buff 内存, 而不使用 stack 上的 array

            loop {
                match socket.read(&mut buf).await {
                    // Return value of `Ok(0)` signifies that the remote has closed
                    Ok(0) => return,
                    Ok(n) => {
                        // Copy the data back to socket
                        if socket.write_all(&buf[..n]).await.is_err() {
                            // Unexpected socket error. There isn't much we can do here so just stop
                            // processing.
                            return;
                        }
                    }
                    Err(_) => {
                        // Unexpected socket error. There isn't much we can do here so just stop
                        // processing.
                        return;
                    }
                }
            }
        });
    }
}
```


## <span class="section-num">9</span> framing {#framing}

<https://tokio.rs/tokio/tutorial/framing>

TCP Stream 是面向 byte stream 的协议, 而 redis 操作是以 frame 为单位的, 所以需要从 byte stream 中构造 frame.

核心是使用 bytes::BytesBuf 作为读写缓存, 它内部维护了读写位置, 自动根据读写位置来伸缩内部的缓冲区.

Framing is the process of taking a byte stream and converting it to a stream of frames. A frame is a
unit of data transmitted between two peers. The Redis protocol frame is defined as follows:

```rust
use bytes::Bytes;

enum Frame {
    Simple(String),
    Error(String),
    Integer(u64),
    Bulk(Bytes),
    Null,
    Array(Vec<Frame>),
}


use tokio::io::AsyncReadExt;
use bytes::Buf;
use mini_redis::Result;

pub async fn read_frame(&mut self)
    -> Result<Option<Frame>>
{
    loop {
        // Attempt to parse a frame from the buffered data. If
        // enough data has been buffered, the frame is
        // returned.
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }

        // There is not enough buffered data to read a frame.
        // Attempt to read more data from the socket.
        //
        // On success, the number of bytes is returned. `0`
        // indicates "end of stream".
        if 0 == self.stream.read_buf(&mut self.buffer).await? {
            // The remote closed the connection. For this to be
            // a clean shutdown, there should be no data in the
            // read buffer. If there is, this means that the
            // peer closed the socket while sending a frame.
            if self.buffer.is_empty() {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        }
    }
}

use mini_redis::{Frame, Result};
use mini_redis::frame::Error::Incomplete;
use bytes::Buf;
use std::io::Cursor;

fn parse_frame(&mut self)
    -> Result<Option<Frame>>
{
    // Create the `T: Buf` type.
    let mut buf = Cursor::new(&self.buffer[..]); // 从 BytesMut buffer 创建 cursor

    // Check whether a full frame is available
    match Frame::check(&mut buf) {
        Ok(_) => {
            // Get the byte length of the frame
            let len = buf.position() as usize;

            // Reset the internal cursor for the
            // call to `parse`.
            buf.set_position(0);

            // Parse the frame
            let frame = Frame::parse(&mut buf)?; // 从 cursor buf 解析数据

            // Discard the frame from the buffer
            self.buffer.advance(len); // 增加底层 BytesMut buffer 的 read 位置

            // Return the frame to the caller.
            Ok(Some(frame))
        }
        // Not enough data has been buffered
        Err(Incomplete) => Ok(None),
        // An error was encountered
        Err(e) => Err(e.into()),
    }
}


use tokio::io::BufWriter;
use tokio::net::TcpStream;
use bytes::BytesMut;

pub struct Connection {
    stream: BufWriter<TcpStream>,
    buffer: BytesMut,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream: BufWriter::new(stream),
            buffer: BytesMut::with_capacity(4096),
        }
    }
}

use tokio::io::{self, AsyncWriteExt};
use mini_redis::Frame;

async fn write_frame(&mut self, frame: &Frame)
    -> io::Result<()>
{
    match frame {
        Frame::Simple(val) => {
            self.stream.write_u8(b'+').await?;
            self.stream.write_all(val.as_bytes()).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Error(val) => {
            self.stream.write_u8(b'-').await?;
            self.stream.write_all(val.as_bytes()).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Integer(val) => {
            self.stream.write_u8(b':').await?;
            self.write_decimal(*val).await?;
        }
        Frame::Null => {
            self.stream.write_all(b"$-1\r\n").await?;
        }
        Frame::Bulk(val) => {
            let len = val.len();

            self.stream.write_u8(b'$').await?;
            self.write_decimal(len as u64).await?;
            self.stream.write_all(val).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Array(_val) => unimplemented!(),
    }

    self.stream.flush().await;

    Ok(())
}
```


## <span class="section-num">10</span> Async in depth {#async-in-depth}

```rust
use tokio::net::TcpStream;

// 定义一个异步函数, 内部可以调用 .await
async fn my_async_fn() {
    println!("hello from async");
    let _socket = TcpStream::connect("127.0.0.1:3000").await.unwrap();
    println!("async TCP operation complete");
}

// main 函数也是一个异步函数, 作为异步入口, 由 tokio runtime 执行
#[tokio::main]
async fn main() {
    let what_is_this = my_async_fn(); // 返回一个 std::future::Future 对象
    // Nothing has been printed yet.

    what_is_this.await; // Pool Future 对象
    // Text has been printed and socket has been
    // established and closed.
}
```

std:future::Future 对象的定义:

-   Pin 用于在 async 函数中, 第一次 poll 时确保 Future 对象不被 move, 从而防止内部保存的 stack 变量指针不会失效.
-   Future 对象不会在后台自动执行, 而是被 poll 时才执行;

<!--listend-->

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Self::Output>;
}
```

生成一个 Future 对象的方式:

1.  使用 async fn 或 async block, 当调用它们时, rustc 自动生成一个 Future 对象.
2.  定义一个对象, 实现 Future trait;
3.  .await 操作时, 如果 Future poll 返回 Pending, 则生成一个新的 Future 对象, 保存当前执行位置上下文,
    下次 poll 时从该位置继续运行;

<!--listend-->

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str; // 返回值的关联类型

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done") // Ready 返回关联值类型
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref(); // 通过传入 ctx 的 waker 来通知 executor 来再次进行 poll.
     	// 有缺陷版本: 应该起一个 thread 来确保条件满足时调用 waker, 而不是当前的 busy poll
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };

    let out = future.await; // Delay 实现了 Future, 所以可以 .await
    assert_eq!(out, "done");
}
```

tokio::spawn(async move{}) : 创建一个 async task, 并提交给 executor 来调度执行.

executor 在 poll Future 时, 需要传入 context, 其中包含后续可以唤醒 executor 来重复 poll 的
waker. executor 负责创建该 waker, Future 实现负责保存传入的 waker, 然后合适实际调用它;

Tokio Executor 的简单实现:

-   使用 VecDeque 来保存 Future, 当 Future Pending, 再插入队尾, 下次再次轮询 poll;
-   实际实现时, executor 应该在手动 Future 的 waker 通知时再 poll 它;

<!--listend-->

```rust
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use futures::task;

fn main() {
    let mut mini_tokio = MiniTokio::new();

    mini_tokio.spawn(async {
        let when = Instant::now() + Duration::from_millis(10);
        let future = Delay { when };

        let out = future.await;
        assert_eq!(out, "done");
    });

    mini_tokio.run();
}

struct MiniTokio {
    tasks: VecDeque<Task>,
}

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

impl MiniTokio {
    fn new() -> MiniTokio {
        MiniTokio {
            tasks: VecDeque::new(),
        }
    }

    /// Spawn a future onto the mini-tokio instance.
    fn spawn<F>(&mut self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        self.tasks.push_back(Box::pin(future));
    }

    fn run(&mut self) {
        let waker = task::noop_waker();
        let mut cx = Context::from_waker(&waker); // executor 创建 waker 并在 poll future 时传入

        while let Some(mut task) = self.tasks.pop_front() {
            if task.as_mut().poll(&mut cx).is_pending() {
                self.tasks.push_back(task);
            }
        }
    }
}
```

Future 内部保存和使用 waker(有缺陷版本):

-   缺陷: executor 有可能多次 poll 该 Future, 每次传入不同的 waker, 而 Future 的 poll 实现没有只保存最新的 waker. 同时没有判断是否已经创建 thread, 而不是每次都新建;

<!--listend-->

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use std::thread;

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Get a handle to the waker for the current task
            let waker = cx.waker().clone(); // 保存传入的 waker
            let when = self.when;

            // Spawn a timer thread.
            thread::spawn(move || { // 另起一个 thread, 当条件满足时, 调用保存的 waker 来唤醒 executor 来再次 poll
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                waker.wake();
            });

            Poll::Pending
        }
    }
}
```

Future 的优化实现版本: 保存最新的 Waker, 只启动一个 thread:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
    // This is Some when we have spawned a thread, and None otherwise.
    waker: Option<Arc<Mutex<Waker>>>, // 通过 Arc 和锁来保存最新的 waker
}

impl Future for Delay {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // Check the current instant. If the duration has elapsed, then
        // this future has completed so we return `Poll::Ready`.
        if Instant::now() >= self.when {
            return Poll::Ready(());
        }

        // The duration has not elapsed. If this is the first time the future
        // is called, spawn the timer thread. If the timer thread is already
        // running, ensure the stored `Waker` matches the current task's waker.
        if let Some(waker) = &self.waker {
            let mut waker = waker.lock().unwrap();

            // Check if the stored waker matches the current task's waker.
            // This is necessary as the `Delay` future instance may move to
            // a different task between calls to `poll`. If this happens, the
            // waker contained by the given `Context` will differ and we
            // must update our stored waker to reflect this change.
            if !waker.will_wake(cx.waker()) {
                *waker = cx.waker().clone();
            }
        } else {
            let when = self.when;
            let waker = Arc::new(Mutex::new(cx.waker().clone()));
            self.waker = Some(waker.clone());

            // This is the first time `poll` is called, spawn the timer thread.
	        // 线程内实际捕获的时 waker 对象, 即 Arc, 而不是 Waker struct, 这样后续更新时能拿到更新后的值
            thread::spawn(move || {
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                // The duration has elapsed. Notify the caller by invoking the waker.
                let waker = waker.lock().unwrap(); // 醒来后,从 Arc 中获得最新的 waker 对象
                waker.wake_by_ref();
            });
        }

        // By now, the waker is stored and the timer thread is started.
        // The duration has not elapsed (recall that we checked for this
        // first thing), ergo the future has not completed so we must
        // return `Poll::Pending`.
        //
        // The `Future` trait contract requires that when `Pending` is
        // returned, the future ensures that the given waker is signalled
        // once the future should be polled again. In our case, by
        // returning `Pending` here, we are promising that we will
        // invoke the given waker included in the `Context` argument
        // once the requested duration has elapsed. We ensure this by
        // spawning the timer thread above.
        //
        // If we forget to invoke the waker, the task will hang
        // indefinitely.
        Poll::Pending
    }
}
```

tokio 的新版实现: 收到 Future waker 通知时再 poll 它:

-   Task 自己实现了 Waker, 从自身创建要给 Waker, 当 waker 时, wake_by_ref() 会将 Task 自身添加到
    executor 的 poll 队列中, 而从实现 executor 按需 poll 执行 task;

<!--listend-->

```rust
use std::sync::mpsc;
use std::sync::Arc;
use std::sync::{Arc, Mutex};

struct MiniTokio {
    scheduled: mpsc::Receiver<Arc<Task>>, // tokio executor 接受通知的 channel
    sender: mpsc::Sender<Arc<Task>>,  // Task 向 tokio executor 发送通知的 channel
}

/// A structure holding a future and the result of the latest call to its `poll` method.
struct TaskFuture {
    future: Pin<Box<dyn Future<Output = ()> + Send>>,
    poll: Poll<()>, // 保存 future 的执行结果
}

struct Task {
    // The `Mutex` is to make `Task` implement `Sync`. Only one thread accesses `task_future` at any
    // given time.  The `Mutex` is not required for correctness. Real Tokio does not use a mutex
    // here, but real Tokio has more lines of code than can fit in a single tutorial page.
    task_future: Mutex<TaskFuture>,
    executor: mpsc::Sender<Arc<Task>>, // 向 tokio executor 发送通知的 channel
}

impl Task {
    fn schedule(self: &Arc<Self>) {
        self.executor.send(self.clone()); // 调度执行 Task
    }
}

// import futures = "0.3"
use futures::task::{self, ArcWake};
use std::sync::Arc;
impl ArcWake for Task {
    // 让 Task 实现 futures 的 ArcWake, 从而可以转换为 futures::task::Waker,
    // 在调用该 waker 时, 通过 schedule() 来通知 executor 来执行该 Task.
    fn wake_by_ref(arc_self: &Arc<Self>) {
        arc_self.schedule();
    }
}

impl MiniTokio {
    fn run(&self) {
        while let Ok(task) = self.scheduled.recv() {
            task.poll();
        }
    }

    /// Initialize a new mini-tokio instance.
    fn new() -> MiniTokio {
        let (sender, scheduled) = mpsc::channel();

        MiniTokio { scheduled, sender }
    }

    /// Spawn a future onto the mini-tokio instance.
    ///
    /// The given future is wrapped with the `Task` harness and pushed into the
    /// `scheduled` queue. The future will be executed when `run` is called.
    fn spawn<F>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        Task::spawn(future, &self.sender);
    }
}

impl TaskFuture {
    fn new(future: impl Future<Output = ()> + Send + 'static) -> TaskFuture {
        TaskFuture {
            future: Box::pin(future),
            poll: Poll::Pending,
        }
    }

    fn poll(&mut self, cx: &mut Context<'_>) {
        // Spurious wake-ups are allowed, even after a future has
        // returned `Ready`. However, polling a future which has
        // already returned `Ready` is *not* allowed. For this
        // reason we need to check that the future is still pending
        // before we call it. Failure to do so can lead to a panic.
        if self.poll.is_pending() {
            self.poll = self.future.as_mut().poll(cx);
        }
    }
}

impl Task {
    fn poll(self: Arc<Self>) {
        // Create a waker from the `Task` instance. This uses the `ArcWake` impl from above.
        let waker = task::waker(self.clone());  // Task 自身实现了 Waker
        let mut cx = Context::from_waker(&waker);

        // No other thread ever tries to lock the task_future
        let mut task_future = self.task_future.try_lock().unwrap();

        // Poll the inner future
        task_future.poll(&mut cx);
    }

    // Spawns a new task with the given future.
    //
    // Initializes a new Task harness containing the given future and pushes it
    // onto `sender`. The receiver half of the channel will get the task and
    // execute it.
    fn spawn<F>(future: F, sender: &mpsc::Sender<Arc<Task>>)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(Task {
            task_future: Mutex::new(TaskFuture::new(future)),
            executor: sender.clone(),
        });

        let _ = sender.send(task);
    }
}
```

tokio::sync::Notify 工具: 可以作为替代 waker 的异步函数内部 thread 间同步的工具:

```rust
use tokio::sync::Notify;
use std::sync::Arc;
use std::time::{Duration, Instant};
use std::thread;

async fn delay(dur: Duration) {
    let when = Instant::now() + dur;
    let notify = Arc::new(Notify::new());
    let notify_clone = notify.clone();

    thread::spawn(move || {
        let now = Instant::now();

        if now < when {
            thread::sleep(when - now);
        }

        notify_clone.notify_one();
    });


    notify.notified().await;
}
```


## <span class="section-num">11</span> select {#select}

The `tokio::select!` macro allows waiting on `multiple async computations` and returns when a single
computation completes. The branch that does not complete `is dropped`. In the example, the computation
is awaiting the oneshot::Receiver for each channel. `The oneshot::Receiver for the channel that did
not complete yet is dropped`.

-   select 随机选择一个 branch 来 first poll;

<!--listend-->

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    tokio::spawn(async {
        let _ = tx1.send("one");
    });

    tokio::spawn(async {
        let _ = tx2.send("two");
    });

    tokio::select! {  // select 并发 poll rx1 和 rx2, 如果 rx1 先 Ready 则结果赋值给 val, 同时 drop rx2.
        val = rx1 => {
            println!("rx1 completed first with {:?}", val);
        }
        val = rx2 => {
            println!("rx2 completed first with {:?}", val);
        }
    }
}
```

tokio::sync::oneshot::Receiver 实现了 Drop, 当他被 Drop 时对应的 tokio::sync::oneshot::Sender 可以收到 closed 通知(sender 的 async closed() 方法返回).

-   tokio::spawn(async {...}) 向 executor 理解调度执行一个异步 task, 返回一个实现 Future 的JoinHandle,
    Future 的关联类型是 Result&lt;T, JoinError&gt;, 所以通过 .await  它可以获得 task 的返回值 T.

<!--listend-->

```rust
use tokio::sync::oneshot;

async fn some_operation() -> String { // 异步函数
    // Compute value here
}

#[tokio::main]
async fn main() {
    let (mut tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    tokio::spawn(async {
        // Select on the operation and the oneshot's `closed()` notification.
        tokio::select! {
            val = some_operation() => {
                let _ = tx1.send(val);
            }
            // tx1.closed() 是一个 async 方法, 故可以被 select, 当对端 rx1 被 Drop 时返回.
            _ = tx1.closed() => {
                // `some_operation()` is canceled, the task completes and `tx1` is dropped.
            }
        }
    });

    tokio::spawn(async {
        let _ = tx2.send("two");
    });

    tokio::select! {
        val = rx1 => {
            println!("rx1 completed first with {:?}", val);
        }
        val = rx2 => {
            println!("rx2 completed first with {:?}", val);
        }
    }
}
```

The select! macro can handle more than two branches. `The current limit is 64 branches`. Each branch
is structured as:

```text
<pattern> = <async expression> => <handler>,
```

When the select macro is evaluated, `all the <async expression>s are aggregated and executed
concurrently` . When an expression completes, the result is matched against &lt;pattern&gt;. If the result
matches the pattern, `then all remaining async expressions are dropped` and &lt;handler&gt; is executed. The
&lt;handler&gt; expression has access to any bindings established by &lt;pattern&gt;.

-   rust 并发执行所有的 aync expressions;
-   当第一个 async expressions 返回时, 将他的 result 与 pattern 匹配时, drop 所有其他的 async
    expression, 然后执行 handler. 如果 result 与 pattern 不匹配时, 继续等待下一个异步返回并检查返回值是否匹配.

The basic case for &lt;pattern&gt; is a variable name, the result of the async expression is bound to the
variable name and &lt;handler&gt; has access to that variable. This is why, in the original example, val
was used for &lt;pattern&gt; and &lt;handler&gt; was able to access val.

If &lt;pattern&gt; does not match the result of the async computation, then the remaining async
expressions `continue to execute concurrently` until the next one completes. At this time, the same
logic is applied to that result.

Because select! takes any async expression, it is possible to define more complicated computations
to select on.

```rust
use tokio::net::TcpStream;
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    // Spawn a task that sends a message over the oneshot
    tokio::spawn(async move {
        tx.send("done").unwrap();
    });

    tokio::select! {
        socket = TcpStream::connect("localhost:3465") => {
            println!("Socket connected {:?}", socket);
        }
        msg = rx => {
            println!("received message first {:?}", msg);
        }
    }
}

// 另一个例子
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        tx.send(()).unwrap();
    });

    let mut listener = TcpListener::bind("localhost:3465").await?;

    // 并发执行两个 branch, 直到 rx 收到消息或者 accept 出错.
    tokio::select! {
        _ = async {
            loop {
                let (socket, _) = listener.accept().await?; // listener 出错时才返回
                tokio::spawn(async move { process(socket) });
            }

            // Help the rust type inferencer out
            Ok::<_, io::Error>(())
        } => {}
        _ = rx => {
            println!("terminating accept loop");
        }
    }

    Ok(())
}
```

select 宏有返回值, 各 branch 必须返回相同类型值:

```rust
async fn computation1() -> String {
    // .. computation
}

async fn computation2() -> String {
    // .. computation
}

#[tokio::main]
async fn main() {
    let out = tokio::select! {
        res1 = computation1() => res1,
        res2 = computation2() => res2,
    };

    println!("Got = {}", out);
}
```

Errors:

-   如果 async exprssion 中有 ? 则该表达式返回 Result::Err, 例如 accept().await? 出错时, res 为
    Result::Err;
-   如果是 handler 中有 ?, 则会立即向 error 传递到 select 表达式外部, 如 res? 会将错误传递到 main() 函数;

<!--listend-->

```rust
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    // [setup `rx` oneshot channel]

    let listener = TcpListener::bind("localhost:3465").await?;

    tokio::select! {
        res = async {
            loop {
                let (socket, _) = listener.accept().await?;
                tokio::spawn(async move { process(socket) });
            }

            // Help the rust type inferencer out
            Ok::<_, io::Error>(())
        } => {
            res?;
        }
        _ = rx => {
            println!("terminating accept loop");
        }
    }

    Ok(())
}
```

async expression 返回值支持 pattern match, 并且可以有一个 else 分支:

-   当所有 pattern match 都不匹配时, 才执行 else 分支.

<!--listend-->

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (mut tx1, mut rx1) = mpsc::channel(128);
    let (mut tx2, mut rx2) = mpsc::channel(128);

    tokio::spawn(async move {
        // Do something w/ `tx1` and `tx2`
    });

    tokio::select! {
        Some(v) = rx1.recv() => {
            println!("Got {:?} from rx1", v);
        }
        Some(v) = rx2.recv() => {
            println!("Got {:?} from rx2", v);
        }
        else => {
            println!("Both channels closed");
        }
    }
}
```

Borrowing: 对于 tokio::spawn(async {..}) 必须 move 捕获要使用的数据, 但是对于 select 的多 branch
async expression 则不需要, 只要遵守 borrow 的规则即可, 比如同时访问 &amp; 共享数据, 唯一访问 &amp;mut 数据:

-   The `data` variable is being borrowed immutably from both async expressions. When one of the
    operations completes successfully, the other one is dropped. Because we pattern match on Ok(_), if
    an expression fails, the other one `continues to execute`.
    -   select 必须异步并发执行所有 aysnc expression, 并且当第一个 expression 返回的结果匹配 pattern 时才 drop 其他正在执行的 async expression.
-   When it comes to each branch's &lt;handler&gt;, select! guarantees that `only a single <handler> runs`
    . Because of this, each &lt;handler&gt; `may mutably borrow the same data`.

<!--listend-->

```rust
use tokio::io::AsyncWriteExt;
use tokio::net::TcpStream;
use std::io;
use std::net::SocketAddr;

async fn race(
    data: &[u8],
    addr1: SocketAddr,
    addr2: SocketAddr
) -> io::Result<()> {
    tokio::select! {
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr1).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr2).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        else => {}
    };

    Ok(())
}

// 由于 select 保证只执行一个 handler, 所以多个 handler 中可以 &mut 使用相同的数据.
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    let mut out = String::new();

    tokio::spawn(async move {
        // Send values on `tx1` and `tx2`.
    });

    tokio::select! {
        _ = rx1 => {
            out.push_str("rx1 completed");
        }
        _ = rx2 => {
            out.push_str("rx2 completed");
        }
    }

    println!("{}", out);
}
```

Loop:

-   并发执行多个 aysnc expression, 当他们都 recv() 返回时, select 随机选择一个 branch 来执行
    handler. 未执行的 handler branch 的 message 也不会丢.

<!--listend-->

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx1, mut rx1) = mpsc::channel(128);
    let (tx2, mut rx2) = mpsc::channel(128);
    let (tx3, mut rx3) = mpsc::channel(128);

    loop {
	    // 当 rx1.recv() 返回时, drop 的是 rx2.recv() 和 rx3.recv() 的 Future 对象, 而不是 rx2和
	    // rx3,所以下一次 loop 还可以继续调用.

	    // 当 channel 被 close , .recv() 返回 None, 不匹配 Some, 所以其他 rx 还是继续进行, 直到
	    // 所有 rx 都被关闭.

	    // 注意: std::sync::mpsc 的 Receiver.recv() 返回 Result, 而 async 的返回 Option
        let msg = tokio::select! {
            Some(msg) = rx1.recv() => msg,
            Some(msg) = rx2.recv() => msg,
            Some(msg) = rx3.recv() => msg,
            else => { break }
        };

        println!("Got {:?}", msg);
    }

    println!("All channels have been closed.");
}
```

重用 aysnc 操作:

-   必须先 tokio::pin!(operation), 该宏将返回同名的变量 operation , 但是类型已经时 Pin 类型.
-   再在 async expression 中 &amp;mut 借用;

<!--listend-->

```rust
async fn action() {
    // Some asynchronous logic
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);

    let operation = action(); // 返回一个 Future 对象, 没有对它 .await
    tokio::pin!(operation);  // 必须 Pin 该 Future 对象

    loop {
        tokio::select! {
	        // BUG:  operation 一旦完成, 就不能被重用. 需要使用  if precheck 条件来避免.
            _ = &mut operation => break, // 重用该 Future 对象, 而不是调用 action(). 同时必须是 &mut 借用.
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    break;
                }
            }
        }
    }
}
```

这里必须使用 tokio::pin!(operation) 来 Pin Future 对象(返回一个同名的但是类型时 Pin 的对象), 否则报错:

-   std::pin::Pin&lt;&amp;mut &amp;mut impl std::future::Future&gt; 的解释:
    -   &amp;mut impl std::future::Future 是一个实现 Future trait 的 &amp;mut 引用;
    -   &amp;mut &amp;mut impl std::future::Future 是对 &amp;mut 的 &amp;mut 引用;

<!--listend-->

```rust
error[E0599]: no method named `poll` found for struct
     `std::pin::Pin<&mut &mut impl std::future::Future>`
     in the current scope
  --> src/main.rs:16:9
   |
16 | /         tokio::select! {
17 | |             _ = &mut operation => break,
18 | |             Some(v) = rx.recv() => {
19 | |                 if v % 2 == 0 {
...  |
22 | |             }
23 | |         }
   | |_________^ method not found in
   |             `std::pin::Pin<&mut &mut impl std::future::Future>`
   |
   = note: the method `poll` exists but the following trait bounds
            were not satisfied:
           `impl std::future::Future: std::marker::Unpin`
           which is required by
           `&mut impl std::future::Future: std::future::Future`
```

Modifying a branch

-   async fn 一旦 completion 就不能再 resume 执行. 一般情况下, future.await? 会消耗 future 对象.
-   branch 可以使用 if 表达式先判断是否需要执行  &amp;mut operation, if !done =&gt; {xxx};

<!--listend-->

```rust
async fn action(input: Option<i32>) -> Option<String> {
    // If the input is `None`, return `None`.
    // This could also be written as `let i = input?;`
    let i = match input {
        Some(input) => input,
        None => return None,
    };
    // async logic here
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);

    let mut done = false;
    let operation = action(None);
    tokio::pin!(operation);

    tokio::spawn(async move {
        let _ = tx.send(1).await;
        let _ = tx.send(3).await;
        let _ = tx.send(2).await;
    });

    loop {
        tokio::select! {
             // 先判断 done, 为 true 时才执行 async expression. 这里使用的是 &mut 引用, 多以 .await
	         // 并不会消耗对象, operation 在下一次 loop 还存在.
            res = &mut operation, if !done => {
                done = true;

                if let Some(v) = res {
                    println!("GOT = {}", v);
                    return;
                }
            }
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    // `.set` is a method on `Pin`.
                    operation.set(action(Some(v)));
                    done = false;
                }
            }
        }
    }
}
```

tokio::spawn 和 select! 的区别:

1.  Both tokio::spawn and select! enable `running concurrent asynchronous operations`. However, the
    strategy used to run concurrent operations differs. The `tokio::spawn` function takes an
    asynchronous operation and `spawns a new task` to run it. A task is the object that the Tokio
    runtime schedules. Two different tasks are `scheduled independently` by Tokio. They may run
    simultaneously on `different operating system threads`. Because of this, a spawned task has the
    same restriction as a spawned thread: `no borrowing`.
2.  The select! macro runs all branches `concurrently on the same task`. Because all branches of the
    select! macro are executed on the same task, they will `never run simultaneously`. The select!
    macro multiplexes asynchronous operations `on a single task`.


## <span class="section-num">12</span> stream {#stream}

Stream 是同步 std::iter::Iterator 的异步版本, 返回一系列 value, 它是 `futures` crate 定义的 Stream
trait 类型:

-   Stream 可以在 async fn 里进行迭代, 也可以使用类似于 Iterator 的 adapter(tokio 的 StreamExt trait
    提供);
-   tokio 在单独的 tokio-stream create 中提供 Stream 支持功能.

<!--listend-->

```rust
// https://docs.rs/futures-core/0.3.30/futures_core/stream/trait.Stream.html
pub trait Stream {
    type Item;

    // Required method
    fn poll_next(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;

    // Provided method
    fn size_hint(&self) -> (usize, Option<usize>) { ... }
}
```

迭代 Stream:

-   tokio_stream::StreamExt 是 futures_core::stream::Stream 子 trait, 提供了额外的 trait 方法, 包括:
    各种 adapter 方法(如 map/filter 等), 以及用于迭代的 next() 方法;

<!--listend-->

```rust
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let mut stream = tokio_stream::iter(&[1, 2, 3]);  // iter 返回的 stream 实现了 Unpin, 故不需要明确的 Pin 操作

    while let Some(v) = stream.next().await { // None 表示迭代终止
        println!("GOT = {:?}", v);
    }
}
```

复杂的例子:

```rust
use tokio_stream::StreamExt;
use mini_redis::client;

async fn publish() -> mini_redis::Result<()> {
    let mut client = client::connect("127.0.0.1:6379").await?;

    // Publish some data
    client.publish("numbers", "1".into()).await?;
    client.publish("numbers", "two".into()).await?;
    client.publish("numbers", "3".into()).await?;
    client.publish("numbers", "four".into()).await?;
    client.publish("numbers", "five".into()).await?;
    client.publish("numbers", "6".into()).await?;
    Ok(())
}

async fn subscribe() -> mini_redis::Result<()> {
    let client = client::connect("127.0.0.1:6379").await?;
    let subscriber = client.subscribe(vec!["numbers".to_string()]).await?;
    let messages = subscriber.into_stream(); // 返回一个 Stream 对象

    // tokio_stream::iter() 返回的 Stream 实现了 Unpin trait, 不需要明确的 Pin 操作
    tokio::pin!(messages); // Pin 到 stack

    while let Some(msg) = messages.next().await { // next 要求 message Stream 必须时 Pinned
        println!("got = {:?}", msg);
    }

    Ok(())
}

#[tokio::main]
async fn main() -> mini_redis::Result<()> {
    tokio::spawn(async {
        publish().await
    });

    subscribe().await?;

    println!("DONE");

    Ok(())
}
```

A Rust value is "pinned" when it can `no longer be moved in memory`. A key property of a pinned value
is that pointers can be taken to the pinned data and the caller can be confident `the pointer stays
 valid`. This feature is used by `async/await` to support `borrowing data across .await points`.

-   因为 async fn 内的 data 时保存在 stack 上的, 但 across .await 使用这些 data 时, 在 .await 位置,
    async fn 执行可能会暂停或被调度到其他 thread 上运行, 所以生成的 Future 对象会保存这些 data 在
    stack 上的指针, 并且确保这些 data 时 Pin/Send/Sync 类型的.

Adapters 指的是从 Stream 生成新的 Stream, 如 map/take/filter;

-   tokio_stream::StreamExt 是 futures_core::stream::Stream 子 trait, 提供了额外的 trait 方法, 包括:
    各种 adapter 方法(如 map/filter 等), 以及用于迭代的 next() 方法;

<!--listend-->

```rust
let messages = subscriber
    .into_stream()
    .take(3);

let messages = subscriber
    .into_stream()
    .filter(|msg| match msg {
        Ok(msg) if msg.content.len() == 1 => true,
        _ => false,
    })
    .take(3);

let messages = subscriber
    .into_stream()
    .filter(|msg| match msg {
        Ok(msg) if msg.content.len() == 1 => true,
        _ => false,
    })
    .map(|msg| msg.unwrap().content)
    .take(3);
```

实现一个 Stream:

-   能返回值时, poll_next() 返回 Poll::Ready, 否则返回 Poll::Pending;
-   一般在 poll_next() 内部需要使用 Future 和其他 Stream 来实现;

<!--listend-->

```rust
use tokio_stream::Stream;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

struct Interval {
    rem: usize,
    delay: Delay,
}

impl Interval {
    fn new() -> Self {
        Self {
            rem: 3,
            delay: Delay { when: Instant::now() }
        }
    }
}

impl Stream for Interval {
    type Item = ();

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<()>>
    {
        if self.rem == 0 {
            // No more delays
            return Poll::Ready(None);
        }

        match Pin::new(&mut self.delay).poll(cx) {
            Poll::Ready(_) => {
                let when = self.delay.when + Duration::from_millis(10);
                self.delay = Delay { when };
                self.rem -= 1;
                Poll::Ready(Some(()))
            }
            Poll::Pending => Poll::Pending,
        }
    }
}
```

Rust 的 async/await 还不支持定义 Stream, 但是 async-stream crate 提供了 stream! 宏,可以用来创建 Stream:

```rust
use async_stream::stream;
use std::time::{Duration, Instant};

stream! {
    let mut when = Instant::now();
    for _ in 0..3 {
        let delay = Delay { when };
        delay.await;
        yield (); // yield 返回 Stream 可以迭代的值.
        when += Duration::from_millis(10);
    }
}
```


## <span class="section-num">13</span> 同步函数中执行异步代码 {#同步函数中执行异步代码}

\#[tokio::main] 在一个 multi thread executor 中指定 async main 函数代码:

```rust
#[tokio::main]
async fn main() {
    println!("Hello world");
}

// 等效为
fn main() {
    tokio::runtime::Builder::new_multi_thread() // 默认是多线程 executor
        .enable_all()
        .build()
        .unwrap()
        .block_on(async { // block_on() 用来执行异步代码, 同步返回结果. 是同步到异步的入口.
            println!("Hello world");
        })
}
```

tokio::task::spawn_blocking() 是立即创建一个新 thread 来执行可能会 block 的 `同步代码`, 并等待直到执行返回, 一般用于 CPU 密集型任务: &lt;-- 同步

-   tokio::task::spawn() 是创建一个异步 task 然后发送给 executor 的线程池来执行, 并不会创建新线程, 而且是立即返回一个可以 .await 的 JoinHandle; &lt;-- 异步
-   tokio::task::block_on() 是立即执行当前 async fn 并 `等待直到执行返回` . &lt;-- 同步

<!--listend-->

```rust
use tokio::task;

// Initial input
let mut v = "Hello, ".to_string();
let res = task::spawn_blocking(move || { // 执行一部分同步代码的闭包
    // Stand-in for compute-heavy work or using synchronous APIs
    v.push_str("world");
    // Pass ownership of the value back to the asynchronous context
    v
}).await?;

// `res` is the value returned from the thread
assert_eq!(res.as_str(), "Hello, world");
```

通过封装一个 tokio::runtime::Runtime 和使用他的 block_on 方法, 可以将异步代码封装为同步代码:

-   tokio::runtime::Runtime 是 current_thread 类型, `该类型不会创建新的 thread` , 而是在当前 thread 执行
    rt.block_on(async fn) 方法, 直到 async fn 结束返回, 返回异步函数执行的结果.
-   如果调用 current_thread executor 的 spawn() 方法, 则并不会立即在后台执行, 而是在 `后续调用
        block_on() 时才会执行` .

<!--listend-->

```rust
use tokio::net::ToSocketAddrs;
use tokio::runtime::Runtime;

pub use crate::clients::client::Message;

/// Established connection with a Redis server.
pub struct BlockingClient {
    /// The asynchronous `Client`.
    inner: crate::clients::Client,

    /// A `current_thread` runtime for executing operations on the asynchronous client in a blocking
    /// manner.
    rt: Runtime,
}

impl BlockingClient {
    pub fn connect<T: ToSocketAddrs>(addr: T) -> crate::Result<BlockingClient> {
        let rt = tokio::runtime::Builder::new_current_thread()
            .enable_all()
            .build()?;

        // Call the asynchronous connect method using the runtime.
        let inner = rt.block_on(crate::clients::Client::connect(addr))?;

        Ok(BlockingClient { inner, rt })
    }
}

// 其他使用 selfrt.block_on() 封装的同步方法
use bytes::Bytes;
use std::time::Duration;

impl BlockingClient {
    pub fn get(&mut self, key: &str) -> crate::Result<Option<Bytes>> {
        self.rt.block_on(self.inner.get(key))
    }

    pub fn set(&mut self, key: &str, value: Bytes) -> crate::Result<()> {
        self.rt.block_on(self.inner.set(key, value))
    }

    pub fn set_expires(
        &mut self,
        key: &str,
        value: Bytes,
        expiration: Duration,
    ) -> crate::Result<()> {
        self.rt.block_on(self.inner.set_expires(key, value, expiration))
    }

    pub fn publish(&mut self, channel: &str, message: Bytes) -> crate::Result<u64> {
        self.rt.block_on(self.inner.publish(channel, message))
    }
}

/// A client that has entered pub/sub mode.
///
/// Once clients subscribe to a channel, they may only perform
/// pub/sub related commands. The `BlockingClient` type is
/// transitioned to a `BlockingSubscriber` type in order to
/// prevent non-pub/sub methods from being called.
pub struct BlockingSubscriber {
    /// The asynchronous `Subscriber`.
    inner: crate::clients::Subscriber,

    /// A `current_thread` runtime for executing operations on the
    /// asynchronous client in a blocking manner.
    rt: Runtime,
}

impl BlockingClient {
    pub fn subscribe(self, channels: Vec<String>) -> crate::Result<BlockingSubscriber> {
        let subscriber = self.rt.block_on(self.inner.subscribe(channels))?;
        Ok(BlockingSubscriber {
            inner: subscriber,
            rt: self.rt,
        })
    }
}

impl BlockingSubscriber {
    pub fn get_subscribed(&self) -> &[String] {
        self.inner.get_subscribed() // 同步方法调用
    }

    pub fn next_message(&mut self) -> crate::Result<Option<Message>> {
        self.rt.block_on(self.inner.next_message()) // 调用 subscribe() 方法后, 连续调用 next_message() 返回消息
    }

    pub fn subscribe(&mut self, channels: &[String]) -> crate::Result<()> {
        self.rt.block_on(self.inner.subscribe(channels))
    }

    pub fn unsubscribe(&mut self, channels: &[String]) -> crate::Result<()> {
        self.rt.block_on(self.inner.unsubscribe(channels))
    }
}
```

总结下, 在同步代码中调用异步代码的方法:

1.  Create a Runtime and call block_on on the async code.
2.  Create a Runtime and spawn things on it.
3.  Run the Runtime in a separate thread and send messages to it.

Spawning things on a runtime

-   tokio::task::spawn() 是创建一个异步 task 然后发送给 executor 的线程池来执行, 并不会创建新线程, 而且是立即返回一个可以 .await 的 JoinHandle; &lt;-- 异步
-   除了使用 runtime.block_on(handle).unwrap();来等待异步任务结束, 还可以使用其他机制在异步和同步代码间传递结束信息.

    -   Use a message passing channel such as tokio::sync::mpsc.
    -   Modify a shared value protected by e.g. a Mutex. This can be a good approach for a progress bar
        in a GUI, where the GUI reads the shared value every frame.

    <!--listend-->

    ```rust
        use tokio::runtime::Builder;
        use tokio::time::{sleep, Duration};

        fn main() {
    	let runtime = Builder::new_multi_thread() // 多线程 executor
                .worker_threads(1) // 只有一个 worker 线程(实际还有一个执行 main 的主线程)
                .enable_all()
                .build()
                .unwrap();

    	let mut handles = Vec::with_capacity(10);
    	// 创建 10 个后台执行的异步任务
    	for i in 0..10 {
      	      // runtime.spawn 是调度一个后台执行的异步任务, 返回 JoinHandle
                handles.push(runtime.spawn(my_bg_task(i)));
    	}

    	// Do something time-consuming while the background tasks execute.
    	std::thread::sleep(Duration::from_millis(750));
    	println!("Finished time-consuming task.");

    	// Wait for all of them to complete.
    	for handle in handles {
                // The `spawn` method returns a `JoinHandle`. A `JoinHandle` is a future, so we can wait for
                // it using `block_on`.
      	      // JoinHandle 是实现 Future 的异步对象, 可以 .await(在异步函数中) 或 runtime.block_on() 来等待返回.
                runtime.block_on(handle).unwrap();
    	}
        }

        async fn my_bg_task(i: u64) {
    	// By subtracting, the tasks with larger values of i sleep for a
    	// shorter duration.
    	let millis = 1000 - 50 * i;
    	println!("Task {} sleeping for {} ms.", i, millis);

    	sleep(Duration::from_millis(millis)).await;

    	println!("Task {} stopping.", i);
        }
    ```

Sending messages:

-   改进: 使用 tokio Semaphore 来限制可以提交的 active task 数量;

<!--listend-->

```rust
use tokio::runtime::Builder;
use tokio::sync::mpsc;

pub struct Task {
    name: String,
    // info that describes the task
}

async fn handle_task(task: Task) {
    println!("Got task {}", task.name);
}

#[derive(Clone)]
pub struct TaskSpawner {
    spawn: mpsc::Sender<Task>,
}

impl TaskSpawner {
    pub fn new() -> TaskSpawner {
        // Set up a channel for communicating.
        let (send, mut recv) = mpsc::channel(16);

        // Build the runtime for the new thread.
        //
        // The runtime is created before spawning the thread to more cleanly forward errors if the
        // `unwrap()` panics.
        let rt = Builder::new_current_thread()
            .enable_all()
            .build()
            .unwrap();

        std::thread::spawn(move || { // 单起一个 thread 来执行多线程的 executor
            rt.block_on(async move { // executor.block_on(async fn)  来运行异步代码
                while let Some(task) = recv.recv().await {
                    tokio::spawn(handle_task(task)); // spawn() 产生 rt 并发执行的异步任务
                }

                // Once all senders have gone out of scope, the `.recv()` call returns None and it
                // will exit from the while loop and shut down the thread.
            });
        });

        TaskSpawner {
            spawn: send,
        }
    }

    pub fn spawn_task(&self, task: Task) {
        match self.spawn.blocking_send(task) { // mpsc::Sender.blocking_send() 是发送任务的 同步函数
            Ok(()) => {},
            Err(_) => panic!("The shared runtime has shut down."),
        }
    }
}
```


## <span class="section-num">14</span> Actors with Tokio {#actors-with-tokio}

<https://ryhl.io/blog/actors-with-tokio/>

This article is about building actors with Tokio directly, without using any actor libraries such as
Actix. This turns out to be rather easy to do, however there are some details you should be aware
of:

-   Where to put the tokio::spawn call.
-   Struct with run method vs bare function.
-   Handles to the actor.
-   Backpressure and bounded channels.
-   Graceful shutdown.

The techniques outlined in this article should work with any executor, but for simplicity we will
only talk about Tokio. There is some overlap with the spawning and channel chapters from the Tokio
tutorial, and I recommend also reading those chapters.

Before we can talk about how to write an actor, we need to know `what an actor is`. The basic idea
behind an actor is to `spawn a self-contained task that performs some job independently` of other
parts of the program. Typically these actors communicate with the rest of the program through the
use of `message passing channels`. Since each actor runs independently, programs designed using them
are `naturally parallel`.

actor 是自包含的独立任务(分配独立的部分资源或任务), 使用 message channel 和程序其他部分通信, 天然支持并发.

A common use-case of actors is to assign the actor `exclusive ownership of some resource` you want to
share, and then let other tasks access this resource indirectly by `talking to the actor`. For
example, if you are implementing a chat server, you may spawn a task for each connection, and a
master task that routes chat messages between the other tasks. This is useful because the master
task can avoid having to deal with network IO, and the connection tasks can focus exclusively on
dealing with network IO.

This article is also available as a talk on YouTube.

The Recipe

An actor is split into two parts: `the task and the handle`. The task is the independently spawned
Tokio task that actually performs the duties of the actor, and the handle is a struct that allows
you to communicate with the task.

Let's consider a simple actor. The actor internally stores a counter that is used to obtain some
sort of unique id. The basic structure of the actor would be something like the following:

```rust
use tokio::sync::{oneshot, mpsc};

struct MyActor {
    receiver: mpsc::Receiver<ActorMessage>,
    next_id: u32,
}
enum ActorMessage {
    GetUniqueId {
        respond_to: oneshot::Sender<u32>,
    },
}

impl MyActor {
    fn new(receiver: mpsc::Receiver<ActorMessage>) -> Self {
        MyActor {
            receiver,
            next_id: 0,
        }
    }
    fn handle_message(&mut self, msg: ActorMessage) {
        match msg {
            ActorMessage::GetUniqueId { respond_to } => {
                self.next_id += 1;

                // The `let _ =` ignores any errors when sending.
                //
                // This can happen if the `select!` macro is used
                // to cancel waiting for the response.
                let _ = respond_to.send(self.next_id);
            },
        }
    }
}

async fn run_my_actor(mut actor: MyActor) {
    while let Some(msg) = actor.receiver.recv().await {
        actor.handle_message(msg);
    }
}
```

Now that we have the actor itself, we also need a handle to the actor. A handle is an object that
other pieces of code can use to talk to the actor, and is also what keeps the actor alive.

The handle will look like this:

```rust
#[derive(Clone)]
pub struct MyActorHandle {
    sender: mpsc::Sender<ActorMessage>,
}

impl MyActorHandle {
    pub fn new() -> Self {
        let (sender, receiver) = mpsc::channel(8);
        let actor = MyActor::new(receiver);
        tokio::spawn(run_my_actor(actor));

        Self { sender }
    }

    pub async fn get_unique_id(&self) -> u32 {
        let (send, recv) = oneshot::channel();
        let msg = ActorMessage::GetUniqueId {
            respond_to: send,
        };

        // Ignore send errors. If this send fails, so does the
        // recv.await below. There's no reason to check for the
        // same failure twice.
        let _ = self.sender.send(msg).await;
        recv.await.expect("Actor task has been killed")
    }
}
```

full example

Let's take a closer look at the different pieces in this example.

ActorMessage. The ActorMessage enum defines the kind of messages we can send to the actor. By using an enum, we can have many different message types, and each message type can have its own set of arguments. We return a value to the sender by using an oneshot channel, which is a message passing channel that allows sending exactly one message.

In the example above, we match on the enum inside a handle_message method on the actor struct, but that isn't the only way to structure this. One could also match on the enum in the run_my_actor function. Each branch in this match could then call various methods such as get_unique_id on the actor object.

Errors when sending messages. When dealing with channels, not all errors are fatal. Because of this, the example sometimes uses let _ = to ignore errors. Generally a send operation on a channel fails if the receiver has been dropped.

The first instance of this in our example is the line in the actor where we respond to the message we were sent. This can happen if the receiver is no longer interested in the result of the operation, e.g. if the task that sent the message might have been killed.

Shutdown of actor. We can detect when the actor should shut down by looking at failures to receive messages. In our example, this happens in the following while loop:

while let Some(msg) = actor.receiver.recv().await {
    actor.handle_message(msg);
}

When all senders to the receiver have been dropped, we know that we will never receive another message and can therefore shut down the actor. When this happens, the call to .recv() returns None, and since it does not match the pattern Some(msg), the while loop exits and the function returns.

\#[derive(Clone)] The MyActorHandle struct derives the Clone trait. It can do this because mpsc means that it is a multiple-producer, single-consumer channel. Since the channel allows multiple producers, we can freely clone our handle to the actor, allowing us to talk to it from multiple places.
A run method on a struct

The example I gave above uses a top-level function that isn't defined on any struct as the thing we spawn as a Tokio task, however many people find it more natural to define a run method directly on the MyActor struct and spawn that. This certainly works too, but the reason I give an example that uses a top-level function is that it more naturally leads you towards the approach that doesn't give you lots of lifetime issues.

To understand why, I have prepared an example of what people unfamiliar with the pattern often come up with.

impl MyActor {
    fn run(&amp;mut self) {
        tokio::spawn(async move {
            while let Some(msg) = self.receiver.recv().await {
                self.handle_message(msg);
            }
        });
    }

pub async fn get_unique_id(&amp;self) -&gt; u32 {
    let (send, recv) = oneshot::channel();
    let msg = ActorMessage::GetUniqueId {
	respond_to: send,
    };

        _/ Ignore send errors. If this send fails, so does the
        /_ recv.await below. There's no reason to check for the
        // same failure twice.
        let _ = self.sender.send(msg).await;
        recv.await.expect("Actor task has been killed")
    }
}

... and no separate MyActorHandle

The two sources of trouble in this example are:

The tokio::spawn call is inside run.
The actor and the handle are the same struct.

The first issue causes problems because the tokio::spawn function requires the argument to be 'static. This means that the new task must own everything inside it, which is a problem because the method borrows self, meaning that it is not able to give away ownership of self to the new task.

The second issue causes problems because Rust enforces the single-ownership principle. If you combine both the actor and the handle into a single struct, you are (at least from the compiler's perspective) giving every handle access to the fields owned by the actor's task. E.g. the next_id integer should be owned only by the actor's task, and should not be directly accessible from any of the handles.

That said, there is a version that works. By fixing the two above problems, you end up with the following:

impl MyActor {
    async fn run(&amp;mut self) {
        while let Some(msg) = self.receiver.recv().await {
            self.handle_message(msg);
        }
    }
}

impl MyActorHandle {
    pub fn new() -&gt; Self {
        let (sender, receiver) = mpsc::channel(8);
        let actor = MyActor::new(receiver);
        tokio::spawn(async move { actor.run().await });

        Self { sender }
    }
}

This works identically to the top-level function. Note that, strictly speaking, it is possible to write a version where the tokio::spawn is inside run, but I don't recommend that approach.
Variations on the theme

The actor I used as an example in this article uses the request-response paradigm for the messages, but you don't have to do it this way. In this section I will give some inspiration to how you can change the idea.
No responses to messages

The example I used to introduce the concept includes a response to the messages sent over a oneshot channel, but you don't always need a response at all. In these cases there's nothing wrong with just not including the oneshot channel in the message enum. When there's space in the channel, this will even allow you to return from sending before the message has been processed.

You should still make sure to use a bounded channel so that the number of messages waiting in the channel don't grow without bound. In some cases this will mean that sending still needs to be an async function to handle the cases where the send operation needs to wait for more space in the channel.

However there is an alternative to making send an async method. You can use the try_send method, and handle sending failures by simply killing the actor. This can be useful in cases where the actor is managing a TcpStream, forwarding any messages you send into the connection. In this case, if writing to the TcpStream can't keep up, you might want to just close the connection.
Multiple handle structs for one actor

If an actor needs to be sent messages from different places, you can use multiple handle structs to enforce that some message can only be sent from some places.

When doing this you can still reuse the same mpsc channel internally, with an enum that has all the possible message types in it. If you do want to use separate channels for this purpose, the actor can use tokio::select! to receive from multiple channels at once.

loop {
    tokio::select! {
        Some(msg) = chan1.recv() =&gt; {
            _/ handle msg
        },
        Some(msg) = chan2.recv() =&gt; {
            /_ handle msg
        },
        else =&gt; break,
    }
}

You need to be careful with how you handle when the channels are closed, as their recv method immediately returns None in this case. Luckily the tokio::select! macro lets you handle this case by providing the pattern Some(msg). If only one channel is closed, that branch is disabled and the other channel is still received from. When both are closed, the else branch runs and uses break to exit from the loop.
Actors sending messages to other actors

There is nothing wrong with having actors send messages to other actors. To do this, you can simply give one actor the handle of some other actor.

You need to be a bit careful if your actors form a cycle, because by holding on to each other's handle structs, the last sender is never dropped, preventing shutdown. To handle this case, you can have one of the actors have two handle structs with separate mpsc channels, but with a tokio::select! that looks like this:

loop {
    tokio::select! {
        opt_msg = chan1.recv() =&gt; {
            let msg = match opt_msg {
                Some(msg) =&gt; msg,
                None =&gt; break,
            };
            _/ handle msg
        },
        Some(msg) = chan2.recv() =&gt; {
            /_ handle msg
        },
    }
}

The above loop will always exit if chan1 is closed, even if chan2 is still open. If chan2 is the channel that is part of the actor cycle, this breaks the cycle and lets the actors shut down.

An alternative is to simply call abort on one of the actors in the cycle.
Multiple actors sharing a handle

Just like you can have multiple handles per actor, you can also have multiple actors per handle. The most common example of this is when handling a connection such as a TcpStream, where you commonly spawn two tasks: one for reading and one for writing. When using this pattern, you make the reading and writing tasks as simple as you can — their only job is to do IO. The reader task will just send any messages it receives to some other task, typically another actor, and the writer task will just forward any messages it receives to the connection.

This pattern is very useful because it isolates the complexity associated with performing IO, meaning that the rest of the program can pretend that writing something to the connection happens instantly, although the actual writing happens sometime later when the actor processes the message.
Beware of cycles

I already talked a bit about cycles under the heading “Actors sending messages to other actors”, where I discussed shutdown of actors that form a cycle. However, shutdown is not the only problem that cycles can cause, because a cycle can also result in a deadlock where each actor in the cycle is waiting for the next actor to receive a message, but that next actor wont receive that message until its next actor receives a message, and so on.

To avoid such a deadlock, you must make sure that there are no cycles of channels with bounded capacity. The reason for this is that the send method on a bounded channel does not return immediately. Channels whose send method always returns immediately do not count in this kind of cycle, as you cannot deadlock on such a send.

Note that this means that a oneshot channel cannot be part of a deadlocked cycle, since their send method always returns immediately. Note also that if you are using try_send rather than send to send the message, that also cannot be part of the deadlocked cycle.

Thanks to matklad for pointing out the issues with cycles and deadlocks.


## <span class="section-num">15</span> Graceful Shutdown {#graceful-shutdown}

<https://tokio.rs/tokio/topics/shutdown>

要是先 Graceful Shutdown, 包括三方面:

1.  Figuring out when to shut down.
2.  Telling every part of the program to shut down.
3.  Waiting for other parts of the program to shut down.

Figuring out when to shut down:

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    // ... spawn application as separate task ...

    // l sleep until such a signal is received.
    match signal::ctrl_c().await {
        Ok(()) => {},
        Err(err) => {
            eprintln!("Unable to listen for shutdown signal: {}", err);
            // we also shut down in case of error
        },
    }

    // send shutdown signal to application and wait
}

// 或者更复杂的场景, 有多个 shutdown conditions 时可以使用 mpsc channel
use tokio::signal;
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (shutdown_send, mut shutdown_recv) = mpsc::unbounded_channel();

    // ... spawn application as separate task ...
    //
    // application uses shutdown_send in case a shutdown was issued from inside
    // the application

    tokio::select! {
        _ = signal::ctrl_c() => {},
        _ = shutdown_recv.recv() => {},
    }

    // send shutdown signal to application and wait
}
```

Telling things to shut down

-   When you want to tell one or more tasks to shut down, you can use `Cancellation` Tokens.

<!--listend-->

```rust
// Step 1: Create a new CancellationToken
let token = CancellationToken::new();

// Step 2: Clone the token for use in another task
let cloned_token = token.clone();  // 可以被 clone 复用

// Task 1 - Wait for token cancellation or a long time
let task1_handle = tokio::spawn(async move {
    tokio::select! {
        // Step 3: Using cloned token to listen to cancellation requests
	    // 等待 token 被 cancel
        _ = cloned_token.cancelled() => {
            // The token was cancelled, task can shut down
        }
        _ = tokio::time::sleep(std::time::Duration::from_secs(9999)) => {
            // Long work has completed
        }
    }
});

// Task 2 - Cancel the original token after a small delay
tokio::spawn(async move {
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;

    // Step 4: Cancel the original or cloned token to notify other tasks about shutting down gracefully
    token.cancel();  // cancel 所有 token (包括 clone 的实例)
});

// Wait for tasks to complete  // 如果要等待多个 task 都因为 cancel 返回, 可以使用 tokio_util::task::TaskTracker;
task1_handle.await.unwrap()
```

Waiting for things to finish shutting down:

-   Once you have told other tasks to shut down, you will need to `wait for them to finish`. One easy
    way to do so is to use a `task tracker`. A task tracker is a collection of tasks.
-   tracker.close() 并不会主动 kill 或停止签名 spawn 异步任务, 异步任务需要自己主动执行结束, 当所有
    spawd 的 task 都执行结束后, wait() 才会返回;
-   所以 TaskTracker 一般是和 CancelllationToken 结合使用.

<!--listend-->

```rust
use std::time::Duration;
use tokio::time::sleep;
use tokio_util::task::TaskTracker;

#[tokio::main]
async fn main() {
    let tracker = TaskTracker::new();

    for i in 0..10 { // 使用 tacker 代替 tokio::spawn/blocking_on/spawn_blocking() 来提交异步任务
        tracker.spawn(some_operation(i));
    }

    // Once we spawned everything, we close the tracker.
    tracker.close(); // 关闭 tracker

    // Wait for everything to finish.
    tracker.wait().await; // 等待 tracker spwan 的所有任务结束

    println!("This is printed after all of the tasks.");
}

async fn some_operation(i: u64) {
    sleep(Duration::from_millis(100 * i)).await;
    println!("Task {} shutting down.", i);
}
```


## <span class="section-num">16</span> unit testing {#unit-testing}

<https://tokio.rs/tokio/topics/testing>

`#[tokio::test]` 默认创建一个 current_thread runtime.

Pausing and resuming time in tests

-   pause time: The current value of Instant::now() is saved and all subsequent calls to
    Instant::now() will return the saved value. The saved value can be changed by advance or by the
    time auto-advancing once the runtime has no work to do. This only affects the Instant type in
    Tokio, and the Instant in std continues to work as normal.

<!--listend-->

```rust
#[tokio::test]
async fn paused_time() {
    tokio::time::pause();
    let start = std::time::Instant::now();
    tokio::time::sleep(Duration::from_millis(500)).await;
    println!("{:?}ms", start.elapsed().as_millis()); // This code prints 0ms on a reasonable machine.
}
```

可以使用属性来更简便的开启 time pause:

```rust
#[tokio::test(start_paused = true)]
async fn paused_time() {
    let start = std::time::Instant::now();
    tokio::time::sleep(Duration::from_millis(500)).await;
    println!("{:?}ms", start.elapsed().as_millis());
}
```

虽然开启了 time pause, 但是异步函数的执行顺序和时间关系还是正常保持的:

-   立即打印 4 次 "Tick!"

<!--listend-->

```rust
#[tokio::test(start_paused = true)]
async fn interval_with_paused_time() {
    let mut interval = interval(Duration::from_millis(300));
    let _ = timeout(Duration::from_secs(1), async move {
        loop {
            interval.tick().await;
            println!("Tick!");
        }
    })
    .await;
}
```

使用 tokio_test::io::Builder 来 Mock AsyncRead and AsyncWrite:

```rust
use tokio::io::{AsyncBufReadExt, AsyncRead, AsyncWrite, AsyncWriteExt, BufReader};

async fn handle_connection<Reader, Writer>(
    reader: Reader,
    mut writer: Writer,
) -> std::io::Result<()>
where
    Reader: AsyncRead + Unpin,
    Writer: AsyncWrite + Unpin,
{
    let mut line = String::new();
    let mut reader = BufReader::new(reader);

    loop {
        if let Ok(bytes_read) = reader.read_line(&mut line).await {
            if bytes_read == 0 {
                break Ok(());
            }
            writer
                .write_all(format!("Thanks for your message.\r\n").as_bytes())
                .await
                .unwrap();
        }
        line.clear();
    }
}

#[tokio::test]
async fn client_handler_replies_politely() {
    let reader = tokio_test::io::Builder::new()
        .read(b"Hi there\r\n")
        .read(b"How are you doing?\r\n")
        .build();
    let writer = tokio_test::io::Builder::new()
        .write(b"Thanks for your message.\r\n")
        .write(b"Thanks for your message.\r\n")
        .build();
    let _ = handle_connection(reader, writer).await;
}
```


## <span class="section-num">17</span> async_channel {#async-channel}

std:sync::mpsc 和 crossbeam::channel 都是同步 channel, 不能在 async func 中使用, 否则可能 block 当前线程和 task.

tokio::sync:mpsc 是 multi-producer signle-consumer channel, 可以在 async 异步函数中使用, 而
async_channel crate 提供了multi-producer multi-consumer channel, 每个 message 只能被一个 consumer 消费.


## <span class="section-num">18</span> async_stream {#async-stream}

<https://docs.rs/async-stream/latest/async_stream/>

Asynchronous stream of elements.

Provides two macros, stream! and try_stream!, allowing the caller to define asynchronous streams of
elements. These are implemented using async &amp; await notation. This crate works without unstable
features.

```rust
use async_stream::stream;

use futures_util::pin_mut;
use futures_util::stream::StreamExt;

#[tokio::main]
async fn main() {
    let s = stream! {
        for i in 0..3 {
            yield i;
        }
    };

    pin_mut!(s); // needed for iteration

    while let Some(value) = s.next().await {
        println!("got {}", value);
    }
}
```
