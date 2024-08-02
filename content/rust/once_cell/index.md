---
title: "once_cell"
author: ["zhangjun"]
lastmod: 2024-07-29T10:56:08+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 15
---

[once_cell](https://lib.rs/crates/once_cell) 是 lazy_static 的替代品，功能更加强大。

-   支持局部 static 变量的初始化；

Parts of once_cell API are included into std as of `Rust 1.70.0`.

once_cell 提供了两个 Cell 类似的类型 unsync::OnceCell 和 sync::OnceCell，可以用来保存 non-Copy 的类型对象（对比标准库的 Cell 只能保存实现 Copy 的对象，而 RefCell 可以保存 non-Copy 对象）。

-   OnceCell 只能设置 set() 一次，后续的 get() 返回的是已保存的值；《--- 单例模式
-   OnceCell 具有内部可变性，对于 set() 方法，使用共享引用即可。
-   sync::OnceCell 类型支持线程安全；

<!--listend-->

```rust
impl OnceCell<T> {
    fn new() -> OnceCell<T> { ... }
    fn set(&self, value: T) -> Result<(), T> { ... }
    fn get(&self) -> Option<&T> { ... }
}


use std::{sync::Mutex, collections::HashMap};
use once_cell::sync::Lazy;

static GLOBAL_DATA: Lazy<Mutex<HashMap<i32, String>>> = Lazy::new(|| {
    let mut m = HashMap::new();
    m.insert(13, "Spica".to_string());
    m.insert(74, "Hoyten".to_string());
    Mutex::new(m)
});

fn main() {
    println!("{:?}", GLOBAL_DATA.lock().unwrap());
}
```

OnceCell 类型和标准库其它类型的对比：

| !Sync types               | Access Mode                    | Drawbacks                |
|---------------------------|--------------------------------|--------------------------|
| Cell&lt;T&gt;             | T                              | requires T: Copy for get |
| RefCell&lt;T&gt;          | RefMut&lt;T&gt; / Ref&lt;T&gt; | may panic at runtime     |
| unsync::OnceCell&lt;T&gt; | &amp;T                         | assignable only once     |
|                           |                                |                          |

| Sync types              | Access Mode         | Drawbacks                                     |
|-------------------------|---------------------|-----------------------------------------------|
| AtomicT                 | T                   | works only with certain Copy types            |
| Mutex&lt;T&gt;          | MutexGuard&lt;T&gt; | may deadlock at runtime, may block the thread |
| sync::OnceCell&lt;T&gt; | &amp;T              | assignable only once, may block the thread    |

1.  全局对象安全初始化
    ```rust
       use std::{env, io};

       use once_cell::sync::OnceCell;

       #[derive(Debug)]
       pub struct Logger {
           // ...
       }
       static INSTANCE: OnceCell<Logger> = OnceCell::new();

       impl Logger {
           pub fn global() -> &'static Logger {
               INSTANCE.get().expect("logger is not initialized")
           }

           fn from_cli(args: env::Args) -> Result<Logger, std::io::Error> {
               // ...
           }
       }

       fn main() {
           let logger = Logger::from_cli(env::args()).unwrap();
           INSTANCE.set(logger).unwrap();
           // use `Logger::global()` from now on
       }
    ```

    1.  全局对象延迟初始化

        -   和 lazy_static!() 类似，但是没有使用 macro 语法和实现。
        -   可以使用 sync::Lazy 和 unsync::Lazy 类型来简化代码（变量类型必须是 static 而非 const），支持线程安全。

        <!--listend-->

        ```rust
                  use std::{sync::Mutex, collections::HashMap};

                  use once_cell::sync::OnceCell;

                  fn global_data() -> &'static Mutex<HashMap<i32, String>> {
           	   static INSTANCE: OnceCell<Mutex<HashMap<i32, String>>> = OnceCell::new();
           	   INSTANCE.get_or_init(|| {
                          let mut m = HashMap::new();
                          m.insert(13, "Spica".to_string());
                          m.insert(74, "Hoyten".to_string());
                          Mutex::new(m)
           	   })
                  }

                  // 使用 Lazy 类型来简化代码
                  use std::{sync::Mutex, collections::HashMap};
                  use once_cell::sync::Lazy;

                  // 使用 Lazy 类型定义全局 static 变量
                  static GLOBAL_DATA: Lazy<Mutex<HashMap<i32, String>>> = Lazy::new(|| {
           	   let mut m = HashMap::new();
           	   m.insert(13, "Spica".to_string());
           	   m.insert(74, "Hoyten".to_string());
           	   Mutex::new(m)
                  });

                  fn main() {
           	   println!("{:?}", GLOBAL_DATA.lock().unwrap());
                  }


                  // Lazy 还支持局部变量
                  use once_cell::unsync::Lazy;

                  fn main() {
           	   let ctx = vec![1, 2, 3];
           	   let thunk = Lazy::new(|| {
                          ctx.iter().sum::<i32>()
           	   });
           	   assert_eq!(*thunk, 6);
                  }

                  // 如果是在 struct field 中，则使用 OnceCell 类型，它只是使用 &self
                  use std::{fs, path::PathBuf};

                  use once_cell::unsync::OnceCell;

                  struct Ctx {
           	   config_path: PathBuf,
           	   config: OnceCell<String>,
                  }

                  impl Ctx {
           	   pub fn get_config(&self) -> Result<&str, std::io::Error> {
                          let cfg = self.config.get_or_try_init(|| {
           		   fs::read_to_string(&self.config_path)
                          })?;
                          Ok(cfg.as_str())
           	   }
                  }
        ```

使用 OnceCell 定义一个延迟初始化的正则表达式：

-   返回一个 &amp;'static Regex 类型的已编译、单例模式的 Regex 对象；

<!--listend-->

```rust
macro_rules! regex {
    ($re:literal $(,)?) => {{
        static RE: once_cell::sync::OnceCell<regex::Regex> = once_cell::sync::OnceCell::new();
        RE.get_or_init(|| regex::Regex::new($re).unwrap())
    }};
}
```

OnceCell 类型：线程安全的 cell 类型，只能被写入一次。

```rust
use once_cell::sync::OnceCell;

static CELL: OnceCell<String> = OnceCell::new();
assert!(CELL.get().is_none());

std::thread::spawn(|| {
    let value: &String = CELL.get_or_init(|| {
        "Hello, World!".to_string()
    });
    assert_eq!(value, "Hello, World!");
}).join().unwrap();

let value: Option<&String> = CELL.get();
assert!(value.is_some());
assert_eq!(value.unwrap().as_str(), "Hello, World!");
```

OnceCell 的方法：

-   new() ：创建；
-   set()/get_or_init() ：设置值；
-   get()/get_mut(): 获取值；

<!--listend-->

```rust
pub const fn new() -> OnceCell<T>
pub const fn with_value(value: T) -> OnceCell<T>

// Gets the reference to the underlying value. Returns None if the cell is empty, or being
// initialized. This method never blocks.
// get 有可能返回 None，如 cell 没有初始化，或者正在初始化中时。
pub fn get(&self) -> Option<&T>

// 等待，直到 cell 被初始化完成
pub fn wait(&self) -> &T

pub fn get_mut(&mut self) -> Option<&mut T>
pub unsafe fn get_unchecked(&self) -> &T

// 当 cell 为 empty 时，返回 Ok，否则返回 Err(value)
pub fn set(&self, value: T) -> Result<(), T>
pub fn try_insert(&self, value: T) -> Result<&T, (&T, T)>

// Gets the contents of the cell, initializing it with f if the cell was empty.  Many threads may
// call get_or_init concurrently with different initializing functions, but it is guaranteed that
// only one function will be executed.
pub fn get_or_init<F>(&self, f: F) -> &T where F: FnOnce() -> T,

pub fn get_or_try_init<F, E>(&self, f: F) -> Result<&T, E> where F: FnOnce() -> Result<T, E>,
pub fn take(&mut self) -> Option<T>
pub fn into_inner(self) -> Option<T>
```

例子：

```rust
use once_cell::sync::OnceCell;

let mut cell = std::sync::Arc::new(OnceCell::new());
let t = std::thread::spawn({
    let cell = std::sync::Arc::clone(&cell);
    move || cell.set(92).unwrap()
});

// Returns immediately, but might return None.
let _value_or_none = cell.get();

// Will return 92, but might block until the other thread does `.set`.
let value: &u32 = cell.wait();
assert_eq!(*value, 92);


// set 方法
use once_cell::sync::OnceCell;
static CELL: OnceCell<i32> = OnceCell::new();

fn main() {
    assert!(CELL.get().is_none());

    std::thread::spawn(|| {
        assert_eq!(CELL.set(92), Ok(()));
    }).join().unwrap();

    assert_eq!(CELL.set(62), Err(62));
    assert_eq!(CELL.get(), Some(&92));
}
```

OnceCell 需要明确的调用 set()/get_or_init() 方法来设置值。而 `Lazy` 则通过实现了 Deref/DerefMut trait，来在第一次 get() 或解引用访问时自动执行闭包函数来初始化值：

-   Lazy 支持并发安全；

<!--listend-->

```rust
use std::collections::HashMap;

use once_cell::sync::Lazy;

static HASHMAP: Lazy<HashMap<i32, String>> = Lazy::new(|| {
    println!("initializing");
    let mut m = HashMap::new();
    m.insert(13, "Spica".to_string());
    m.insert(74, "Hoyten".to_string());
    m
});

fn main() {
    println!("ready");
    std::thread::spawn(|| {
        println!("{:?}", HASHMAP.get(&13)); // 第一个 get() 访问时，调用 Lazy::new() 闭包函数来设置值
    }).join().unwrap();
    println!("{:?}", HASHMAP.get(&74));

    // Prints:
    //   ready
    //   initializing
    //   Some("Spica")
    //   Some("Hoyten")
}
```
