---
title: "lazy_static"
author: ["zhangjun"]
lastmod: 2024-06-08T21:57:44+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 14
---

<https://docs.rs/lazy_static/latest/lazy_static/>

注：建议使用较新的 once_cell crate, 它功能更加强大，支持 local variables 的延迟初始化，所以也可以用于数组 field。

Rust 对 static 变量的 `编译时初始化` 有一些限制：只能使用常量表达式、const 函数来初始化。而不支持非
const 函数，复杂表达式的初始化。

lazy_static 宏提供了在运行时对 static 变量进行初始化的能力，而且支持复杂表达式、调用非 const 函数的初始化。

语法：

```rust
lazy_static! {
    [pub] static ref NAME_1: TYPE_1 = EXPR_1;
    [pub] static ref NAME_2: TYPE_2 = EXPR_2;
    ...
    [pub] static ref NAME_N: TYPE_N = EXPR_N;
}

// 或者
lazy_static! {
    /// This is an example for using doc comment attributes
    static ref EXAMPLE: u8 = 42;
}
```

实现细节：For a given `static ref NAME: TYPE = EXPR;`, the macro generates `a unique type` that
implements `Deref<TYPE>` and stores it in `a static with name NAME`. (Attributes end up attaching to
this type.)。 `On first deref`, EXPR gets evaluated and stored internally, such that all further
derefs can return a reference to `the same object`.

-   The Deref implementation uses a hidden static variable that is guarded by an atomic check on each
    access.

总结：

1.  lay_static! 实现了一个内部类型，它实现了 Deref&lt;TYPE&gt;, 该内部类型作为 NAME 的类型；
2.  当对 NAME 第一次 deref 时，Deref 的实现会用 EXPR 初始化 NAME，然后后续每次再 deref 时复用第一次初始化的结果；《=== 延迟初始化

<!--listend-->

```rust
lazy_static! {
    /// This is an example for using doc comment attributes
    static ref EXAMPLE: u8 = 42;
}

// 复杂例子，可以用任意表达式初始化 lazy static 变量。
#[macro_use]
extern crate lazy_static;

use std::collections::HashMap;

lazy_static! {
    static ref HASHMAP: HashMap<u32, &'static str> = {
        let mut m = HashMap::new();
        m.insert(0, "foo");
        m.insert(1, "bar");
        m.insert(2, "baz");
        m
    };
    static ref COUNT: usize = HASHMAP.len();
    static ref NUMBER: u32 = times_two(21);
}

fn times_two(n: u32) -> u32 { n * 2 }

fn main() {
    println!("The map has {} entries.", *COUNT);
    println!("The entry for `0` is \"{}\".", HASHMAP.get(&0).unwrap());
    println!("A expensive calculation on a static results in: {}.", *NUMBER);
}
```
