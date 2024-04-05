---
title: "Rust Notes"
author: ["张俊(zj@opsnull.com)"]
date: 2024-04-05T00:00:00+08:00
lastmod: 2024-04-05T11:02:48+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust-lang"]
series_order: 1
---

STARTUP: overview hideblocks


## <span class="section-num">1</span> comment {#comment}

常规注释(不显示在 doc 中)：

1.  `//` ：单行注释， 注释到行尾；
2.  `/* */` : 块注释；

cargo doc 文档注释：

1.  `///` ：为 following item 创建 lib doc；
2.  `/**` ： 为 following item 创建 lib doc；
3.  `//!` : 为 eclosing item 创建 lib doc, 例如 module/crate 级别的注释.

<!--listend-->

```rust
//! A doc comment that applies to the implicit anonymous module of this crate

pub mod outer_module {

    //!  - Inner line doc
    //!! - Still an inner line doc (but with a bang at the beginning)

    /*!  - Inner block doc */
    /*!! - Still an inner block doc (but with a bang at the beginning) */

    //   - Only a comment
    ///  - Outer line doc (exactly 3 slashes)
    //// - Only a comment

    /*   - Only a comment */
    /**  - Outer block doc (exactly) 2 asterisks */
    /*** - Only a comment */

    pub mod inner_module {}

    pub mod nested_comments {
        /* In Rust /* we can /* nest comments */ */ */

        // All three types of block comments can contain or be nested inside
        // any other type:

        /*   /* */  /** */  /*! */  */
        /*!  /* */  /** */  /*! */  */
        /**  /* */  /** */  /*! */  */
        pub mod dummy_item {}
    }

    pub mod degenerate_cases {
        // empty inner line doc
        //!

        // empty inner block doc
        /*!*/

        // empty line comment
        //

        // empty outer line doc
        ///

        // empty block comment
        /**/

        pub mod dummy_item {}

        // empty 2-asterisk block isn't a doc block, it is a block comment
        /***/

    }

    /* The next one isn't allowed because outer doc comments
       require an item that will receive the doc */

    /// Where is my item?
}
```


## <span class="section-num">2</span> type {#type}

rust 基本类型: Integer, Float, bool, char,Unit type (), array [T; N], tuple, pointer.

-   struct/enum/union 等属于自定义类型；

type alias 只是原样替换，并没有引入新类型，所以可以按照本来的方式使用别名，它可以提升代码的可读性。

```rust
type Thunk = Box<dyn Fn() + Send + 'static>;
let f: Thunk = Box::new(|| println!("hi"));
fn takes_long_type(f: Thunk) {
    // --snip--
}
fn returns_long_type() -> Thunk {
    // --snip--
}

type Result<T> = std::result::Result<T, std::io::Error>;

type Meters = u32;
let x: u32 = 5;
let y: Meters = 5;
println!("x + y = {}", x + y);  // Meters 是 u32 的 alias，还具有 u32 的所有操作。
```

rust 不对 primitive 类型提供隐式的转换（coercion），但是可以通过 as 关键字做显式的类型转换（casting），转换规则和 C 类似。as 转换只适合 primitive 类型，对于自定义类型，如 struct/enum，需要使用 From/Into
等 trait 的方法来转换。

```rust
#![allow(overflowing_literals)]

fn main() {
    let decimal = 65.4321_f32;

    // Error! No implicit conversion
    let integer: u8 = decimal;

    // Explicit conversion
    let integer = decimal as u8;
    let character = integer as char;

    // Error! There are limitations in conversion rules.  A float cannot be directly converted to a
    // char.
    let character = decimal as char;

    println!("Casting: {} -> {} -> {}", decimal, integer, character);

    // when casting any value to an unsigned type, T,
    // T::MAX + 1 is added or subtracted until the value
    // fits into the new type

    // 1000 already fits in a u16
    println!("1000 as a u16 is: {}", 1000 as u16);

    // 1000 - 256 - 256 - 256 = 232
    // Under the hood, the first 8 least significant bits (LSB) are kept,
    // while the rest towards the most significant bit (MSB) get truncated.
    println!("1000 as a u8 is : {}", 1000 as u8);
    // -1 + 256 = 255
    println!("  -1 as a u8 is : {}", (-1i8) as u8);

    // For positive numbers, this is the same as the modulus
    println!("1000 mod 256 is : {}", 1000 % 256);

    // When casting to a signed type, the (bitwise) result is the same as
    // first casting to the corresponding unsigned type. If the most significant
    // bit of that value is 1, then the value is negative.

    // Unless it already fits, of course.
    println!(" 128 as a i16 is: {}", 128 as i16);

    // In boundary case 128 value in 8-bit two's complement representation is -128
    println!(" 128 as a i8 is : {}", 128 as i8);

    // repeating the example above
    // 1000 as u8 -> 232
    println!("1000 as a u8 is : {}", 1000 as u8);
    // and the value of 232 in 8-bit two's complement representation is -24
    println!(" 232 as a i8 is : {}", 232 as i8);

    // Since Rust 1.45, the `as` keyword performs a *saturating cast*
    // when casting from float to int. If the floating point value exceeds
    // the upper bound or is less than the lower bound, the returned value
    // will be equal to the bound crossed.

    // 300.0 as u8 is 255
    println!(" 300.0 as u8 is : {}", 300.0_f32 as u8);
    // -100.0 as u8 is 0
    println!("-100.0 as u8 is : {}", -100.0_f32 as u8);
    // nan as u8 is 0
    println!("   nan as u8 is : {}", f32::NAN as u8);

    // This behavior incurs a small runtime cost and can be avoided with unsafe methods, however the
    // results might overflow and return **unsound values**. Use these methods wisely:
    unsafe {
        // 300.0 as u8 is 44
        println!(" 300.0 as u8 is : {}", 300.0_f32.to_int_unchecked::<u8>());
        // -100.0 as u8 is 156
        println!("-100.0 as u8 is : {}", (-100.0_f32).to_int_unchecked::<u8>());
        // nan as u8 is 0
        println!("   nan as u8 is : {}", f32::NAN.to_int_unchecked::<u8>());
    }
}
```

栈变量类型：

1.  原始值；
2.  array/struct/tuple/enum/union

堆变量类型：

1.  字符串：String
2.  容器：Vec/HashMap/HashSet
3.  Slice
4.  智能指针：Box/Rc/Arc/


## <span class="section-num">3</span> scalar {#scalar}

Scalar 类型如下：

Signed integers
: i8, i16, i32, i64, i128 and isize (pointer size)，默认为 i32；

Unsigned integers
: u8, u16, u32, u64, u128 and usize (pointer size)

Floating point
: f32, f64, 默认为 f64;

char
: Unicode scalar values like 'a', 'α' and '∞' (4 bytes each)

bool
: true/false, 占用 1 byte 空间;

The unit type ()
: 只有一个空值 ()；

对于数值变量:

1.  没有指定类型时, 默认为 i32 和 f64;
2.  可以加类型后缀, 如 23u8, 12.3f64;
3.  数字/类型后缀之间可以加下划线, 如 2_3_u8 = 23u8;
4.  可以使用 0b/0o/0x 表示整型；

<!--listend-->

```rust
let x = 42; // 类型推断为 i32
let y = 42u64; // 使用类型后缀指定为 u64
let bin = 0b1111_0000; // 二进制，等于十进制的 240
let oct = 0o377; // 八进制，等于十进制的 255
let hex = 0xff; // 十六进制，等于十进制的 255

let remainder = 43.0 % 5.0; // 取模运算, 截断除法

let age: u32 = 18;
let is_adult: bool = if age >= 18 { true } else { false };
assert!(result == 4, "Expected the result to be 4");  // assert!() 检查表达式值是否为 true。

fn main() {
    // Variables can be type annotated.
    let logical: bool = true;

    let a_float: f64 = 1.0;
    let an_integer   = 5i32; // Suffix annotation
    let default_float   = 3.0; // `f64`
    let default_integer = 7;   // `i32`

    // A type can also be inferred from context.
    let mut inferred_type = 12; // Type i64 is inferred from another line.
    inferred_type = 4294967296i64;

    // A mutable variable's value can be changed.
    let mut mutable = 12; // Mutable `i32`
    mutable = 21;
    // Error! The type of a variable can't be changed.
    mutable = true;

    // Variables can be overwritten with shadowing.
    let mutable = true;
}
```

整数在超出其类型能表示的范围时会溢出。

1.  在 debug 构建中，Rust 检查整数溢出并导致 panic。
2.  在 release 构建中，溢出不会被检查，并可能导致 "环绕" 行为。

<!--listend-->

```rust
fn main() {
    let x: u8 = 255;
    let y: u8 = x.wrapping_add(1); // 使用 wrapping_add 可以防止 panic
    println!("y: {}", y); // 输出: y: 0
}
```

char 是 unicode codepoint, 占用 4 bytes, 可以表示所有 unicode 字符.

```rust
let emoji: char = '😂';
let chinese_character: char = '中';
fn main() { // 遍历字符串中的 char 字符
    let word = "Rust语言";
    for ch in word.chars() {
        println!("{}", ch);
    }
}

// 使用 as 将 char 转换为整型码点, 使用 std::char::from_u32() 将码点 u32 转换为 char
let unicode_codepoint = '🦀' as u32; // 将字符转换为对应的 Unicode 代码点
println!("The Unicode code point of '🦀' is: U+{:X}", unicode_codepoint);
let character_from_codepoint = std::char::from_u32(unicode_codepoint).unwrap_or_default();
println!("The character from code point U+{:X} is: '{}'", unicode_codepoint, character_from_codepoint);

// 向 String 添加 char 和 &str
let mut s = String::new();
s.push('H'); // 添加单个字符到字符串
s.push_str("ello"); // 添加字符串到现有字符串
```

rust 不会为 primitive type 做隐式的转换，需要使用 as 关键字来显式转换：

-   as 是后缀运算法，优先级非常高。
-   类型转换时，如果浮点数转换为整数，其小数部分将被截断（不进行四舍五入）。如果整数类型转换为浮点数，值将被转换为最接近的浮点数表示。

<!--listend-->

```rust
fn main() {
    let decimal = 97.123_f32;
    let integer: u8 = decimal as u8; // 小数部分被阶段
    let c1: char = decimal as char;
    let c2 = integer as char;
    assert_eq!(integer, 'b' as u8);
    println!("Success!");

    let integer: u32 = 5;
    let float: f64 = 3.0;

    // 整数转换为浮点数
    let int_to_float = integer as f64; // 5.0

    // 浮点数转换为整数
    let float_to_int = float as u32; // 3
}
```

复杂的类型转换, 需要使用 From/Into/TryInto trait. try_into 方法会返回一个 Result 类型，当转换失败时（例如，因为类型溢出或数据丢失），它会返回一个错误。from 方法则通常用于无风险的转换，它不会产生错误。

```rust
use std::convert::TryInto;

fn main() {
    let decimal = 65.4321_f64;

    // 使用 `try_into` 方法进行安全转换
    let integer: u8 = decimal.try_into().unwrap_or_default(); // 出错时返回缺省值 0

    // 使用 `from` 方法进行安全转换
    let integer_from = u8::from(42); // 因为 42 可以安全地转换为 `u8`

    println!("Safe casting: {} -> {}", decimal, integer);
    println!("From casting: {}", integer_from);
}
```

Rust 标准库还提供了一些数值类型的转换函数，如 to_string 和 parse，用于在数值类型和字符串之间进行转换。

-   任何实现了 Display/Error trait 的对象都自动提供了 to_string() 方法;

<!--listend-->

```rust
fn main() {
    // parse() 方法将字符串转换为 其他类型
    let my_str = "10";
    let my_int = my_str.parse::<i32>().unwrap();
    println!("String to int: {} -> {}", my_str, my_int);

    // to_string() 将其他类型转换为 String.
    let my_new_str = my_int.to_string();
    println!("Int to string: {} -> {}", my_int, my_new_str);
}
```

单元类型 (Unit Type)：() 既是类型也是它的唯一值。单元类型在 Rust 中的一个主要用途是作为函数的返回类型，表明该函数执行一些操作，但不返回任何数据。

```rust
fn main() {
    println!("{:p}, {:p}", &(), &()); // 打印地址相同, 说明是唯一类型值

    print_message();

    // 显式使用单元类型和单元值
    let my_unit: () = ();

    // 函数参数接受单元类型值 ()
    take_unit(());

    // 泛型类型也可以使用单元类型, 常用于不需要返回实际值的 Ok.
    let result: Result<(), &str> = Ok(());
    match result {
        Ok(_) => println!("Operation was successful."),
        Err(e) => println!("Error occurred: {}", e),
    }
}


fn print_message() {
    println!("Hello, world!");
    // 这个函数隐式返回单元类型 `()`
}

fn take_unit(_unit: ()) {
    println!("This function takes a unit type.");
}
```


## <span class="section-num">4</span> char/str/String/OsStr/OsString/CStr/CString {#char-str-string-osstr-osstring-cstr-cstring}

char 是固定 4 bytes 的 Unicode 字符码点, 可以使用 as 在 u8/u32 相互转换。str 和 String 保存的是
UTF-8 变长编码的 Unicode 字符.

str 是 rust 的原始类型，对应一块类型为 [u8] 的连续内存区域，和 String 类似，保存的是字符串的 UTF-8编码值，str 是编译时大小未知的，一般不能直接作为变量类型使用，而是使用借用类型 &amp;str；

-   rust 字符串字面量类型是 &amp;str，而且默认具有 'static lifetime；

<!--listend-->

```rust
fn main() {
    let s: str = "hello, world"; // 错误，str 不能直接作为类型
    let s: &str = "hello, world"; // OK
    println!("Success!");
}

// &str 不能自动协变到 &[u8], 可以使用 as_bytes() 方法来转换为 &[u8]
let bytes = "bors".as_bytes();
assert_eq!(b"bors", bytes);
```

也可以使用 Box&lt;str&gt; 来将未知大小的 str 转换为智能指针类型，，然后使用 Box&lt;T&gt; 缺省实现的
Derefer&lt;Target=T&gt; 来在需要 &amp;T 的地方使用 &amp;Box&lt;T&gt;:

```rust
fn main() {
    let s: Box<str> = "hello, world".into(); // 在对上分配对象，s 拥有该对象
    greetings(&s)
}

fn greetings(s: &str) {
    println!("{}",s)
}
```

&amp;str 和 String 的相互转换：

1.  String -&gt; &amp;str: String.as_str();
2.  String::from("Sunfei") 或 "Sunface".to_string()

String 是 UTF-8 编码的可变长字符串，位于 heap 中, 可以自动增加和修改内容:

-   char 是固定的 4 bytes 长度的 Unicode 码点，而 UTF-8 是可变长编码；
-   b'x' 返回字符 x 的 UTF-8 编码值（u8 类型）， 如 104 = b'h';
-   b"xyz" 返回 &amp;[u8; N] 类型数组的借用，支持转义字符, 即为 &amp;['x', 'y', 'z'];
-   r###"\\a\\b\\c"###: 不对字符串内容转义，r 后面的 # 数量可变，且只能使用连续的 # 字符；
-   br##"\\a\\b\\c\\t\\n"##: 类型为 &amp;[u8, 10], 不对字符串转移，必须是 br 而不能是 rb；

字符串可以包含换行, 转义字符(如 \\x23, \\u{211D}), 默认左对齐, 行尾如果是 \\ 字符, 则删除换行符:

```rust
let s1 = String::from("hello,");
println!("#{:20.20}#", s1); // 字符串显示默认左对齐(数字是右对齐),显示: #hello,              #

println!("{}", "a\t
      b  \
       c d
      ef
      ");

// 输出:
// #hello,              #
// a
// b  c d
// ef

```

String 和 &amp;str 的 Index 操作返回 &amp;str, 但是需要保证 &amp;s[i..j] 的 i..j 是有效的字符边界，否则 panic，可以使用 non-panicking 版本 get();

-   s[i] 是禁止的， 因为 String/&amp;str 是 UTF-8 编码，返回 &amp;u8 可能是无意义的。

<!--listend-->

```rust
let s = "hello";
println!("The first letter of s is {}", s[0]); // 错误，不支持 s[0];

// 可以使用 as_bytres() 方法将 String/&str 转换为 &[u8], 然后再 index 某个 u8:
let s = "hello";
assert_eq!(s.as_bytes()[0], 104);
// or
assert_eq!(s.as_bytes()[0], b'h');

// The first byte is 240 which isn't obviously useful
let s = "💖💖💖💖💖";
assert_eq!(s.as_bytes()[0], 240);

let s = String::from("hello world");
let hello = &s[0..5]; // &str 类型
```

String 类型实现了 Defref&lt;Target = str&gt;, 所以编译器自动将 &amp;String 转换为 &amp;str：

1.  String 类似可以使用 str 定义的所有方法；
2.  在需要 &amp;str 的函数参数，可以传入 &amp;String;

<!--listend-->

```rust
// some bytes, in a vector
let sparkle_heart = vec![240, 159, 146, 150]; // UTF-8 编码值（非 char）
// We know these bytes are valid, so we'll use `unwrap()`.
let sparkle_heart = String::from_utf8(sparkle_heart).unwrap();
assert_eq!("💖", sparkle_heart);

let s = "hello";
let third_character = s.chars().nth(2); // charts() 返回 char 类型（固定 4 bytes Unicode 码点）
assert_eq!(third_character, Some('l'));
```

str 和 String 都是 `严格遵守 UTF-8 编码的` ，但是对于一些操作系统文件名或路径，可以不是 UTF-8 编码的字符串。所以 rust 引入了 std::ffi::OsStr/OsString 类型：

-   OsStr 是 unsized type，一般需要和 &amp; 和 Box 使用，不可以改变，类似于 str；
-   OsString 是 sized type，是 Owned 版本，可以修改，类似于 String；
-   OsString 实现了 Deref&lt;target = OsStr&gt;， 所以 &amp;OsString 可以使用 &amp;OsStr 定义的所有方法。

<!--listend-->

```rust
use std::ffi::OsStr;
let os_str = OsStr::new("foo");
```

String 可以 +/+= &amp;str, 但是不支持 &amp;str 之间的 +/+= 操作, 以及 &amp;str + String:

```rust
let mut ss = String::from("abcd");
ss += " def"; // OK: String + &str

// " def" + ss; // `+` cannot be used to concatenate a `&str` with a `String`
" def".to_owned() + &ss;   // OK

let s1 = String::from("hello,");
let s2 = String::from("world!");
let s3 = s1 + &s2;   // let s3 = s1.clone() +&s2;
assert_eq!(s3, "hello,world!");
println!("{}", s1); // s1 已经在上面的 + 操作被 move, 导致继续使用 s1 出错。
```

OsStr 的方法:

-   pub fn as_encoded_bytes(&amp;self) -&gt; &amp;[u8]
-   pub fn into_os_string(self: Box&lt;OsStr&gt;) -&gt; OsString
-   pub fn make_ascii_lowercase(&amp;mut self)
-   pub fn to_os_string(&amp;self) -&gt; OsString
-   pub fn to_str(&amp;self) -&gt; Option&lt;&amp;str&gt;
-   pub fn to_string_lossy(&amp;self) -&gt; Cow&lt;'_, str&gt;

OsStr/OsString 都不是 NULL 终止的字符串, 但是类型 `std::ffi::CStr 和 std::ffi::CString` 是 NULL 终止的字符串.

```rust
use std::ffi::CString;
use std::os::raw::c_char;

fn main() {
    let s = String::from("Hello, world!");
    let cs = CString::new(s).unwrap();
    let p = cs.as_ptr() as *const c_char;
    println!("Address: {:?}", p);
}
```

最佳实践: Concatenating strings with format! It is possible to build up strings using the `push` and
`push_str` methods on `a mutable String`, or using `its + operator`. However, it is often more convenient
to use `format!`, especially where there is a mix of literal and non-literal strings.

```rust
  fn say_hello(name: &str) -> String {
    // We could construct the result string manually.
    // let mut result = "Hello ".to_owned();
    // result.push_str(name);
    // result.push('!');
    // result

    // But using format! is better.
    format!("Hello {name}!")
```

FromStr trait: 从 string 来生成各种类型的值：

-   一般被 &amp;str.parse::&lt;T&gt;() 方法隐式调用。
-   rust 的基本类型，如整数、浮点数、bool、char、String、PathBuf、IpAddr、SocketAddr、Ipv4Addr、
    Ipv6Addr 都实现了该 trait。

<!--listend-->

```rust
pub trait FromStr: Sized {
    type Err;

    // Required method
    fn from_str(s: &str) -> Result<Self, Self::Err>;
}
```

str 的 parse::&lt;T&gt;() 方法用于将字符串转换为其它对象：

```rust
pub fn parse<F>(&self) -> Result<F, <F as FromStr>::Err>
  where
      F: FromStr,
```

在使用 &amp;str.parse() 方法时一般需要指定目标对象类型，否则 rustc 不支持该调用哪个类型的 FromStr 实现：

-   注意：只支持目标类型本身，而不是它的 &amp;T 或 &amp;mut T；

<!--listend-->

```rust
let four: u32 = "4".parse().unwrap();
assert_eq!(4, four);

let four = "4".parse::<u32>();
assert_eq!(Ok(4), four);

// Error
let nope = "j".parse::<u32>();
assert!(nope.is_err());
```

b"xxx" 的类型是 &amp;[u8; N] 可以被当作 &amp;[u8] 使用， 支持如下方法：

```rust
impl [u8]

// 检查 [u8] 各元素是否是 ascii
pub const fn is_ascii(&self) -> bool

pub const fn as_ascii(&self) -> Option<&[AsciiChar]>
pub const unsafe fn as_ascii_unchecked(&self) -> &[AsciiChar]
pub fn eq_ignore_ascii_case(&self, other: &[u8]) -> bool
pub fn make_ascii_uppercase(&mut self)
pub fn make_ascii_lowercase(&mut self)
pub fn escape_ascii(&self) -> EscapeAscii<'_>
    let s = b"0\t\r\n'\"\\\x9d";
    let escaped = s.escape_ascii().to_string();
    assert_eq!(escaped, "0\\t\\r\\n\\'\\\"\\\\\\x9d");

pub const fn trim_ascii_start(&self) -> &[u8]
#![feature(byte_slice_trim_ascii)]
assert_eq!(b" \t hello world\n".trim_ascii_start(), b"hello world\n");
assert_eq!(b"  ".trim_ascii_start(), b"");
assert_eq!(b"".trim_ascii_start(), b"");

pub const fn trim_ascii_end(&self) -> &[u8]
pub const fn trim_ascii(&self) -> &[u8]
```

[AsciiChar] 的字符串转换方法：

-   [u8] 的 as_ascii() 返回该对象。

<!--listend-->

```rust
impl [AsciiChar]
pub const fn as_str(&self) -> &str
pub const fn as_bytes(&self) -> &[u8]
```


## <span class="section-num">5</span> array {#array}

array 是固定类型和长度的数组，用 [T; N] 表示，N 必须是编译时常量且是类型的一部分，在栈上分配的连续内存空间, 可以高效连续访问和遍历。

```rust
fn init_arr(n: i32) {
    let arr = [1; n]; // 错误, n 不是编译时常量.
}
```

array 的特性

1.  同类型元素：数组中的所有元素必须是相同的数据类型。
2.  固定长度：数组的长度在编译时就确定，并且之后不能更改。
3.  栈分配：数组通常在栈上分配，这意味着它们的访问速度非常快。

创建 array：

```rust
let numbers: [i32; 5] = [1, 2, 3, 4, 5]; // 声明一个有 5 个 i32 整数的数组
let zeroes: [i32; 5] = [0; 5]; // 声明一个有 5 个元素都是 0 的数组. 表达式右侧 [Value; N] 的 Value 必须实现 Copy
println!("numbers: {:?}", numbers);
println!("zeroes: {:?}", zeroes);

let mut values: [i32; 3] = [10, 20, 30]; // 声明可变数组，后续可以修改。
values[1] = 25;
println!("values: {:?}", values);
println!("The array has {} elements.", values.len());
```

固定大小的 array [T; N] 可以被 type coerce 到大小未知的 slice [T]：

-   &amp;[T; N ] 可以被隐式自动转换为 &amp;[T]，所以 `array 可以调用 slice 的方法` 。
-   array 并没有实现 Deref trait，所以上面的自动转换不是 Deref 的行为；

<!--listend-->

```rust
let mut array: [i32; 3] = [0; 3]; // 左边是类型, 右边是初始化表达式!

// coercing an array to a slice
let str_slice: &[&str] = &["one", "two", "three"];

let numbers = &[0, 1, 2]; // numbers 是 &[i32; 3] 类型
print_type_of(&numbers);

// 数组 [i32; 3] 可以被 coerce 到  [T], 所以 &[i32; 3] 可以被赋值给 &[i32]
let numbers: &[i32] = &[0, 1, 2]; //
print_type_of(&numbers); // &[i32]

for n in numbers { // number 虽然前面没有加 &, 但是它本身是 &[i32] 类型, 所以迭代后元素 n 是 &32 类型.
    print_type_of(&n);
}
fn print_type_of<T>(v: &T) -> String {
    format!("{}", std::any::type_name_of_val(v))
}

print_type_of(&numbers[0]); // i32，切片引用支持 index 操作，返回元素本身, 必须实现 Copy, 否则会被转移所有权
```

rust 数组和集合的元素索引都从 0 开始, 必须 &lt; len(), 否则会 panic. 但是可以通过 get(i) 返回的
Option&lt;T&gt; 来判断 index 对应的元素是否存在.

array 的 slice 操作, 如 a[start..ennd] 返回一个 dynamic size 的 slice [T] 类型，一般使用 2 个 usize
大小的 &amp;[T] 类型，它包含指向连续内存 array 的起始地址以及 slice 的长度。

-   slice 操作返回的 &amp;a[start..ennd] 不需要拷贝内存, 它们不拥有任何数据，而只是借用数组或其他集合中的数据。
-   slice [T] 是 dynamic size, 不能反向 coerce 到 array, 但是可以使用 slice.try_into().unwrap() or
    &lt;ArrayType&gt;::try_from(slice).unwrap(). 来在相同长度的 slice 和 array 之间转换:

<!--listend-->

```rust
let arr = [1, 2, 3, 4, 5];
// 创建一个包含整个数组的切片
let slice_whole = &arr[..]; // 等同于 &arr
// 创建一个包含数组中一部分元素的切片
let slice_part = &arr[1..4]; // 包含索引1，2，3的元素

let s = String::from("hello world");
// 创建一个字符串切片，包含前5个字符
let hello = &s[0..5];
// 创建一个字符串切片，包含后5个字符
let world = &s[6..11];

let bytes: [u8; 3] = [1, 0, 2]; // bytes 是 array
// &bytes[0..2] 返回 slice
// <[u8; 2]>::try_from(&bytes[0..2]) 是从 slice 生成 array
assert_eq!(1, u16::from_le_bytes(<[u8; 2]>::try_from(&bytes[0..2]).unwrap()));

// bytes[1..3] 返回 slice, 用来生成 array
assert_eq!(512, u16::from_le_bytes(bytes[1..3].try_into().unwrap()));

let mut bytes: [u8; 3] = [1, 0, 2];
let bytes_head: [u8; 2] = <[u8; 2]>::try_from(&mut bytes[0..2]).unwrap();
assert_eq!(1, u16::from_le_bytes(bytes_head));
let bytes_tail: [u8; 2] = (&mut bytes[1..3]).try_into().unwrap();
assert_eq!(512, u16::from_le_bytes(bytes_tail));

let a = [1, 2, 3, 4, 5];
let slice = &a[1..3]; // a[1..3] 结果为 [i32], 再 & 操作返回固定大小的 &[T]
assert_eq!(slice, &[2, 3]); // &[i32] 可以直接和 &[i32; 2] 类型比较

// 一个接受切片作为参数的函数
fn sum(slice: &[i32]) -> i32 {
    let mut total = 0;
    for i in slice {
        total += i;
    }
    total
}
fn main() {
    let arr = [1, 2, 3, 4, 5];
    let result = sum(&arr[1..4]); // 只计算数组一部分的和
    println!("The sum of the part of the array is: {}", result);
}
```

array 支持 for-in 迭代，结果为数组元素 T：

-   slice 操作 &amp;a[m..n], 结果为切片引用 &amp;[T]，它也支持迭代，迭代结果为 &amp;T;

<!--listend-->

```rust
fn main() {
    let mut numbers: [i32; 5] = [1, 2, 3, 4, 5];
    for number in numbers { // numbers.iter()/numbers.iter_mut()/numbers.into_iter()
        println!("number: {}", number);
    }
}
```

rust index 操作返回的是元素本身, 如果 array 中元素不支持 Copy 则不能被 move.(rust 不允许
Array/Vec/HashMap/HashSet 中的元素被 move 出来).

-   其他类似场景, 如 Vec/Struct/Enum 也不允许部分元素或成员被 move out；
-   解决办法是使用 std::mem::replace() 来用其同类型对象来替换:
-   Slice patterns can match both arrays of fixed size and slices of dynamic size.

<!--listend-->

```rust
fn move_away(_: String) { /* Do interesting things. */ }
let [john, roa] = ["John".to_string(), "Roa".to_string()];
move_away(john);
move_away(roa);

// Fixed size
let arr = [1, 2, 3];
match arr {
    [1, _, _] => "starts with one",
    [a, b, c] => "starts with something else",
};

// Dynamic size
let v = vec![1, 2, 3];
match v[..] {
    [a, b] => { /* this arm will not apply because the length doesn't match */ }
    [a, b, c] => { /* this arm will apply */ }
    _ => { /* this wildcard is required, since the length is not known statically */ }
};

// 有问题代码:
struct Buffer<T> { buf: Vec<T> }
impl<T> Buffer<T> {
    fn replace_index(&mut self, i: usize, v: T) -> T {
        // error: cannot move out of dereference of `&mut`-pointer
        let t = self.buf[i];
        self.buf[i] = v;
        t
    }
}

// std::mem::replace 对 &mut 对象操作, 替换并返回 Array/Vec 的元素:
use std::mem;
impl<T> Buffer<T> {
    fn replace_index(&mut self, i: usize, v: T) -> T {
        mem::replace(&mut self.buf[i], v)
    }
}
let mut buffer = Buffer { buf: vec![0, 1] };
assert_eq!(buffer.buf[0], 0);
assert_eq!(buffer.replace_index(0, 2), 0);
assert_eq!(buffer.buf[0], 2);
```

如果 array 的元素类型实现了如下 trait，这 array 也实现了对应 trait：

-   Copy，Clone
-   Debug
-   IntoIterator (implemented for [T; N], &amp;[T; N] and &amp;mut [T; N])
-   PartialEq, PartialOrd, Eq, Ord
-   Hash
-   AsRef, AsMut
-   Borrow, BorrowMut


## <span class="section-num">6</span> slice {#slice}

slice 代表一块连续的内存区域，用 [T] 表示，它是未知大小的类型，一般转换为指针类型，如 &amp; 或 Box/Rc 等；

-   [T] 是编译时位置大小的类型, 但 &amp;[T]/&amp;mut [T] 是固定 2 usize 大小的指针类型；
-   作为变量/函数输入/输出参数类型来使用时, 一般使用具体固定大小的 &amp;[T] 类型；

<!--listend-->

```rust
let pointer_size = std::mem::size_of::<&u8>();
assert_eq!(2 * pointer_size, std::mem::size_of::<&[u8]>());
assert_eq!(2 * pointer_size, std::mem::size_of::<*const [u8]>());
assert_eq!(2 * pointer_size, std::mem::size_of::<Box<[u8]>>());
assert_eq!(2 * pointer_size, std::mem::size_of::<Rc<[u8]>>());
```

创建 slice &amp;[T]：

-   对 array/vec/String/&amp;str 的 range index 操作返回 [T];
-   Vec[T] 实现了 Deref&lt;Target=[T]&gt;，所以 &amp;Vec&lt;T&gt; 可以被隐式转换为 &amp;[T]，在需要 &amp;[T] 类型的地方可以传入 &amp;Vec&lt;T&gt; 类型，Vec 对象也可以调用 slice [T] 的方法；
    -   &amp;vec 返回 &amp;Vec&lt;i32&gt; 类型，而 &amp;vec[n..m] 返回 &amp;[i32]；
-   array [T; N] 可以被 type coercing 到 [T], 所以 &amp;[T; N] 可以被隐式转换为 &amp;[T]，这样 array 对象也可以调用 slice [T] 的方法；

<!--listend-->

```rust
// slicing a Vec
let vec = vec![1, 2, 3];
let int_slice = &vec[..];   // &vec 返回的是 &Vec<i32> 类型而非 &[i32]
let int_slice: &[i32] = &vec; // 由于 Vec[T] 实现了 Deref<Target=[T]>，所以 &Vec<i32> 可以被转换为 &[i32] 类型

// coercing an array to a slice
let str_slice: &[&str] = &["one", "two", "three"];

let mut x = [1, 2, 3];
let x = &mut x[..]; // Take a full slice of `x`.
x[1] = 7;
assert_eq!(x, &[1, 7, 3]);

// 对 String 类型进行 slice 操作, 结果为 &str
let s = String::from("hello world");
let hello = &s[0..5];

// 由于数组 [i32; 3] 可以被 coerce 到 unsize 的 [T], 所以 &[i32; 3] 可以被赋值给 &[i32]
let numbers: &[i32] = &[0, 1, 2];
print_type_of(&numbers); // &[i32]，数组引用类型可以被自动转换为切片引用类型
for n in numbers {
    print_type_of(&n);  // &i32，迭代切片引用，返回元素的引用
}
print_type_of(&numbers[0]); // i32，切片引用的支持 index 操作，返回元素本身

fn read_slice(slice: &[usize]) {
    // ...
}
let v = vec![0, 1];
read_slice(&v); // Deref 自动转换

let u: &[usize] = &v; // Deref 自动转换
// or like this:
let u: &[_] = &v;

// 其他例子
#![allow(unused)]
fn print_type_of<T>(_: &T) {
    println!("{}", std::any::type_name::<T>())
}
fn main() {
    let x = [1_u32, 2, 3]; // [u32;3]，数组类型
    let x2 = &x; // &[u32; 3] ，数组引用类型
    let x3 = &x[..]; // &[u32]，切片引用类型
    let x4 = &x[1..]; // &[u32]，切片引用类型

    let y = vec![1_u32, 2, 3]; // Vec<u32>，向量类型
    let y2 = &y; // &Vec<u32>，向量引用
    let y3 = &y[..]; // &[u32]，切片引用
    // u32，切片引用的支持 index 操作，返回元素本身
    y3[1];
    print_type_of(&y3[1]); // u32


    let numbers = &[0, 1, 2];
    print_type_of(&numbers); // &[i32; 3]，数组引用类型

    let numbers: &[i32] = &[0, 1, 2];
    print_type_of(&numbers); // &[i32]，数组引用类型可以被自动转换为切片引用类型
    for n in numbers {
        print_type_of(&n);  // &i32，迭代切片引用，返回元素的引用
    }
    print_type_of(&numbers[0]); // i32，切片引用的支持 index 操作，返回元素本身
}
```

slice.to_vec() 方法将 slice 内容 clone 到一个新的 Vec 中.

slice [T] 的 index 操作返回的 T 类型, mut [T] 的 index 操作返回的 mut T 类型:

-   s[i] 返回的 s 的元素类型而非它的引用，所以支持将 x[i] 作为左值。

<!--listend-->

```rust
let mut x = [1, 2, 3];
let x = &mut x[..]; // Take a full slice of `x`.
x[1] = 7; // x[1] 的类型是 mut i32, 所以可以进行修改.
```

for-in 迭代 &amp;[T] 时返回 &amp;T 元素：

```rust
let numbers: &[i32] = &[0, 1, 2];  // &[0, 1, 2] 的类型是 &[i32; 3] 被 rust 自动转换为 &[i32]
for n in numbers { // n 是 &i32 类型
    println!("{n} is a number!");
}

let mut scores: &mut [i32] = &mut [7, 8, 9];
for score in scores { // score 是 &mut i32 类型.
    *score += 1;
}
```

对 array/slice 进行 index 操作时，如果超过了 length，则会 panic。解决办法是使用安全的 .get() 方法，它返回一个 Option，get() 方法的参数是 SliceIndex&lt;[T]&gt;，Range&lt;usize&gt;/RangeFull/RangeFrom&lt;usize&gt; 等均实现了该 trait：

```rust
// Arrays can be safely accessed using `.get`, which returns an `Option`. This can be matched as
// shown below, or used with `.expect()` if you would like the program to exit with a nice message
// instead of happily continue.
for i in 0..xs.len() + 1 { // Oops, one element too far!
    match xs.get(i) {
        Some(xval) => println!("{}: {}", i, xval),
        None => println!("Slow down! {} is too far!", i),
    }
}

let v = [10, 40, 30];
assert_eq!(Some(&40), v.get(1));
assert_eq!(Some(&[10, 40][..]), v.get(0..2));
assert_eq!(None, v.get(3));
assert_eq!(None, v.get(0..4));
```

数组 slice 的 flatten：

```rust
impl<T, const N: usize> [[T; N]]
pub const fn flatten(&self) -> &[T]

#![feature(slice_flatten)]
assert_eq!([[1, 2, 3], [4, 5, 6]].flatten(), &[1, 2, 3, 4, 5, 6]);
assert_eq!(
    [[1, 2, 3], [4, 5, 6]].flatten(),
    [[1, 2], [3, 4], [5, 6]].flatten(),
);
let slice_of_empty_arrays: &[[i32; 0]] = &[[], [], [], [], []];
assert!(slice_of_empty_arrays.flatten().is_empty());
let empty_slice_of_arrays: &[[u32; 10]] = &[];
assert!(empty_slice_of_arrays.flatten().is_empty());
```


### <span class="section-num">6.1</span> slice 方法 {#slice-方法}

```rust
impl<T> [T]

pub const fn len(&self) -> usize
pub const fn is_empty(&self) -> bool

// slice 有可能为空，所以 first/last 都返回 Option
pub const fn first(&self) -> Option<&T>
pub fn first_mut(&mut self) -> Option<&mut T>
pub const fn last(&self) -> Option<&T>
pub fn last_mut(&mut self) -> Option<&mut T>

pub const fn split_first(&self) -> Option<(&T, &[T])>
pub fn split_first_mut(&mut self) -> Option<(&mut T, &mut [T])>
pub const fn split_last(&self) -> Option<(&T, &[T])>
pub fn split_last_mut(&mut self) -> Option<(&mut T, &mut [T])>
let x = &[0, 1, 2];  // x 是 &[i32; 3] 类型，但是可以被 type coer 到 &[i32] 类型，所以可以调用 slice [T] 的方法。
if let Some((first, elements)) = x.split_first() {
    assert_eq!(first, &0);
    assert_eq!(elements, &[1, 2]);
}

// 返回第一个 N 个元素的数组，如果元素少于 N 则返回 None
pub const fn first_chunk<const N: usize>(&self) -> Option<&[T; N]>
pub fn first_chunk_mut<const N: usize>(&mut self) -> Option<&mut [T; N]>
let u = [10, 40, 30];
assert_eq!(Some(&[10, 40]), u.first_chunk::<2>());
let v: &[i32] = &[10];
assert_eq!(None, v.first_chunk::<2>());
let w: &[i32] = &[];
assert_eq!(Some(&[]), w.first_chunk::<0>());

// 返回第一个或最后一个 chunk 数组和剩下的 slice，如果元素少于 N 则返回 None
pub const fn split_first_chunk<const N: usize>(&self) -> Option<(&[T; N], &[T])>
pub fn split_first_chunk_mut<const N: usize>( &mut self ) -> Option<(&mut [T; N], &mut [T])>
pub const fn split_last_chunk<const N: usize>(&self) -> Option<(&[T], &[T; N])>
pub fn split_last_chunk_mut<const N: usize>( &mut self ) -> Option<(&mut [T], &mut [T; N])>
pub fn last_chunk<const N: usize>(&self) -> Option<&[T; N]>
pub fn last_chunk_mut<const N: usize>(&mut self) -> Option<&mut [T; N]>
let x = &[0, 1, 2];
if let Some((first, elements)) = x.split_first_chunk::<2>() {
    assert_eq!(first, &[0, 1]);
    assert_eq!(elements, &[2]);
}
assert_eq!(None, x.split_first_chunk::<4>());

// 安全的返回 slice中元素（s[index] 当 index 不在范围时会 panic ）
pub fn get<I>(&self, index: I) -> Option<&<I as SliceIndex<[T]>>::Output> where I: SliceIndex<[T]>
pub fn get_mut<I>( &mut self, index: I ) -> Option<&mut <I as SliceIndex<[T]>>::Output> where I: SliceIndex<[T]>
pub unsafe fn get_unchecked<I>( &self, index: I ) -> &<I as SliceIndex<[T]>>::Output where I: SliceIndex<[T]>
pub unsafe fn get_unchecked_mut<I>( &mut self, index: I ) -> &mut <I as SliceIndex<[T]>>::Output where I: SliceIndex<[T]>
let v = [10, 40, 30];
assert_eq!(Some(&40), v.get(1));
assert_eq!(Some(&[10, 40][..]), v.get(0..2));
assert_eq!(None, v.get(3));
assert_eq!(None, v.get(0..4));

pub const fn as_ptr(&self) -> *const T
pub const fn as_mut_ptr(&mut self) -> *mut T
let x = &[1, 2, 4];
let x_ptr = x.as_ptr();
unsafe {
    for i in 0..x.len() {
        assert_eq!(x.get_unchecked(i), &*x_ptr.add(i));
    }
}
let x = &mut [1, 2, 4];
let x_ptr = x.as_mut_ptr();
unsafe {
    for i in 0..x.len() {
        *x_ptr.add(i) += 2;
    }
}
assert_eq!(x, &[3, 4, 6]);

// 返回包含所有元素的原始指针的区间（因为 slice 内存空间连续）
pub const fn as_ptr_range(&self) -> Range<*const T>
pub const fn as_mut_ptr_range(&mut self) -> Range<*mut T>
let a = [1, 2, 3];
let x = &a[1] as *const _;
let y = &5 as *const _;
assert!(a.as_ptr_range().contains(&x));
assert!(!a.as_ptr_range().contains(&y));

pub fn swap(&mut self, a: usize, b: usize)
let mut v = ["a", "b", "c", "d", "e"];
v.swap(2, 4);
assert!(v == ["a", "b", "e", "d", "c"]);
pub unsafe fn swap_unchecked(&mut self, a: usize, b: usize)

pub fn reverse(&mut self)

// 返回可迭代对象
pub fn iter(&self) -> Iter<'_, T>
pub fn iter_mut(&mut self) -> IterMut<'_, T>
pub fn windows(&self, size: usize) -> Windows<'_, T>  // 可重叠，如果元素数量比窗口小，则返回 None
let slice = ['l', 'o', 'r', 'e', 'm'];
let mut iter = slice.windows(3);
assert_eq!(iter.next().unwrap(), &['l', 'o', 'r']);
assert_eq!(iter.next().unwrap(), &['o', 'r', 'e']);
assert_eq!(iter.next().unwrap(), &['r', 'e', 'm']);
assert!(iter.next().is_none());
let slice = ['f', 'o', 'o'];
let mut iter = slice.windows(4);
assert!(iter.next().is_none());

pub fn chunks(&self, chunk_size: usize) -> Chunks<'_, T> // 不重叠
pub fn chunks_mut(&mut self, chunk_size: usize) -> ChunksMut<'_, T>
pub fn chunks_exact(&self, chunk_size: usize) -> ChunksExact<'_, T>
pub fn chunks_exact_mut(&mut self, chunk_size: usize) -> ChunksExactMut<'_, T>
pub const unsafe fn as_chunks_unchecked<const N: usize>(&self) -> &[[T; N]]
let slice = ['l', 'o', 'r', 'e', 'm'];
let mut iter = slice.chunks(2);
assert_eq!(iter.next().unwrap(), &['l', 'o']);
assert_eq!(iter.next().unwrap(), &['r', 'e']);
assert_eq!(iter.next().unwrap(), &['m']);
assert!(iter.next().is_none());

let slice = ['l', 'o', 'r', 'e', 'm'];
let mut iter = slice.chunks_exact(2);
assert_eq!(iter.next().unwrap(), &['l', 'o']);
assert_eq!(iter.next().unwrap(), &['r', 'e']);
assert!(iter.next().is_none()); //如果最后一波元素少与数量，则返回 None，可以使用 remainer() 方法来获取
assert_eq!(iter.remainder(), &['m']);

// 将 slice 分为 N 个元素数组的 slice 和最后剩下的元素 slice
pub const fn as_chunks<const N: usize>(&self) -> (&[[T; N]], &[T])
pub const fn as_rchunks<const N: usize>(&self) -> (&[T], &[[T; N]])
#![feature(slice_as_chunks)]
let slice = ['l', 'o', 'r', 'e', 'm'];
let (chunks, remainder) = slice.as_chunks();
assert_eq!(chunks, &[['l', 'o'], ['r', 'e']]);
assert_eq!(remainder, &['m']);
#![feature(slice_as_chunks)]
let slice = ['R', 'u', 's', 't'];
let (chunks, []) = slice.as_chunks::<2>() else { // 使用 let-else 来匹配剩下元素的列表
    panic!("slice didn't have even length")
};
assert_eq!(chunks, &[['R', 'u'], ['s', 't']]);

// array_chunks 是 chunks_exact 的泛型常量版本
pub fn array_chunks<const N: usize>(&self) -> ArrayChunks<'_, T, N>
#![feature(array_chunks)]
let slice = ['l', 'o', 'r', 'e', 'm'];
let mut iter = slice.array_chunks();
assert_eq!(iter.next().unwrap(), &['l', 'o']);
assert_eq!(iter.next().unwrap(), &['r', 'e']);
assert!(iter.next().is_none());
assert_eq!(iter.remainder(), &['m']);

pub const unsafe fn as_chunks_unchecked_mut<const N: usize>( &mut self ) -> &mut [[T; N]]
pub const fn as_chunks_mut<const N: usize>( &mut self ) -> (&mut [[T; N]], &mut [T])
pub const fn as_rchunks_mut<const N: usize>( &mut self) -> (&mut [T], &mut [[T; N]])
pub fn array_chunks_mut<const N: usize>(&mut self) -> ArrayChunksMut<'_, T, N>
pub fn array_windows<const N: usize>(&self) -> ArrayWindows<'_, T, N>
pub fn rchunks(&self, chunk_size: usize) -> RChunks<'_, T>
pub fn rchunks_mut(&mut self, chunk_size: usize) -> RChunksMut<'_, T>
pub fn rchunks_exact(&self, chunk_size: usize) -> RChunksExact<'_, T>
pub fn rchunks_exact_mut(&mut self, chunk_size: usize) -> RChunksExactMut<'_, T>

//使用 pred 来分割 slice（不重合的分割）
pub fn chunk_by<F>(&self, pred: F) -> ChunkBy<'_, T, F> where F: FnMut(&T, &T) -> bool
pub fn chunk_by_mut<F>(&mut self, pred: F) -> ChunkByMut<'_, T, F> where F: FnMut(&T, &T) -> pub
let slice = &[1, 1, 1, 3, 3, 2, 2, 2];
let mut iter = slice.chunk_by(|a, b| a == b);
assert_eq!(iter.next(), Some(&[1, 1, 1][..]));
assert_eq!(iter.next(), Some(&[3, 3][..]));
assert_eq!(iter.next(), Some(&[2, 2, 2][..]));
assert_eq!(iter.next(), None);

// 在指定的 index 位置拆分 slice
bool const fn split_at(&self, mid: usize) -> (&[T], &[T])
pub fn split_at_mut(&mut self, mid: usize) -> (&mut [T], &mut [T])
pub const unsafe fn split_at_unchecked(&self, mid: usize) -> (&[T], &[T])
pub unsafe fn split_at_mut_unchecked( &mut self, mid: usize ) -> (&mut [T], &mut [T])
pub fn split_at_checked(&self, mid: usize) -> Option<(&[T], &[T])>
pub fn split_at_mut_checked( &mut self, mid: usize ) -> Option<(&mut [T], &mut [T])>
let v = [1, 2, 3, 4, 5, 6];
{
    let (left, right) = v.split_at(0);
    assert_eq!(left, []);
    assert_eq!(right, [1, 2, 3, 4, 5, 6]);
}
{
    let (left, right) = v.split_at(2);
    assert_eq!(left, [1, 2]);
    assert_eq!(right, [3, 4, 5, 6]);
}
{
    let (left, right) = v.split_at(6);
    assert_eq!(left, [1, 2, 3, 4, 5, 6]);
    assert_eq!(right, []);
}

//  使用指定的 pred 分割 slice，可能会导致空 slice
pub fn split<F>(&self, pred: F) -> Split<'_, T, F> where F: FnMut(&T) -> bool
pub fn split_mut<F>(&mut self, pred: F) -> SplitMut<'_, T, F> where F: FnMut(&T) -> bool
pub fn split_inclusive<F>(&self, pred: F) -> SplitInclusive<'_, T, F> where F: FnMut(&T) -> bool
pub fn split_inclusive_mut<F>(&mut self, pred: F) -> SplitInclusiveMut<'_, T, F> where F: FnMut(&T) -> bool
pub fn rsplit<F>(&self, pred: F) -> RSplit<'_, T, F> where F: FnMut(&T) -> bool
pub fn rsplit_mut<F>(&mut self, pred: F) -> RSplitMut<'_, T, F> where F: FnMut(&T) -> bool
pub fn splitn<F>(&self, n: usize, pred: F) -> SplitN<'_, T, F> where F: FnMut(&T) -> bool
pub fn splitn_mut<F>(&mut self, n: usize, pred: F) -> SplitNMut<'_, T, F> where    F: FnMut(&T) -> bool
pub fn rsplitn<F>(&self, n: usize, pred: F) -> RSplitN<'_, T, F> where    F: FnMut(&T) -> bool
pub fn rsplitn_mut<F>(&mut self, n: usize, pred: F) -> RSplitNMut<'_, T, F> where    F: FnMut(&T) -> bool
pub fn split_once<F>(&self, pred: F) -> Option<(&[T], &[T])> where    F: FnMut(&T) -> bool
pub fn rsplit_once<F>(&self, pred: F) -> Option<(&[T], &[T])> where    F: FnMut(&T) -> bool
let slice = [10, 6, 33, 20];
let mut iter = slice.split(|num| num % 3 == 0);
assert_eq!(iter.next().unwrap(), &[10]);
assert_eq!(iter.next().unwrap(), &[]);
assert_eq!(iter.next().unwrap(), &[20]);
assert!(iter.next().is_none());

let slice = [10, 40, 33];
let mut iter = slice.split(|num| num % 3 == 0);
assert_eq!(iter.next().unwrap(), &[10, 40]);
assert_eq!(iter.next().unwrap(), &[]); // 结尾空 slice
assert!(iter.next().is_none());
let slice = [10, 40, 33, 20];
let mut iter = slice.split_inclusive(|num| num % 3 == 0);
assert_eq!(iter.next().unwrap(), &[10, 40, 33]);
assert_eq!(iter.next().unwrap(), &[20]);
assert!(iter.next().is_none());
let v = [10, 40, 30, 20, 60, 50];
for group in v.splitn(2, |num| *num % 3 == 0) {
    println!("{group:?}");
}
#![feature(slice_split_once)]
let s = [1, 2, 3, 2, 4];
assert_eq!(s.split_once(|&x| x == 2), Some((
    &[1][..],
    &[3, 2, 4][..]
)));
assert_eq!(s.split_once(|&x| x == 0), None);


pub fn contains(&self, x: &T) -> bool where T: PartialEq
let v = [10, 40, 30];
assert!(v.contains(&30));
assert!(!v.contains(&50));

pub fn starts_with(&self, needle: &[T]) -> bool where T: PartialEq
pub fn ends_with(&self, needle: &[T]) -> bool where T: PartialEq
let v = [10, 40, 30];
assert!(v.starts_with(&[10]));
assert!(v.starts_with(&[10, 40]));
assert!(!v.starts_with(&[50]));
assert!(!v.starts_with(&[10, 50]));

pub fn strip_prefix<P>(&self, prefix: &P) -> Option<&[T]> where P: SlicePattern<Item = T> + ?Sized, T: PartialEq
pub fn strip_suffix<P>(&self, suffix: &P) -> Option<&[T]> where P: SlicePattern<Item = T> + ?Sized, T: PartialEq
let v = &[10, 40, 30];
assert_eq!(v.strip_prefix(&[10]), Some(&[40, 30][..]));
assert_eq!(v.strip_prefix(&[10, 40]), Some(&[30][..]));
assert_eq!(v.strip_prefix(&[50]), None);
assert_eq!(v.strip_prefix(&[10, 50]), None);
let prefix : &str = "he";
assert_eq!(b"hello".strip_prefix(prefix.as_bytes()), Some(b"llo".as_ref()));

pub fn binary_search(&self, x: &T) -> Result<usize, usize> where T: Ord
pub fn binary_search_by<'a, F>(&'a self, f: F) -> Result<usize, usize> where F: FnMut(&'a T) -> Ordering
pub fn binary_search_by_key<'a, B, F>(&'a self, b: &B,f:F) -> Result<usize, usize> where F: FnMut(&'a T) -> B,B: Ord

// 对 slice 进行排序，unstable 表示不保证重复元素的顺序
pub fn sort_unstable(&mut self) where T: Ord
pub fn sort_unstable_by<F>(&mut self, compare: F) where    F: FnMut(&T, &T) -> Ordering
pub fn sort_unstable_by_key<K, F>(&mut self, f: F) where    F: FnMut(&T) -> K,    K: Ord
pub fn select_nth_unstable(    &mut self,    index: usize) -> (&mut [T], &mut T, &mut [T]) where    T: Ord
pub fn select_nth_unstable_by<F>(    &mut self,    index: usize,    compare: F) -> (&mut [T], &mut T, &mut [T]) where F: FnMut(&T, &T) -> Ordering
pub fn select_nth_unstable_by_key<K, F>( &mut self, index: usize, f: F ) -> (&mut [T], &mut T, &mut [T]) where F: FnMut(&T) -> K, K: Ord
let mut v = [-5, 4, 1, -3, 2];
v.sort_unstable();
assert!(v == [-5, -3, 1, 2, 4]);

// 返回两个 slice，分别是没有重复的元素，重复的元素（没有顺序）
pub fn partition_dedup(&mut self) -> (&mut [T], &mut [T]) where T: PartialEq
pub fn partition_dedup_by<F>(&mut self, same_bucket: F) -> (&mut [T], &mut [T]) where F: FnMut(&mut T, &mut T) -> bool
pub fn partition_dedup_by_key<K, F>(&mut self, key: F) -> (&mut [T], &mut [T]) where F: FnMut(&mut T) -> K, K: PartialEq
#![feature(slice_partition_dedup)]
let mut slice = [1, 2, 2, 3, 3, 2, 1, 1];
let (dedup, duplicates) = slice.partition_dedup();
assert_eq!(dedup, [1, 2, 3, 2, 1]);
assert_eq!(duplicates, [2, 3, 1]);

// 向左轮转两个元素
pub fn rotate_left(&mut self, mid: usize)
pub fn rotate_right(&mut self, k: usize)
let mut a = ['a', 'b', 'c', 'd', 'e', 'f'];
a.rotate_left(2);
assert_eq!(a, ['c', 'd', 'e', 'f', 'a', 'b']);

// 使用指定值填充
pub fn fill(&mut self, value: T) where    T: Clone
// 使用指定函数返回值填充
pub fn fill_with<F>(&mut self, f: F) where    F: FnMut() -> T
let mut buf = vec![0; 10];
buf.fill(1);
assert_eq!(buf, vec![1; 10]);

// 从 src clone 元素到 self，src 和 self 的长度必须一致
pub fn clone_from_slice(&mut self, src: &[T]) where    T: Clone
pub fn copy_from_slice(&mut self, src: &[T]) where    T: Copy
let src = [1, 2, 3, 4];
let mut dst = [0, 0];
// Because the slices have to be the same length, we slice the source slice from four elements to
// two. It will panic if we don't do this.
dst.clone_from_slice(&src[2..]);
assert_eq!(src, [1, 2, 3, 4]);
assert_eq!(dst, [3, 4]);

// 使用 memmove 将 src 的范围元素移动到 dest 开始的位置，两者可以有重复
pub fn copy_within<R>(&mut self, src: R, dest: usize) where    R: RangeBounds<usize>, T: Copy
let mut bytes = *b"Hello, World!";
bytes.copy_within(1..5, 8);
assert_eq!(&bytes, b"Hello, Wello!");

// 交换内容，两个 slice 的长度必须一致
pub fn swap_with_slice(&mut self, other: &mut [T])
let mut slice1 = [0, 0];
let mut slice2 = [1, 2, 3, 4];
slice1.swap_with_slice(&mut slice2[2..]);
assert_eq!(slice1, [3, 4]);
assert_eq!(slice2, [1, 2, 0, 0]);

pub unsafe fn align_to<U>(&self) -> (&[T], &[U], &[T])
pub unsafe fn align_to_mut<U>(&mut self) -> (&mut [T], &mut [U], &mut [T])

pub fn as_simd<const LANES: usize>(&self) -> (&[T], &[Simd<T, LANES>], &[T]) where Simd<T, LANES>: AsRef<[T; LANES]>, T: SimdElement, LaneCount<LANES>: SupportedLaneCount
pub fn as_simd_mut<const LANES: usize>( &mut self ) -> (&mut [T], &mut [Simd<T, LANES>], &mut [T]) where Simd<T, LANES>: AsMut<[T; LANES]>, T: SimdElement, LaneCount<LANES>: SupportedLaneCount

pub fn is_sorted(&self) -> bool where T: PartialOrd
pub fn is_sorted_by<'a, F>(&'a self, compare: F) -> bool where F: FnMut(&'a T, &'a T) -> bool
pub fn is_sorted_by_key<'a, F, K>(&'a self, f: F) -> bool where F: FnMut(&'a T) -> K, K: PartialOrd

pub fn partition_point<P>(&self, pred: P) -> usize where P: FnMut(&T) -> bool
let v = [1, 2, 3, 3, 5, 6, 7];
let i = v.partition_point(|&x| x < 5);
assert_eq!(i, 4);
assert!(v[..i].iter().all(|&x| x < 5));
assert!(v[i..].iter().all(|&x| !(x < 5)));
let a = [2, 4, 8];
assert_eq!(a.partition_point(|x| x < &100), a.len());
let a: [i32; 0] = [];
assert_eq!(a.partition_point(|x| x < &100), 0);

// 从 self 拿出 range 元素并返回，self 是剩下的元素
pub fn take<R, 'a>(self: &mut &'a [T], range: R) -> Option<&'a [T]> where R: OneSidedRange<usize>
pub fn take_mut<R, 'a>(self: &mut &'a mut [T], range: R) -> Option<&'a mut [T]> where R: OneSidedRange<usize>
pub fn take_first<'a>(self: &mut &'a [T]) -> Option<&'a T>
pub fn take_first_mut<'a>(self: &mut &'a mut [T]) -> Option<&'a mut T>
pub fn take_last<'a>(self: &mut &'a [T]) -> Option<&'a T>
pub fn take_last_mut<'a>(self: &mut &'a mut [T]) -> Option<&'a mut T>
#![feature(slice_take)]
let mut slice: &[_] = &['a', 'b', 'c', 'd'];
let mut first_three = slice.take(..3).unwrap();
assert_eq!(slice, &['d']);
assert_eq!(first_three, &['a', 'b', 'c']);
#![feature(slice_take)]
let mut slice: &[_] = &['a', 'b', 'c', 'd'];
let mut tail = slice.take(2..).unwrap();
assert_eq!(slice, &['a', 'b']);
assert_eq!(tail, &['c', 'd']);
#![feature(slice_take)]
let mut slice: &[_] = &['a', 'b', 'c'];
let first = slice.take_first().unwrap();
assert_eq!(slice, &['b', 'c']);
assert_eq!(first, &'a');

pub unsafe fn get_many_unchecked_mut<const N: usize>( &mut self, indices: [usize; N] ) -> [&mut T; N]
pub fn get_many_mut<const N: usize>( &mut self, indices: [usize; N] ) -> Result<[&mut T; N], GetManyMutError<N>>
#![feature(get_many_mut)]
let v = &mut [1, 2, 3];
if let Ok([a, b]) = v.get_many_mut([0, 2]) {
    *a = 413;
    *b = 612;
}
assert_eq!(v, &[413, 2, 612]);
```

其他 slice 方法：

```rust
impl<T> [T]

pub fn sort(&mut self) where T: Ord
let mut v = [-5, 4, 1, -3, 2];
v.sort();
assert!(v == [-5, -3, 1, 2, 4]);

pub fn sort_by<F>(&mut self, compare: F) where F: FnMut(&T, &T) -> Ordering
pub fn sort_by_key<K, F>(&mut self, f: F) where F: FnMut(&T) -> K, K: Ord
pub fn sort_by_cached_key<K, F>(&mut self, f: F) where F: FnMut(&T) -> K, K: Ord
let mut v = [-5i32, 4, 1, -3, 2];
v.sort_by_key(|k| k.abs());
assert!(v == [1, 2, -3, 4, -5]);

// 从 slice 生成 vec
pub fn to_vec(&self) -> Vec<T> where T: Clone
let s = [10, 40, 30];
let x = s.to_vec();
// Here, `s` and `x` can be modified independently.

pub fn to_vec_in<A>(&self, alloc: A) -> Vec<T, A> where A: Allocator, T: Clone

pub fn into_vec<A>(self: Box<[T], A>) -> Vec<T, A> where A: Allocator
let s: Box<[i32]> = Box::new([10, 40, 30]);
let x = s.into_vec();
// `s` cannot be used anymore because it has been converted into `x`.
assert_eq!(x, vec![10, 40, 30]);

pub fn repeat(&self, n: usize) -> Vec<T> where T: Copy
assert_eq!([1, 2].repeat(3), vec![1, 2, 1, 2, 1, 2]);

// 将 slice T 打平为一个值 Self::Output
pub fn concat<Item>(&self) -> <[T] as Concat<Item>>::Output where [T]: Concat<Item>, Item: ?Sized
assert_eq!(["hello", "world"].concat(), "helloworld");
assert_eq!([[1, 2], [3, 4]].concat(), [1, 2, 3, 4]);

// 使用指定分隔符打平 slice T
pub fn join<Separator>( &self, sep: Separator) -> <[T] as Join<Separator>>::Output where [T]: Join<Separator>
assert_eq!(["hello", "world"].join(" "), "hello world");
assert_eq!([[1, 2], [3, 4]].join(&0), [1, 2, 0, 3, 4]);
assert_eq!([[1, 2], [3, 4]].join(&[0, 0][..]), [1, 2, 0, 0, 3, 4]);

// 不建议使用，被 join 代替
pub fn connect<Separator>( &self, sep: Separator ) -> <[T] as Join<Separator>>::Output where [T]: Join<Separator>
```


## <span class="section-num">7</span> tuple {#tuple}

tuple 中的元素类型可以不同,格式为 (T1, T2, T3), 可以使用 pattern match 进行析构, 这使得元组非常灵活和强大，非常适合于存储和传递一组异构数据。元组也可以作为函数的返回值, 或者将数据组织成单个复合类型。

元组的特性

1.  类型多样性：元组可以包含不同类型的数据。
2.  固定大小：一旦定义，元组的大小和类型就固定下来，不能更改。
3.  索引访问：可以通过索引来访问元组的特定元素。

<!--listend-->

```rust
fn main() {
    let _t0: (u8,i16) = (0, -1);
    let _t1: (u8, (i16, u32)) = (0, (-1, 1));
    let t: (u8, u16, i64, &str, String) = (1u8, 2u16, 3i64, "hello", String::from(", world"));
    println!("Success!");
}

// 函数接受一个元组作为参数，并返回一个元组
fn swap(tup: (i32, f64)) -> (f64, i32) {
    // 返回一个新的元组，元素顺序与输入相反
    (tup.1, tup.0)
}
let input_tup = (123, 4.56);
let output_tup = swap(input_tup);

// 创建一个嵌套的元组结构
let nested_tup = (1, (2, 3), 4);
// 访问嵌套元组中的元素
let (a, (b, c), d) = nested_tup;
// 创建一个零元素的元组，也称为单元类型。
let unit = ();
```

tuple 拥有其中的各元素对象, 和 struct 一样, 允许部分元素被 move 走, 但是后续不能再访问已经 move 的元素:

-   array/vec/slice 等集合不允许元素被 move 走.
-   具体参考: [2](#org-target--partial-move)

使用 index 访问元素, 如 t.0, t.1 等.

析构 tuple: 对于 enum 类型是在枚举 variant 值外部而非内部类匹配 &amp; 或 &amp;mut 的， 对于 tuple 类型也是在
tuple 外部匹配 &amp; 或 &amp;mut 的:

```rust
let x: &Option<i32> = &Some(3);

// OK: 等效为 Some(ref y), y 的类型是 &i32
if let Some(y) = x {}
// OK: 在 variant 外指定 &，y 的类型是 i32
if let &Some(y) = x {}
// ERROR: 不能在 variant 内指定 &，expected `i32`, found `&_`
if let Some(&y) = x {}

let (a, b ) = &(1, 2); // a 和 b 都是 &i32 类型
println!("Results: {a} {b}");

let &(c, d ) = &(1, 2); // c 和 d 都是 i32 类型
println!("Results: {c} {d}");

let (&c, d ) = &(1, 2); // 报错
let (ref c, d ) = &(1, 2); // OK

// 另一个例子
enum MyEnum {
    A { name: String, x: u8 },
    B { name: String },
}
fn a_to_b(e: &mut MyEnum) {
    if let MyEnum::A {
        name,  // name 和 x 都是析构后的变量名，可以在后面的 block 中使用。name 是 &mut String 类型。
        x: 0,
    } = e {
        *e = MyEnum::B {
            name: std::mem::take(name), // take 参数类型是 &mut T, 而 name 类型是 &mut String 故满足
        }
    }
    // if let &mut MyEnum::A {
    //     name,  // OK: name 是 String 类型
    //     x: 0,
    // } = e
}
```

过长的 tuple 不能被格式化输出:

```rust
fn main() {
    let too_long_tuple = (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12);  // 最多 12 个元素才能被格式化
    println!("too long tuple: {:?}", too_long_tuple);
}
```

单个元素: (T,) 不能是 (T), 以免和函数参数混淆.

多个元素, 最后可加可选的 , 号, 如: (1, 2,).

空 tuple () 也称为 unit type, 只有一个空值 ().


## <span class="section-num">8</span> const/static/lazy_static! {#const-static-lazy-static}

rust 支持两种 const 常量，可以在全局或任意 scope 中声明（和变量一样，必须先声明并初始化后才可以使用）：

1.  const：不可变值；
2.  static：可能可变的（static mut）, 默认带 'static lifetime，需要在 unsafe 中读写 static mut 值；
3.  全局常量需要使用 `全大写名称` ，否则编译器警告；

<!--listend-->

```rust
// Globals are declared outside all other scopes.
const THRESHOLD: i32 = 10; // 全局常量
static LANGUAGE: &str = "Rust"; // 全局常量，默认带 'static
// 全局 static 可变变量， 需要在 unsafe 代码中访问
static mut stat_mut = "abc";

fn is_big(n: i32) -> bool {
    // Access constant in some function
    n > THRESHOLD
}

fn main() {
    let n = 16;
    // Access constant in the main thread
    println!("This is {}", LANGUAGE);
    println!("The threshold is {}", THRESHOLD);
    println!("{} is {}", n, if is_big(n) { "big" } else { "small" });

    // Error! Cannot modify a `const`.
    THRESHOLD = 5;
}
```

也可以定义 const 函数, 但 const 函数有一些限制:

1.  内部只能调用其它的 const 函数;
2.  不能分配内存和操作原始指针(即使在 unsafe block 中也不行);
3.  除了声明周期外,不能使用其他类型作为泛型参数;

对 const/static 变量的初始化, 只能使用 const 函数/tuple 类型:

-   如各种原子类型的 new 方法, 如 AtomicUsize::new() , 以及 String::new() 是 const fn，新版的
    Mutex::new() 也是.
-   用 Mutex/Atomic/RwMutex 等内部可变性机制,可以对全局 static 进行修改.

<!--listend-->

```rust
use std::sync::atomic::AtomicUsize;
static PACKETS_SERVED: AtomicUsize = AtomicUsize::new(0); // ok
static MY_GLOBAL: Vec<usize> = Vec::new(); // OK, 但是不可修改。

use std::sync::Mutex;
static HOSTNAME: Mutex<String> = Mutex::new(String::new()); // ok, HOSTNAME 可以修改
fn main() {
    let mut name =  HOSTNAME.lock().unwrap();
    name.push_str("localhost");
    println!("Results: {name}");
}
```

解决办法：使用 lazy_static! 宏定义静态变量，可以使用任何表达式进行初始化，表达式会在变量第一次解引用时运行，值会被存储在变量中以便后续使用。

-   使用 lazy_static! 会导致每次访问静态数据有微小的性能开销。它的实现里使用 了 std::sync::Once，它是一种用于一次性初始化的底层同步原语。
-   在幕后，每一次访 问一个惰性静态变量时，程序都会执行一个原子 load 指令来检查是否已经初始化过。

<!--listend-->

```rust
use std::sync::Mutex;
lazy_static! {
    static ref HOSTNAME: Mutex<String> = Mutex::new(String::new());
}
```


## <span class="section-num">9</span> pointer {#pointer}

Rust 提供如下几种指针类型：

1.  引用（Reference）: &amp;T 和 &amp;mut T
2.  裸指针（Raw Pointer）: \*const T 和 \*mut T
3.  智能指针（Smart Pointer）: 如 Box&lt;T&gt;, Rc&lt;T&gt;, Arc&lt;T&gt; 和 RefCell&lt;T&gt; 等。

引用是最常用的指针类型，它们被广泛用于借用值，而裸指针和智能指针用于更特殊的场景。智能指针的使用是安全的，它们封装了很多底层的细节；而裸指针的使用则需要显式地在 unsafe 代码块中指定。

引用是借用值的安全指针，它们分为不可变引用 (&amp;T) 和可变引用 (&amp;mut T)。

```rust
fn main() {
    let x = 5;
    let y = &x; // 不可变引用

    let mut z = 10;
    let w = &mut z; // 可变引用，借用的值必须是 mut 类型
    *w += 1; // 解引用来修改值

    println!("x: {}, y: {}, z: {}, w: {}", x, y, z, w);
}
```

裸指针（Raw Pointer）可以是不可变 (\*const T) 或可变 (\*mut T)，它们与 C 语言中的指针相似，但它们的使用不受安全检查。因此，裸指针的使用需要 unsafe 代码块。

```rust
fn main() {
    let mut x = 10;
    let ptr_x = &mut x as *mut i32; // 将借用转换为可变裸指针

    unsafe {
        // 在 unsafe 代码块中使用裸指针
        *ptr_x += 10;
        println!("x: {}", *ptr_x);
    }
}
```

智能指针在 Rust 中是一些实现了 Deref 和 Drop trait 的结构体，用于额外的元数据和功能。Box 是最简单的智能指针，用来分配堆上的值。

```rust
fn main() {
    let b = Box::new(5); // 在堆上分配一个i32值
    println!("b: {}", b);

    let rc = Rc::new(5); // 创建一个引用计数指针
    let rc_clone = rc.clone(); // 增加引用计数
    println!("rc: {}, rc_clone: {}", rc, rc_clone);
}
```


## <span class="section-num">10</span> struct {#struct}

struct/enum/union 是 rust 的三种自定义类型。自定义类型名必须是 CamelCase，否则 rustc 会 warnning。

struct 和 enum 一样有三种类型:

1.  unit struct，不含任何 field： `struct MyStruct;`
2.  tuple struct： `struct MyStruct(T1, T2);`
    -   特殊的只有一个元素 T 的 struct 称为 newtype；
3.  C-like struct： `struct MyStruct{field1: type1, field2: type2};`

<!--listend-->

```rust
// An attribute to hide warnings for unused code.
#![allow(dead_code)]

#[derive(Debug)]
struct Person {
    name: String,
    age: u8,
}

struct Unit;
struct Pair(i32, f32);
struct Point {
    x: f32,
    y: f32,
}

// 实例化
let _unit = Unit; // 对于 unit struct，只有唯一的一个对象。
let pair = Pair(1, 0.1);   // 初始化 tuple struct 时，类似于函数调用。

let Pair(integer, decimal) = pair;  // 解构 struct，注意前面的 Pair 不能省。
```

在初始化 struct 对象时, 必须列出每一个 field:

-   与 field 同名的变量赋值, 可以使用简写形式；
-   可以使用某个 struct 对象展开来快速创建一个新的 struct 对象, 它必须位于新 struct 初始化的最后一项,
    不能包含结尾的分号;

<!--listend-->

```rust
fn main() {
    struct Person {
        name: String,
        age: u8,
        hobby: String,
    }

    let age = 30;
    let p = Person { // Error：missing field `hobby` in initializer of `Person`
        name: String::from("sunface"),
        age, // 与 field 同名的变量赋值, 可以使用简写形式。
    };
    println!("Success!");
}

// Create struct with field init shorthand
let name = String::from("Peter");
let age = 27;
let peter = Person { name, age }; // 同名的 field 可以简写。

// newtype idiom, 一般为其他类型添加方法
struct Years(i64);
struct Days(i64);
impl Years {
    pub fn to_days(&self) -> Days {
        Days(self.0 * 365)
    }
}

impl Days {
    /// truncates partial years
    pub fn to_years(&self) -> Years {
        Years(self.0 / 365)
    }
}

fn old_enough(age: &Years) -> bool {
    age.0 >= 18
}

fn main() {
    let age = Years(5);
    let age_days = age.to_days();
    println!("Old enough {}", old_enough(&age));
    println!("Old enough {}", old_enough(&age_days.to_years()));
    // println!("Old enough {}", old_enough(&age_days));
}

// 使用 struct 对象初始化另一个 struct 对象。
#[derive(Debug)]
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}
fn main() {
    let u1 = User {
        email: String::from("someone@example.com"),
        username: String::from("sunface"),
        active: true,
        sign_in_count: 1,
    };
    let u2 = set_email(u1);
    println!("Success! {u2:?}");
}
fn set_email(u: User) -> User {
    User {
        email: String::from("contact@im.dev"),
        ..u // u 必须位于最后, 且不能有逗号结尾。
    }
}
// Make a new point by using struct update syntax to use the fields of our other one
let bottom_right = Point { x: 5.2, ..point };
```

无 field 的 struct MyStruct；示例：

```rust
// Non-copyable types.
struct Empty;
struct Null;

// A trait generic over `T`.
trait DoubleDrop<T> {
    // Define a method on the caller type which takes an additional single parameter `T` and does
    // nothing with it.
    fn double_drop(self, _: T);
}

// Implement `DoubleDrop<T>` for any generic parameter `T` and caller `U`.
impl<T, U> DoubleDrop<T> for U {
    // This method takes ownership of both passed arguments, deallocating both.
    fn double_drop(self, _: T) {}
}

fn main() {
    let empty = Empty; // Empty 是 struct Empty; 的唯一值;
    let null  = Null;
    empty.double_drop(null);
}
```

由于 struct 会 owner 对应的 field value, 所以 field 一般使用 owneed 类型而非 &amp;T/&amp;mut T 类型('static
除外), 因为后者需要声明生命周期参数:

```rust
  struct Person {
      name: String, // name 和 hobby 都是 Owner 类型, 而不是 &str;
      hobby: String,
  }
```

struct 整体和各 field 需要单独设置 public (enum 是整体 public 即可), 没有设置 public 的 filed 默认是私有的, 其他 moudule 不能访问.

struct 默认没有实现 Copy/Clone 以及 Debug, 可以通过 derive 宏来让 rustc 编译器自动生成.

-   不能通过 derive 属性来生成 Display trait, 需要手动实现该 trait.

struct 在被 Destructure 时，如果 filed 没有实现 Copy，这可能会被 partial move, move 的 field 后续不能再访问:

-   enum 也可以被 partial move；
-   array/tuple/vec 元素不能被 partial move；

<!--listend-->

```rust
fn main() {
    #[derive(Debug)]
    struct Person {
        name: String,
        age: Box<u8>,
    }

    let person = Person {
        name: String::from("Alice"),
        age: Box::new(20),
    };

    // `name` is moved out of person, but `age` is referenced
    let Person { name, ref age } = person; // struct 可以作为 pattern match 来进行解构
    println!("The person's age is {}", age);
    println!("The person's name is {}", name);

    // Error! borrow of partially moved value: `person` partial move occurs
    //println!("The person struct is {:?}", person);
    // `person` cannot be used but `person.age` can be used as it is not moved
    println!("The person's age from person struct is {}", person.age);
}
```

struct 包含引用类型成员时需要明确指定 lifetime。嵌套带声明周期的 struct 时，外层 struct 也必须声明生命周期：

-   where 'a: 'b 表示 'a 的 lifetime 至少要比 'b 长。

<!--listend-->

```rust
// This does not compile.
struct S {
    r: &i32 // r 是引用类型，但是没有指定 lifetime anno
}
let s;
{
    let x = 10;
    s = S { r: &x };
}
assert_eq!(*s.r, 10); // bad: reads from dropped `x`


// 正确
struct S {
    r: &'static i32
}
// 正确
struct S<'a> {
    r: &'a i32  // r 引用对象的声明周期至少要比 struct S 大。
}
// 正确，多个 lifetime 参数
struct S<'a, 'b> {
    x: &'a i32,
    y: &'b i32
}
// 函数
fn f<'a, 'b>(r: &'a i32, s: &'b i32) -> &'a i32 { r } // looser


// 错误
struct D {
    s: S // not adequate
}
// 正确
struct D<'a> {
    s: S<'a>
}
```


## <span class="section-num">11</span> enum {#enum}

enum variant 可以包含（own）数据, 和 struct 类似，有 3 种类型:

1.  Quit;
2.  Quit {x: y, xx:yy}; // own struct 的 field 数据
3.  Quit (i32, String);

<!--listend-->

```rust
enum Number {
    Zero, // tag 默认在上一个基础上递增，第一个 tag 为 0。
    One,
    Two,
}

enum Number1 {
    Zero = 0,
    One,
    Two,
}

// C-like enum
enum Number2 {
    Zero = 0.0,
    One = 1.0,
    Two = 2.0,
}

// enum variant 可以包含数据。
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

特殊的空 enum (无 variant)不能作为 value 使用, 主要的使用场景是作为不可能发生错误的 Result，如标准库类型 std::convert::Infallible：

```rust
// std::convert::Infallible
pub enum Infallible {}

impl<T, U> TryFrom<U> for T where U: Into<T> {
    type Error = Infallible;

    fn try_from(value: U) -> Result<Self, Infallible> {
        Ok(U::into(value))  // Never returns `Err`
    }
}
```

enum variant 的数据可以用在 pattern match 中：

```rust
// Create an `enum` to classify a web event. Note how both names and type information together
// specify the variant: `PageLoad != PageUnload` and `KeyPress(char) != Paste(String)`.  Each is
// different and independent.
enum WebEvent {
    // An `enum` variant may either be `unit-like`,
    PageLoad,
    PageUnload,
    // like tuple structs,
    KeyPress(char),
    Paste(String),
    // or c-like structures.
    Click { x: i64, y: i64 },
}

// A function which takes a `WebEvent` enum as an argument and returns nothing.
fn inspect(event: WebEvent) {
    match event {
        WebEvent::PageLoad => println!("page loaded"),
        WebEvent::PageUnload => println!("page unloaded"),
        // Destructure `c` from inside the `enum` variant.
        WebEvent::KeyPress(c) => println!("pressed '{}'.", c),
        WebEvent::Paste(s) => println!("pasted \"{}\".", s),
        // Destructure `Click` into `x` and `y`.
        WebEvent::Click { x, y } => {
            println!("clicked at x={}, y={}.", x, y);
        },
    }
}

fn main() {
    // 创建一个 enum variant 时需要指定对应的类型值（tuple、struct）
    let pressed = WebEvent::KeyPress('x');
    // `to_owned()` creates an owned `String` from a string slice.
    let pasted  = WebEvent::Paste("my text".to_owned());
    let click   = WebEvent::Click { x: 20, y: 80 };
    let load    = WebEvent::PageLoad;
    let unload  = WebEvent::PageUnload;
    inspect(pressed);
    inspect(pasted);
    inspect(click);
    inspect(load);
    inspect(unload);
}
```

enum 的各 variant 都是 enum 类型, 所以可以用在 array 中:

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
fn main() {
    let msgs: [Message; 3] = [  // enum Message 作为类型, 可以在 array 中使用;
        Message::Quit,
        Message::Move{x:1, y:3},
        Message::ChangeColor(255,255,0)
    ];
    for msg in msgs {
        show_message(msg)
    }
}
fn show_message(msg: Message) {
    println!("{}", msg);
}
```

enum variant 可以包含 tag 表达式，可以使用 enum::variant as i32/u32 来获得 tag 值：

```rust
// An attribute to hide warnings for unused code.
#![allow(dead_code)]

// enum with implicit discriminator (starts at 0)
enum Number {
    Zero,  // 默认从 0 开始递增，未指定时在上一个基础上递增。
    One,
    Two,
}

// enum with explicit discriminator
enum Color {
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue // 上一基础上自动递增，所以为 0x00ff01
}

fn main() {
    // `enums` can be cast as integers.
    println!("zero is {}", Number::Zero as i32);
    println!("one is {}", Number::One as i32);
    println!("roses are #{:06x}", Color::Red as i32);
    println!("violets are #{:06x}", Color::Blue as i32);
}
```

如果 enum 名称太长，可以用 type alias 来简化：

```rust
enum VeryVerboseEnumOfThingsToDoWithNumbers {
    Add,
    Subtract,
}

// Creates a type alias
type Operations = VeryVerboseEnumOfThingsToDoWithNumbers;

fn main() {
    // We can refer to each variant via its alias, not its long and inconvenient name.
    let x = Operations::Add; // 使用简化的 enum 类型别名来访问 variant
}

// 最常见的场景是方法中的 Self 类型其实也是 type alias
enum VeryVerboseEnumOfThingsToDoWithNumbers {
    Add,
    Subtract,
}
impl VeryVerboseEnumOfThingsToDoWithNumbers {
    fn run(&self, x: i32, y: i32) -> i32 {
        match self {
            Self::Add => x + y,
            Self::Subtract => x - y,
        }
    }
}
```

enum 的 variant 可以使用 use 按需或一次性导入，这样不需要每次指定 enum::variant 的前面 enum:: 部分：

```rust
// An attribute to hide warnings for unused code.
#![allow(dead_code)]
enum Status {
    Rich,
    Poor,
}
enum Work {
    Civilian,
    Soldier,
}

fn main() {
    // Explicitly `use` each name so they are available without manual scoping.
    use crate::Status::{Poor, Rich};
    // Automatically `use` each name inside `Work`.
    use crate::Work::*;

    // Equivalent to `Status::Poor`.
    let status = Poor;
    // Equivalent to `Work::Civilian`.
    let work = Civilian;

    match status {
        // Note the lack of scoping because of the explicit `use` above.
        Rich => println!("The rich have lots of money!"),
        Poor => println!("The poor have no money..."),
    }

    match work {
        // Note again the lack of scoping.
        Civilian => println!("Civilians work!"),
        Soldier  => println!("Soldiers fight!"),
    }
}
```

enum 只需为整体指定 pub 可见性即可，各 variant 的可见性继承自整体。（struct 需要为每个 field 指定可见性）。

析构 enum：对于 enum 类型是在枚举 variant 值外部而非内部类匹配 &amp; 或 &amp;mut 的，

```rust
let x: &Option<i32> = &Some(3);

// OK: 等效为 Some(ref y), y 的类型是 &i32
if let Some(y) = x {}
// OK: 在 variant 外指定 &，y 的类型是 i32
if let &Some(y) = x {}
// ERROR: 不能在 variant 内指定 &，expected `i32`, found `&_`
if let Some(&y) = x {}

let (a, b ) = &(1, 2); // a 和 b 都是 &i32 类型
println!("Results: {a} {b}");

let &(c, d ) = &(1, 2); // c 和 d 都是 i32 类型
println!("Results: {c} {d}");

let (&c, d ) = &(1, 2); // 报错
let (ref c, d ) = &(1, 2); // OK

// 另一个例子
enum MyEnum {
    A { name: String, x: u8 },
    B { name: String },
}
fn a_to_b(e: &mut MyEnum) {
    if let MyEnum::A {
        name,  // name 和 x 都是析构后的变量名，可以在后面的 block 中使用。name 是 &mut String 类型。
        x: 0,
    } = e {
        *e = MyEnum::B {
            name: std::mem::take(name), // take 参数类型是 &mut T, 而 name 类型是 &mut String 故满足
        }
    }
    // if let &mut MyEnum::A {
    //     name,  // OK: name 是 String 类型
    //     x: 0,
    // } = e
}
```


### <span class="section-num">11.1</span> Option/Result/Error {#option-result-error}

Option/Result 都是 enum 类型，也支持迭代（实现了 IntoIterator），效果就如一个或 0 个元素。

Return consumed argument on error: If a fallible function consumes (moves) an argument, return that
argument back inside an error.

```rust
// https://rust-unofficial.github.io/patterns/idioms/return-consumed-arg-on-error.html
pub fn send(value: String) -> Result<(), SendError> {
    println!("using {value} in a meaningful way");
    // Simulate non-deterministic fallible action.
    use std::time::SystemTime;
    let period = SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH)
        .unwrap();
    if period.subsec_nanos() % 2 == 1 {
        Ok(())
    } else {
        Err(SendError(value))
    }
}

pub struct SendError(String);

fn main() {
    let mut value = "imagine this is very long string".to_string();

    let success = 's: {
        // Try to send value two times.
        for _ in 0..2 {
            value = match send(value) {
                Ok(()) => break 's true,
                Err(SendError(value)) => value,
            }
        }
        false
    };

    println!("success: {success}");
}
```


## <span class="section-num">12</span> variable binding {#variable-binding}

Rust 使用 let 关键字声明变量，而可变性则是由变量能否在其生命周期中改变值来定义的。

默认情况下，Rust 中的变量是不可变的（immutable），这意味着一旦一个变量被赋值后，它的值就不能改变。这种特性有利于保证代码的安全性和避免数据竞争。如果需要可变性，可以选择使用 mut 关键字来声明变量。

rust 是强类型静态语言，每个变量都需要有明确的类型，但一般情况下不需要明确指定而是由编译器推导。rust
根据当前赋值或后续操作、赋值等情况，对变量的类型进行推导：

```rust
let var: type = expression;  // 指定变量值类型
let var = expression; // 由编译器根据 expresion 结果或者后续对 var 的使用方式进行推导。

fn main() {
    // Because of the annotation, the compiler knows that `elem` has type u8.
    let elem = 5u8;

    // Create an empty vector (a growable array).
    let mut vec = Vec::new();

    // At this point the compiler doesn't know the exact type of `vec`, it just knows that it's a
    // vector of something (`Vec<_>`).

    // Insert `elem` in the vector.
    vec.push(elem);

    // Aha! Now the compiler knows that `vec` is a vector of `u8`s (`Vec<u8>`)
    println!("{:?}", vec);
}
```

Rust 禁止使用未初始化的变量。变量必须被声明和初始化后才能使用。也可以先声明，后续再初始化（不建议）：

```rust
fn main() {
    // Declare a variable binding，但是未初始化（注意，即使指定 mut，也可以初始化一次）。
    let a_binding;

    {
        let x = 2;
        // Initialize the binding
        a_binding = x * x; // 变量被首次初始化，后续才可以开始使用。
    }
    println!("a binding: {}", a_binding);

    let another_binding;
    // Error! Use of uninitialized binding
    println!("another binding: {}", another_binding);

    another_binding = 1;
    println!("another binding: {}", another_binding);
}
```

Rust 中的每个变量默认 `都需要被使用` ，否则编译器会警告，可以在变量名前加 _ 来表明该变量可能不被使用：

```rust
fn main() {
    let an_integer = 1u32;
    let a_boolean = true;
    let unit = ();

    println!("A boolean: {:?}", a_boolean);
    println!("Meet the unit value: {:?}", unit);

    // The compiler warns about unused variable bindings; these warnings can
    // be silenced by prefixing the variable name with an underscore
    let _unused_variable = 3u32;
}
```

变量默认是不可变的，可以通过添加 mut 修饰符来修改：

```rust
fn main() {
    let _immutable_binding = 1;
    let mut mutable_binding = 1;

    // Ok
    println!("Before mutation: {}", mutable_binding);
    mutable_binding += 1;
    println!("After mutation: {}", mutable_binding);

    // Error! Cannot assign a new value to an immutable variable
    _immutable_binding += 1;
}
```

变量绑定是有一个 scope 的，默认是所在的 block：

-   rust 变量可以被 shadow，shadow 并不会 drop 前面变量的值，shadown 可以为同名变量指定不同的可变性和变量值类型。
-   如果 shadow 使用 `同名的变量名` ，则非 mut 变量可以将前面同名的 mut 变量 freezing，即不可修改。

<!--listend-->

```rust
fn main() {
    let x = 5;
    let x = x + 1; // 遮蔽第一个 x
    {
        let x = x * 2; // 第三个 x 遮蔽了第二个 x
        println!("The value of x in the inner scope is: {}", x);
    }
    println!("The value of x is: {}", x);
}

fn main() {
    let shadowed_binding = 1;

    {
        println!("before being shadowed: {}", shadowed_binding);
        // This binding *shadows* the outer one
        let shadowed_binding = "abc"; // 变量 shadow 前面（无论是否是同一个 block）的同名变量，类型也可以不同。
        println!("shadowed in inner block: {}", shadowed_binding);
    }
    println!("outside inner block: {}", shadowed_binding);

    // This binding *shadows* the previous binding
    let shadowed_binding = 2;
    println!("shadowed in outer block: {}", shadowed_binding);
}

// freezing
fn main() {
    let mut _mutable_integer = 7i32;
    {
        // Shadowing by immutable `_mutable_integer`
        let _mutable_integer = _mutable_integer;
        // Error! `_mutable_integer` is frozen in this scope
        _mutable_integer = 50;
        // `_mutable_integer` goes out of scope
    }
    // Ok! `_mutable_integer` is not frozen in this scope
    _mutable_integer = 3;
}
```


## <span class="section-num">13</span> refer/borrow {#refer-borrow}

Rust 变量不仅包含 stack 上的数据, 还 own resource, 如 Box&lt;T&gt; 拥有 T 在堆上的数据。

RAII： Rust 中每一个资源或对象只能有一个 Owner（如变量），在 Owner 离开作用域 scope 时，它的 Drop
trait 被调用来释放资源：可以避免资源泄露, 避免手动释放资源。也可以避免多次 free。

-   panic 时的默认行为 unwind 也会调用 Drop trait 来释放资源。

Rust 对象都有唯一的所有权，所有权可以通过赋值表达式、函数传参、函数返回、添加到 struct/tuple 和集合等来转移所有权, 原来的变量变成 `未初始化状态, 不能再使用`, 可以避免 dangling pointers。

-   如果要复制对象，需要实现 Copy/Clone trait。
-   一般情况下堆上分配的对象，例如 String/Vec 没有实现 Copy。自定义类型，如 struct/enum/union 也没有实现 Copy。
-   这种转移 Move 的方式，在性能上和安全性上都是非常有效的（避免了栈和堆内存拷贝），Rust 编译器也会对转移的变量进行错误检查。

<!--listend-->

```rust
// 所有权转移，所以可以从栈上返回对象：
fn new_person() -> Person {
    let person = Person {
        name : String::from("Hao Chen"),
        age : 44,
        sex : Sex::Male,
        email: String::from("haoel@hotmail.com"),
    };
    return person;
}
fn main() {
   let p  = new_person();
}

fn create_box() {
    // Allocate an integer on the heap
    let _box1 = Box::new(3i32);
    // `_box1` is destroyed here, and memory gets freed
}

fn main() {
    // Allocate an integer on the heap
    let _box2 = Box::new(5i32);

    // A nested scope:
    {
        // Allocate an integer on the heap
        let _box3 = Box::new(4i32);

        // `_box3` is destroyed here, and memory gets freed
    }

    // Creating lots of boxes just for fun
    // There's no need to manually free memory!
    for _ in 0u32..1_000 {
        create_box();
    }

    // `_box2` is destroyed here, and memory gets freed
}
```

为了不获得对象所有权的情况下来使用对象，Rust 通过借用操作（borrow/mut borrow）来获得对象的引用。

-   或者通过引用技术类型，如 Rc/Arc 来使用对象。

`Rust borrow checker` 对所有权和借用进行检查，违反时编译报错：

1.  对象可以多次共享借用，但是只能一次可变借用。，如果借用后续不再使用则允许再次可变借用（编译器特性
    NLL，[Non-Lexical Lifetime](https://practice.course.rs/lifetime/advance.html#nll-non-lexical-lifetime))
2.  可变借用的对象本身必须是可变的，即只能对 mut 对象进行 &amp;mut 可变借用，或从已有 &amp;mut 借用生成新的
    &amp;mut 借用；
3.  &amp;mut T 可以自动协变（coerced into）到 &amp;T 类型，所以在需要 &amp;T 的地方可以传入 &amp;mut T 类型值，但是反过来不行。
4.  对象在存在借用的情况下， `不能被修改或 move` ；《== `借用冻结`
5.  对象具有可变借用的情况下，原对象还是可以访问的（但不能修改和 move）；
6.  可变借用是排他的，在有可变借用的情况下，原对象不能访问和修改，只能通过可变借用来访问和修改。
7.  不支持通过借用（无论是可变还是共享借用）来实现对象的所有权转移，例如 `let v2 = *V` 。V 可以实现
    Copy trait，从而通过赋值的形式来克隆直接。

<!--listend-->

```rust
fn main() {
    let mut a = 123;
    let ar = &a;
    a = 456; // Error：cannot assign to `a` because it is borrowed
    println!("{ar}")
}

fn main() {
    let mut s = String::from_str("new string").unwrap();
    let sm = &mut s; // s 本身必须是 mut 类型才能被 &mut，在有 &mut 的情况下，原始值 s 不能在被访问。
    // println!("Result: {s} {sm}"); // cannot borrow `s` as immutable because it is also borrowed as mutable
    println!("Result: {sm}"); // OK

    let s2 = &mut String::from_str("new string").unwrap();
    // let sm2 = &mut s2; // s2 不是 mut 类型，不能被 &mut;
    s2.push_str(" abc"); // s2 虽然不是 mut 类型，但本身是 &mut，所以可以修改；
    println!("s2: {s2}");

    let s3 = s2; // s3 也是 &mut 类型, 也可以修改
    s3.push_str(" def");
    println!("s3: {s3}");

    // let s4 = *s3; // cannot move out of `*s3` which is behind a mutable reference
}

// 在已经 &mut T 的情况下，原始值不能再被访问（不能被 move 和修改）：
use std::str::FromStr;
fn main() {
    let mut s = String::from_str("new string").unwrap();
    let sm = &mut s; // s 本身必须是 mut 类型，才能被 &mut；

    s.push_str("abc"); // Error： 在 sm 后续继续使用的情况下，原来的 s 不能再使用。

    // 不能同时使用 s 和 sm。
    println!("Result: {s} {sm}"); // cannot borrow `s` as immutable because it is also borrowed as mutable
}


fn main() {
    let mut s = String::from_str("new string").unwrap();
    let ss = &s;
    let sm = &mut s; // sm 是可变借用，s 本身或类型必须是 mut 的。
    sm.push_str("abc");  // ss 不再使用，所以允许使用 sm ；

    let mut s = String::from("hello, ");
    let r1 = &mut s;
    r1.push_str("world");
    let r2 = &mut s; // r1 后续不再使用，所以运行再次 &mut s；
    r2.push_str("!");
    println!("{}", r2);
}

struct MyStruct(u8, String);
let mut ms = MyStruct(3, "test".to_string());
let msm = &mut ms;
let msm2 = &mut msm.1; // 可以从 &mut 创建出另一个 &mut
msm2.push_str(" def");
```

在 by-move 转移对象的所有权时可以改变它的可变性（毕竟转移到的变量 own 该对象）：

```rust
fn main() {
    let immutable_box = Box::new(5u32);

    println!("immutable_box contains {}", immutable_box);

    // Mutability error
    //*immutable_box = 4;

    // *Move* the box, changing the ownership (and mutability)
    let mut mutable_box = immutable_box;
    println!("mutable_box contains {}", mutable_box);
    // Modify the contents of the box
    *mutable_box = 4;
    println!("mutable_box now contains {}", mutable_box);
}
```

Temporarily move out of borrowed content：

```rust
self.current_buffer = std::mem::replace(&mut self.next_buffer, Buffer { buffer: buf });

/* ptr::read(src: *const T) -> T会从src指针处获取要复制的内容（假定是T类型实例），然后通过**”浅复制“**的方式，复制一份新实例，并返回。

ptr::read(src: *const Buffer)会从src指针copy Buffer结构体内容 到tmp(*mut Buffer)处；而Buffer内部buffer是String类型，非copy类型，此时tmp内的buffer与src内 */
unsafe {
    let result = ::std::ptr::read(dest);   //result中的buffer(String)，实际上跟dest中的buffer指向共同一块区域
    ::std::ptr::write(dest, src);
    result
}
```

不能通过 &amp;/&amp;mut 来转移 move 对象：

-   一般的解决办法是使用 std::mem::replace(&amp;dest, src) 将 src 值替换 dest，同时返回 dest 的值：

<!--listend-->

```rust
struct Buffer {
    buffer : String,
}
struct Render {
    current_buffer : Buffer,
    next_buffer : Buffer,
}
impl Render {
    fn update_buffer(& mut self, buf : String) {
        // error[E0507]: cannot move out of `self.next_buffer` which is behind a mutable reference
        // move occurs because `self.next_buffer` has type `Buffer`, which does not implement the `Copy` trait
        self.current_buffer = self.next_buffer;
        self.next_buffer = Buffer{ buffer: buf};
    }
}
fn main(){}

// OK 的例子，这里没有使用 &/&mut, 而是直接使用对象的变量 p 来转移 move 对象，这是 OK 的。
#[derive(Debug)]
struct Person {
    name: String,
    email: String,
}
fn main() {
    let  mut p = Person{name: "zzz".to_string(), email: "fff".to_string()};

    let _name = p.name; // 把结构体 Person::name Move掉
    println!("{} {}", _name, p.email); //其它成员可以正常访问

    println!("{:?}", p); //编译出错 "value borrowed here after partial move"

    p.name = "Hao Chen".to_string(); // Person::name又有了。
    println!("{:?}", p); //可以正常的编译了
}

// std::mem::replace
use std::mem::replace
fn update_buffer(& mut self, buf : String) {
  self.current_buffer = replace(&mut self.next_buffer, Buffer{buffer : buf});
}
```

Reborrow：如果以前的 &amp;mut 变量 r 不再使用，则可以使用 &amp;\*r 来获取新的 reborrow;

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn move_to(&mut self, x: i32, y: i32) {
        self.x = x;
        self.y = y;
    }
}

fn main() {
    let mut p = Point { x: 0, y: 0 };
    let r = &mut p;
    let rr: &Point = &*r; // reborrow
    // let p2 = *r // 错误：Point 为实现 copy，不能通过 *r 来进行转移值

    println!("{:?}", rr); // Reborrow ends here, NLL introduced
    // Reborrow is over, we can continue using `r` now
    r.move_to(10, 10);
    println!("{:?}", r);
}
```

`Rust borrow checker` 将变量视为所有权树的根，所以如果要修改对象的成员（如 struct field）、成员的子对象等状态，一般是使用 &amp;mut self，这样可以将 &amp;mut 引用从对象的根传递到 `最内层对象` 。这里的修改包括 3
方面：

1.  对对象本身进行修改；
2.  对对象的成员进行修改；
3.  调用对象或成员的方法，这些方法会改变对象的状态和内部字段等。

Rust 也提供了 `内部可变性` 机制，来让使用共享借用 &amp;self 的方法修改 Cell/RefCell/Mutex/Rwlock 等对象的内部状态。

```rust
let data = Arc::new(Mutex::new(0)); // data 不可变。
let mut data = data.lock().unwrap();  // MutextGuard 支持内部可变性，故可以获得 mut data。
*data += 1;
```

在对 struct/tuple 对象解构时, 可以 by-move（默认）或 by-refer（需要添加 ref/ref mut）：

-   ref/ref mut 表示获得对象的 refer，在表达式左侧使用 ref/ref mut 相当于表达式右侧的 &amp;/&amp;mut 操作，对应的变量是 &amp;/&amp;mut 类型。
-   by-move 可能造成 struct/tuple 的 field 被 `partial move` ，这些 field 后续不能再访问，但是未被
    partial move 的字段 `还是可以访问的` ；   <span class="org-target" id="org-target--partial-move"></span>
-   Vec/Array 等 `容器类型不支持 partial move`, 元素需要实现 Copy 或者被 std::mem::replace。

<!--listend-->

```rust
#[derive(Debug)]
struct Person {
    name: String,
    email: String,
}
fn main() {
    let  mut p = Person{name: "zzz".to_string(), email: "fff".to_string()};

    let _name = p.name; // 把结构体 Person::name Move掉
    println!("{} {}", _name, p.email); //其它成员可以正常访问

    println!("{:?}", p); //编译出错 "value borrowed here after partial move"

    p.name = "Hao Chen".to_string(); // Person::name又有了。
    println!("{:?}", p); //可以正常的编译了
}

// 另一个例子
#[derive(Clone, Copy)]
struct Point { x: i32, y: i32 }
fn main() {
    let c = 'Q';

    // A `ref` borrow on the left side of an assignment is equivalent to an `&` borrow on the right
    // side.
    let ref ref_c1 = c;
    let ref_c2 = &c;
    println!("ref_c1 equals ref_c2: {}", *ref_c1 == *ref_c2);


    let point = Point { x: 0, y: 0 };
    let _copy_of_x = {
        let Point { x: ref ref_to_x, y: _ } = point; // ref_to_x 是 point.x 的引用
        *ref_to_x
    };
    let mut mutable_point = point;
    {
        let Point { x: _, y: ref mut mut_ref_to_y } = mutable_point;
        *mut_ref_to_y = 1;
    }
    println!("point is ({}, {})", point.x, point.y);
    println!("mutable_point is ({}, {})", mutable_point.x, mutable_point.y);

    let mut mutable_tuple = (Box::new(5u32), 3u32);
    {
        let (_, ref mut last) = mutable_tuple;
        *last = 2u32;
    }
    println!("tuple is {:?}", mutable_tuple);
}

// https://practice.course.rs/ownership/ownership.html
fn main() {
    #[derive(Debug)]
    struct Person {
        name: String,
        age: Box<u8>,
    }

    let person = Person {
        name: String::from("Alice"),
        age: Box::new(20),
    };

    // `name` is moved out of person, but `age` is referenced
    let Person { name, ref age } = person;
    println!("The person's age is {}", age);
    println!("The person's name is {}", name);

    // Error! borrow of partially moved value: `person` partial move occurs
    //println!("The person struct is {:?}", person);

    // `person` cannot be used but `person.age` can be used as it is not moved
    println!("The person's age from person struct is {}", person.age);
}
```

变量声明时 mut 的位置差异：

```text
let mut data1: Vec<i32> = vec![1, 2]; vs let data2: &mut Vec<i32> = &mut vec![1, 2];
```

1.  data1 作为一个 mut 变量, 值是可变的, 支持 data1.push(2), 也支持 &amp;mut data1;
2.  data2 不是 mut 变量, 所以不能修改 data2 本身, 但是 &amp;mut 类型, 所以借用的值是可变的, 支持
    data2.push(2), 但是不支持 &amp;mut data2;
3.  不能通过解引用 \`&amp;mut Vec&lt;i32&gt;\` 来获得 \`Vec&lt;i32&gt;\` 的所有权。Rust 的所有权规则决定了 `不能在未解除引用的情况下直接将引用的值赋给另一个变量` , 通常的解法:
    1.  使用 clone() 创建一个新对象;
    2.  使用 std::mem::take(&amp;mut T) 来返回 T 值, 并将原来 T 指向的内容设置为 T 的 Default 值;

<!--listend-->

```rust
fn main() {
    let (a, b) = &(1, 2); // a 和 b 类型是 &i32
    println!("Results: {a} {b}");

    let &(c, d) = &(1, 2); // c 和 d 类型是 i32
    println!("Results: {c} {d}");

    let mut data1: Vec<i32> = vec![1, 2];
    let data2: &mut Vec<i32> = &mut vec![1, 2];
    data1.push(3); // OK
    data2.push(3); // OK
    println!("data1: {:?}, data2: {:?}", data1, data2);

    // data1 = data2; // Error: expected `Vec<i32>`, found `&mut Vec<i32>`
    // data1 = data2.to_vec(); // OK

    // 编译错误，因为你不能通过解引用 `&mut Vec<i32>` 来获得 `Vec<i32>` 的所有权。Rust 的所有权规则决定了你不能在未解除引用的情况下直接将引用的值赋给另一个变量。
    // data1 = *data2; // error[E0507]: cannot move out of `*data2` which is behind a mutable reference

    data1 = data2.clone(); // OK, Vec<i32> 实现了 Clone
    data1.push(4); // OK

    // data2 = data1; // Error: expected `&mut Vec<i32>`, found `Vec<i32>`
    // data2 = &mut data1; // Error: cannot assign twice to immutable variable
}
```

解引用操作符 \* 用于返回引用类型对象的值。由于 Rust 编译器会自动解引用和生成引用，所以实际很少直接通过 \* 操作符来显式解引用：

1.  通过 . 操作符访问对象的成员时，Rust 自动解引用： `ref.filed 等效于 (*ref).field` ;
2.  通过 . 操作符调用方法时，如果方法的第一个参数式 &amp;self 或 &amp;mut self, 则 `自动借用对象来生成引用` ，然后传递给对应的方法：v.sort() 等效为 (&amp;v).sort()，称为 methdo call deref coercion.
    -   一般执行对象的方法调用后，对象本身还是可访问的，所以方法一般都以 &amp;self 或 &amp;mut self 作为第一个参数。
    -   如果对象本身已经是引用，则调用 &amp;self 或 &amp;mut self 方法时，直接传递对象的引用即可。

<!--listend-->

```rust
  // 使用 . 访问引用对象成员时，自动解引用
  struct Anime { name: &'static str, bechdel_pass: bool };
  let aria = Anime { name: "Aria: The Animation", bechdel_pass: true };
  let anime_ref = &aria;
  assert_eq!(anime_ref.name, "Aria: The Animation");
  // Equivalent to the above, but with the dereference written out:
  assert_eq!((*anime_ref).name, "Aria: The Animation");

  // 对象方法是 &self 或 &mut self 时自动生成对象的引用
  let mut v = vec![1973, 1968];
  v.sort(); // implicitly borrows a mutable reference to v
  (&mut v).sort(); // equivalent, but more verbose

  // 但是直接使用 ref 变量时，需要手动解引用
  let x = 5;
  let y = &x;
  assert_eq!(5, *y); // y 需要解引用
```

其他自动解引用场景(都支持 `多级自动解引用` )：

1.  比较操作，所以默认情况下比较的是引用的值，而非引用本身（指针）。
2.  index 操作符。
3.  算术运算操作；
4.  println!()/assert\* 等宏函数自动解引用传入的引用参数：

<!--listend-->

```rust
struct Point { x: i32, y: i32 }
let point = Point { x: 1000, y: 729 }; let r: &Point = &point;
let rr: &&Point = &r;
let rrr: &&&Point = &rr;
assert_eq!(rrr.y, 729); // . 操作支持多级解引用

let x = 10; let y = 10;
let rx = &x; let ry = &y;
let rrx = &rx; let rry = &ry;
assert!(rrx <= rry);  // 比较操作也自动多级解引用
assert!(rrx == rry);

fn factorial(n: usize) -> usize {
    (1..n+1).product()
}
let r = &factorial(6);
assert_eq!(r + &1009, 1729); // 算术运算自动解引用。
```

如果要比较引用地址本身，需要使用 std::ptr::eq 函数，使用 {:p} 来打印指针地址：

```rust
use std::ptr;

let five = 5;
let other_five = 5;
let five_ref = &five;
let same_five_ref = &five;
let other_five_ref = &other_five;

assert!(five_ref == same_five_ref); // 比较操作时，自动多级解引用，所以比较的是值。
assert!(five_ref == other_five_ref);

assert!(ptr::eq(five_ref, same_five_ref)); // 比较地址
assert!(!ptr::eq(five_ref, other_five_ref));
```

Rust 引用操作可以是任意表达式，如字面量，rust 会自动进行转换（coercion）

-   [Type coercions - The Rust Reference](https://doc.rust-lang.org/stable/reference/type-coercions.html#coercion-sites)

<!--listend-->

```rust
r + &1009

let _: &i8 = &mut 42;

fn bar(_: &i8) { }
fn main() {
    bar(&mut 42);

    let x = 5;
    let y = &x;
    assert_eq!(5, y);
    println!("Success!");
}
```

Index 操作符返回 `元素本身` 而非它的引用， `a[i] 等效为 *a.index(i)` : a[i] 使用 Index trait，虽然该
trait 的 index 方法返回引用，但是 rust `会自动解引用` 。

println!() 宏函数自动解引用传入的引用参数；

```rust
// v should have at least one element.
fn smallest(v: &[i32]) -> &i32 {
    let mut s = &v[0];  // v 虽然是引用类型，v[0] 返回值类型是 i32
    for r in &v[1..] { // &v[1..] 返回切片引用，对引用进行迭代，结果 r 还是引用，所以需要 *r 来获得 r 的值。
        if *r < *s { s = r; }
    }
    s
}

fn show(table: &Table) {
    for (artist, works) in table { // 迭代引用时，结果元素 artist 和 works 都是引用类型
        println!("works by {}:", artist); // 宏函数自动解引用
        for work in works { // 迭代引用
            println!("  {}", work); // work 还是引用，宏函数自动解引用
        }
    }
}

let x = 5;
let y = &x;
assert_eq!(5, x);
assert_eq!(5, *y); // OK
assert_eq!(5, y);  // 错误，can't compare `{integer}` with `&{integer}`
```

对于 T，&amp;T，&amp;mut T，Box&lt;T&gt; 在进行 Display 时显示的都是 T 或引用的 T 的值。但是 &amp;T, &amp;mut T, Box&lt;T&gt; 实际是指针类型，可以使用 p 修饰符来 `显示它们的地址而非值` ：

-   Rc::clone() 由于不会发生内存拷贝，而只是增加了引用计数，所以产生的对象与以前的对象是 `相同的地址` 。

<!--listend-->

```rust
use std::rc::Rc;

fn main() {
    let mut t = 123;
    let tp = &mut t;
    let tpp = &tp; //  从 &mut T 变量中可以再借用出共享引用 &T
    println!("{:p} {:p}",  tp, tpp); // tp 和 tpp 是两个不同类型的变量，所以地址不一致

    let rc = Rc::new(String::from("abc"));
    let rc2 = rc.clone();

    // rc: abc, rc2: abc, rc pointer:0x600001bdc2b0, rc2 pointer 0x600001bdc2b0
    // 可见 rc 和 rc2 内存的地址都是一样的，说明 Rc clone 没有发生堆内存拷贝。
    println!("rc: {}, rc2: {}, rc pointer:{:p}, rc2 pointer {:p}", rc, rc2, rc, rc2);
}
```


## <span class="section-num">14</span> lifetime {#lifetime}

Rust 给每一个 `引用类型` 对象设置一个 lifetime（自动或手动），如函数的输入和输出参数，函数内的变量，全局变量，struct/enum 成员等。

设置 lifetime 的目的是指导 Rust borrow checker 对程序各部分借用的对象的引用的生命周期进行检查，发现异常时编译报错。lifetime 只是一个编译时的注解， `没有运行时代表` ，也不能在表达式中使用：

```rust
let b = &'a dyn MyTrait + Send + 'static; // error: expected expression, found keyword `dyn`
let b = &'a(dyn MyTrait + Send + 'static); // error: borrow expressions cannot be annotated with lifetimes
```

lifetime 表达的是一个 `相对的概念` 和约束， `Rust borrow checker` 根据 lifetime anno 来检查引用是否有效：

1.  &lt;T: 'b&gt; ：表示 T 的引用的生命周期比 'b 长。
    -   &amp;'b T 隐式表示 T: 'b, 即 T 的生命周期要比 'b 长。
2.  &lt;T: Trait + 'b&gt; ：表示 T 要实现 Trait 且 T 的生命周期比 'b 长。
3.  &lt;'a: 'b, 'b&gt;： 表示 'a 的生命周期比 'b 长；
    -   注意上面 'a 和 'b 的顺序和语法，错误的情况：&lt;'a, 'b, 'a: 'b&gt;;
4.  struct foo&lt;'a: 'b, 'b，T: 'b&gt; (val1: &amp;'a String, val2: &amp;'a String, val3: &amp;'b String, val4: &amp;T):
    -   val1 和 val2 的生命周期一样长, 且比 val3 的生命周期长；
    -   val4 的生命周期要比 'b 长，即 val4 的生命周期要比 val3 长；
    -   foo 对象的生命周期不能长于 'a 和 'b;
5.  fn print_refs&lt;'a: 'b, 'b&gt;(x: &amp;'a i32, y: &amp;'b i32) -&gt; &amp;'b String
    -   函数执行期间 'a, 'b 的引用要一直有效，即 'a 和 'b 的生命周期比函数长；
    -   'a: 'b 表示 'a 的生命周期比 'b 长，所以 x 的生命周期要比 y 长；
    -   返回值的生命周期要和 y 一样长；

lifetime 作为泛型参数时，必须位于其他泛型参数之前，比如 &lt;'a, T, T2&gt;：

```rust
// lifetime 泛型参数要在类型参数前，'lifetime 要紧贴 &;
fn add_ref<'a, 'b, T>(a: &'a mut T, b: &'b T) -> &'a T // 注意 'a 在 mut 前。
  where
      T: std::ops::Add<T, Output = T> + Copy,
  {
      *a = *a + *b;
      a // OK
      // b // Error: b 的声明周期是 'b 与返回值的声明 'a 不一致。function was supposed to return data with lifetime `'b` but it is returning data with lifetime `'a`
  }

#[derive(Debug)]
struct Ref<'a, T: 'a>(&'a T); // 等效于 struct Ref<'a, T: 'a>(&T);

// `Ref` contains a reference to a generic type `T` that has an unknown lifetime `'a`. `T` is
// bounded such that any *references* in `T` must outlive `'a`. Additionally, the lifetime of `Ref`
// may not exceed `'a`.

// Here a reference to `T` is taken where `T` implements `Debug` and all *references* in `T` outlive
// `'a`. In addition, `'a` must outlive the function.
fn print_ref<'a, T>(t: &'a T) where T: Debug + 'a { // 等效于 fn print_ref<'a, T>(t: &T) where T: Debug + 'a {
    println!("`print_ref`: t is {:?}", t);
}
fn main() {
    let x = 7;
    let ref_x = Ref(&x);
    print_ref(&ref_x);
    print(ref_x);
}
```

如果泛型类型需要 lifetime 参数，但是在实现某个 Trait 时该 Trait 的方法并不需要该 lifetime 参数，则可以使用 &lt;'_&gt;:

```rust
impl<'a> Reader for BufReader<'a> {
    // 'a is not used in the following methods
}
// can be written as :
impl Reader for BufReader<'_> {
}
```

struct/enum 的 lifetime：

-   如果 struct/enum 有 ref 成员，则必须要为 struct/enum 指定 lifetime 参数；
-   struct/enum 对象的 lifetime 要比指定的所有 lifetime 参数 `短` .
-   特殊的 struct MyStruct&lt;'static&gt; 则 MyStruct 对象的 lifetime 可以任意长（因为 'static 是在程序整个运行期间都有效）。

<!--listend-->

```rust
// A type `Borrowed` which houses a reference to an `i32`. The reference to `i32` must outlive
// `Borrowed`.
#[derive(Debug)]
struct Borrowed<'a>(&'a i32);

// Similarly, both references here must outlive this structure.
#[derive(Debug)]
struct NamedBorrowed<'a> {
    x: &'a i32,
    y: &'a i32,
}

// An enum which is either an `i32` or a reference to one.
#[derive(Debug)]
enum Either<'a> {
    Num(i32),
    Ref(&'a i32),
}

fn main() {
    let x = 18;
    let y = 15;

    let single = Borrowed(&x);
    let double = NamedBorrowed { x: &x, y: &y };
    let reference = Either::Ref(&x);
    let number    = Either::Num(y);

    println!("x is borrowed in {:?}", single);
    println!("x and y are borrowed in {:?}", double);
    println!("x is borrowed in {:?}", reference);
    println!("y is *not* borrowed in {:?}", number);
}
```

函数 lifetime:

1.  所有 ref 必须有 lifetime anno:
    -   如果没有明确指定，Rust 编译器自动加 lifetime anno，规则参考: [14.3](#org-target--lifetime-elision-rules)
2.  所有返回值的 ref 的 lifetime 必须和某些输入的值的 lifetime 一致或者是 'static;
3.  如果自动推断后，还是不能确定返回值 ref 和输入值 lifetime 的关系，则编译报错，需要手动加 lifetime：

<!--listend-->

```rust
// `print_refs` takes two references to `i32` which have different lifetimes `'a` and `'b`. These
// two lifetimes must both be at least as long as the function `print_refs`.
fn print_refs<'a, 'b>(x: &'a i32, y: &'b i32) {
    println!("x is {} and y is {}", x, y);
}

// A function which takes no arguments, but has a lifetime parameter `'a`.
fn failed_borrow<'a>() {
    let _x = 12;

    // ERROR: `_x` does not live long enough
    let _y: &'a i32 = &_x;

    // Attempting to use the lifetime `'a` as an explicit type annotation inside the function will
    // fail because the lifetime of `&_x` is shorter than that of `_y`. A short lifetime cannot be
    // coerced into a longer one.
}

fn main() {
    // Create variables to be borrowed below.
    let (four, nine) = (4, 9);

    // Borrows (`&`) of both variables are passed into the function.
    print_refs(&four, &nine);
    // Any input which is borrowed must outlive the borrower.  In other words, the lifetime of
    // `four` and `nine` must be longer than that of `print_refs`.

    failed_borrow();
    // `failed_borrow` contains no references to force `'a` to be longer than the lifetime of the
    // function, but `'a` is longer.  Because the lifetime is never constrained, it defaults to
    // `'static`.
}

// One input reference with lifetime `'a` which must live at least as long as the function.
fn print_one<'a>(x: &'a i32) {
    println!("`print_one`: x is {}", x);
}

// Mutable references are possible with lifetimes as well.
fn add_one<'a>(x: &'a mut i32) {
    *x += 1;
}

// Multiple elements with different lifetimes. In this case, it would be fine for both to have the
// same lifetime `'a`, but in more complex cases, different lifetimes may be required.
fn print_multi<'a, 'b>(x: &'a i32, y: &'b i32) {
    println!("`print_multi`: x is {}, y is {}", x, y);
}

// Returning references that have been passed in is acceptable.  However, the correct lifetime must
// be returned.
fn pass_x<'a, 'b>(x: &'a i32, _: &'b i32) -> &'a i32 { x }

// 要求返回值的 lifetime 和 'a 一样长, 'a 等效于 'static, 而函数内的 ref 在函数返回即失效,所以不能编译.
//fn invalid_output<'a>() -> &'a String { &String::from("foo") }

// The above is invalid: `'a` must live longer than the function.
// Here, `&String::from("foo")` would create a `String`, followed by a
// reference. Then the data is dropped upon exiting the scope, leaving
// a reference to invalid data to be returned.

fn main() {
    let x = 7;
    let y = 9;

    print_one(&x);
    print_multi(&x, &y);

    let z = pass_x(&x, &y);
    print_one(z);

    let mut t = 3;
    add_one(&mut t);
    print_one(&t);
}

// 错误的情况，编译器不能推断出返回值引用的 lifetime 关系。
fn order_string(s1 : &str, s2 : &str) -> (&str, &str) {
    if s1.len() < s2.len() {
        return (s1, s2);
    }
    return (s2, s1);
}

```

闭包 lifetime：闭包函数返回引用时可能会遇到 lifetime 问题（[14.3](#org-target--lifetime-elision-rules) 并不适合闭包）：

```rust
fn fn_elision(x: &i32) -> &i32 { x } // OK

let closure_elision = |x: &i32| -> &i32 { x }; // Error
|     let closure = |x: &i32| -> &i32 { x }; // fails
|                       -        -      ^ returning this value requires that `'1` must outlive `'2`
|                       |        |
|                       |        let's call the lifetime of this reference `'2`
|                       let's call the lifetime of this reference `'1`



```

解决办法：

1.  使用 nightly toolchain 和开启 #\![feature(closure_lifetime_binder)]，这样可以为闭包函数指定 for
    &lt;'a&gt; 语法的 lifetime：<https://github.com/rust-lang/rust/issues/97362>
2.  或者，定义一个 helper 函数，该函数可以指定闭包输入、输出参数所需的 lifetime，然后内部调用闭包；
3.  或者，将闭包转换为 fn 函数指针，函数指针支持使用 for&lt;'a&gt; 来定义高阶函数；<https://stackoverflow.com/a/60906558>

<!--listend-->

```rust
// 解决办法1:
fn main() {
    // let clouse_test = |input: &String| input; // error: lifetime may not live long enough
    //let clouse_test = |input: &String| -> &String {input}; // error: lifetime may not live long enough
    // let clouse_test = |input: &'a String| ->&'a String {input}; // error[E0261]: use of undeclared lifetime name `'a`

    let clouse_test = for <'a> |input: &'a String| ->&'a String {input}; // 需要使用 nightly toolchain 和开启 #![feature(closure_lifetime_binder)]
    println!("Results:")
}

// 解决办法2:
fn testStr<'a> (input: &'a String) -> &'a String {
    let closure_test = |input: &'a String | -> &'a String {input}; // 闭包使用外围 helper 函数定义的 lifetime
    return closure_test(input);
}

fn main() {
    // let clouse_test = |input: &String| input; // error: lifetime may not live long enough

    //let clouse_test = |input: &String| -> &String {input}; // error: lifetime may not live long enough

    // let clouse_test = |input: &'a String| ->&'a String {input}; // error[E0261]: use of undeclared lifetime name `'a`

    // let clouse_test = for <'a> |input: &'a String| ->&'a String {input};

    println!("Results:{}", testStr(&"asdfab".to_string()));
}

// 解决办法3:
// 将闭包转换为 fn 函数指针，函数指针支持使用 for<'a> 来定义高阶函数，而且编译期间大小是已知的。
// 但是不能使用 Fn/FnMut/FnOnce 等 trait 类型。
let test_fn: for<'a> fn(&'a _) -> &'a _ = |p: &String| p;
println!("Results:{}", test_fn(&"asdfab".to_string()));

// 其他例子：https://github.com/rust-lang/rust/pull/56746/files
#![allow(unused)]

fn willy_no_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x| { p }; // no type annotation at all
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn willy_ret_type_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x| -> &str { p }; // type annotation on the return type
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn willy_ret_region_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x| -> &'w str { p }; // type+region annotation on return type
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn willy_arg_type_ret_type_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x: &str| -> &str { p }; // type annotation on arg and return types
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn willy_arg_type_ret_region_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x: &str| -> &'w str { p }; // fully annotated
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn main() {
    let world = format!("World");
    let w1: &str = {
        let hello = format!("He11o");
        willy_no_annot(&world, &hello)
    };
    let w2: &str = {
        let hello = format!("He22o");
        willy_ret_type_annot(&world, &hello)
    };
    let w3: &str = {
        let hello = format!("He33o");
        willy_ret_region_annot(&world, &hello)
    };
    let w4: &str = {
        let hello = format!("He44o");
        willy_arg_type_ret_type_annot(&world, &hello)
    };
    let w5: &str = {
        let hello = format!("He55o");
        willy_arg_type_ret_region_annot(&world, &hello)
    };
    assert_eq!((w1, w2, w3, w4, w5),
        ("World","World","World","World","World"));
}
}
```

更长的 lifetime 可以被 coerced 到短一些的 lifetime：

-   'a: 'b 表示 'a lifetime 至少要比 'b 长，这样在返回 'b 的引用时，可以返回 'a 的 lifetime 对象；
-   类似的 T: 'a 表示，T 的 ref 的 lifetime 至少要比 'a 长；

<!--listend-->

```rust
// Here, Rust infers a lifetime that is as short as possible.  The two references are then coerced
// to that lifetime.
fn multiply<'a>(first: &'a i32, second: &'a i32) -> i32 {
    first * second
}

// `<'a: 'b, 'b>` reads as lifetime `'a` is at least as long as `'b`.  Here, we take in an `&'a i32`
// and return a `&'b i32` as a result of coercion.
fn choose_first<'a: 'b, 'b>(first: &'a i32, _: &'b i32) -> &'b i32 {
    first
}

fn main() {
    let first = 2; // Longer lifetime
    {
        let second = 3; // Shorter lifetime

        println!("The product is {}", multiply(&first, &second));
        println!("{} is the first", choose_first(&first, &second));
    };
}

// 另一个例子
trait MyTrait<'a> {
    fn say_hello(&'a self) -> &'a String;
}

struct MyStruct(String);

impl<'a> MyTrait<'a> for MyStruct {
    fn say_hello(&'a self) -> &'a String {
        println!("hello {}", self.0);
        &self.0
    }
}

fn printf_hello<'a, 'b>(say_hello: Option<&'a (dyn MyTrait<'a> + Send + 'b)>) -> Option<&'b String>
where
    'a: 'b,
{
    let hello = if let Some(my_trait) = say_hello {
        my_trait.say_hello()
    } else {
        return None;
    };
    Some(hello)
}
```


### <span class="section-num">14.1</span> Higher-Rank Trait Bounds (HRTBs) {#higher-rank-trait-bounds--hrtbs}

HRTB 一般只会在 Fn 作为 Bound 时会使用到, 下面没有加 lifetime 的代码是可以正常编译的：

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
      where F: Fn(&(u8, u16)) -> &u8,
  {
      fn call(&self) -> &u8 {
          (self.func)(&self.data)
      }
  }

fn do_it(data: &(u8, u16)) -> &u8 { &data.0 }

fn main() {
    let clo = Closure { data: (0, 1), func: do_it };
    println!("{}", clo.call());
}
```

如果要给上面的代码添加 lifetime bound 则会遇到 F 的 lifetime 该如何指定的问题：

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F> // where F: Fn(&'??? (u8, u16)) -> &'??? u8,
        {
            fn call<'a>(&'a self) -> &'a u8 {
                (self.func)(&self.data)
            }
        }

fn do_it<'b>(data: &'b (u8, u16)) -> &'b u8 { &'b data.0 }

fn main() {
    'x: {
        let clo = Closure { data: (0, 1), func: do_it };
        println!("{}", clo.call());
    }
}


// Error1:
impl<'a, F> Closure<F>
             where F: Fn(&'a (u8, u16)) -> &'a u8,
        {
            fn call<'a>(&'a self) -> &'a u8 { // lifetime name `'a` shadows a lifetime name that is
                // already in scope
                (self.func)(&self.data)
            }
        }

// Error2:
impl<'a, F> Closure<F>
             where F: Fn(&'a (u8, u16)) -> &'a u8,
        {
            fn call<'b>(&'b self) -> &'b u8 { //  method was supposed to return data with lifetime `'a`
                //  but it is returning data with lifetime `'b`
                (self.func)(&self.data)
            }
        }

// Error3:
impl<'a, F> Closure<F>
          where F: Fn(&'a (u8, u16)) -> &'a  u8,
      {
          fn call(& self) -> & u8 { // rustc 自动为 &self 添加 liefitime 如 '1: method was supposed to
              // return data with lifetime `'1` but it is returning data with
              // lifetime `'a`
              (self.func)(&self.data)
          }
      }

// Error4: 可以编译过，但是要求 Closure 的 liefitime 和传入的 Fn 的参数 lifetime 一致，不符合预期语
// 义（Fn 的函数有自己独立的 lifetime，和 Closure 对象 lifetime 无关）。
impl<'a, F> Closure<F>
        where F: Fn(&'a (u8, u16)) -> &'a  u8,
    {
        fn call(&'a self) -> &'a u8 {
            (self.func)(&self.data)
        }
    }

// OK：HRTB
impl<F> Closure<F> // 1. 泛型参数中没有 'a lifetitme
      where F: for <'a> Fn(&'a (u8, u16)) -> &'a u8, // 2. 在 F 的 Bound 中使用 for <'a> 来声明 'a lifetime
  {
      fn call<'a>(&'a self) -> &'a u8 { // 'a 和上面的 for <'a> 没有任何关系，是 call() 方法自己的
          // lifetime。由于 rustc 会自动加 lifetime，所以不指定：fn
          // call(&self) -> &u8
          (self.func)(&self.data)
      }
  }

```

可见 HRTB 一般 `只在 Fn Bound 中使用，for <'a> Fn 表示 Fn 满足任意 'a liftime` ，所以Fn 也满足 call()
方法的 lifetime 要求，可以在 call() 方法中使用。

HRTB 有两种等效语法：

```rust
where F: for<'a> Fn(&'a (u8, u16)) -> &'a u8,
// 等效为
where for <'a> F: Fn(&'a (u8, u16)) -> &'a u8,
```


### <span class="section-num">14.2</span> 'static {#static}

'static 是 Rust 内置的特殊 lifetime anno，在 &amp;str 和函数泛型参数的 Bound 中广泛使用。&amp;'static 表示借用的对象的声明周期和程序的执行时间一样长，也就是在程序运行期间一直存在的对象。'static 值在程序整个运行期间有效 `指的是在 main 函数返回还有效` ，例如全局的 const 变量，全局 static 变量，字符串字面量等，它们都保存在程序二进制的 read-only 部分。

```rust
// A reference with 'static lifetime:
let s: &'static str = "hello world";

// 'static as part of a trait bound:
fn generic<T>(x: T) where T: 'static {}

fn main() {
    let v: &'static string = "hello";
    need_static(v);
    println!("Success!")
}
fn need_static(r : &'static str) {
    assert_eq!(r, "hello");
}
```

'static lifetime 可以被 coerced 到一个更短的生命周期：

```rust
// Make a constant with `'static` lifetime.
static NUM: i32 = 18;

// Returns a reference to `NUM` where its `'static` lifetime is coerced to that of the input
// argument.
fn coerce_static<'a>(_: &'a i32) -> &'a i32 {
    &NUM
}

fn main() {
    {
        // Make an integer to use for `coerce_static`:
        let lifetime_num = 9;

        // Coerce `NUM` to lifetime of `lifetime_num`:
        let coerced_static = coerce_static(&lifetime_num);

        println!("coerced_static: {}", coerced_static);
    }

    println!("NUM: {} stays accessible!", NUM);
}
```

'static 也可以作为函数泛型参数的 Bound 约束，表示不能包含 non-static refer。

注意：当 receiver hold value 直到 drop 它才失效时，相当于声明了 'static lifetime bound。但是
ref to owned data 不满足该 'static 要求。

```rust
use std::fmt::Debug;

// 函数 hold T 值，所以 T 的 Bound 会隐式的自动会加 'static 并自动满足。
fn print_it<T: Debug + 'static>(input: T) {
    println!("'static value passed in is: {:?}", input);
}

// 函数 hold input 值，所以 input 的 Bound 会隐式的自动会加 'static 并自动满足。
fn print_it1(input: impl Debug + 'static) {
    println!("'static value passed in is: {:?}", input);
}

// input 是 ref，借用的 T 值必须是 'static 即延续到整个程序的生命周期。
fn print_it2<T: Debug + 'static>(input: &T) {
    println!("'static value passed in is: {:?}", input);
}

fn main() {
    // i is owned and contains no references, thus it's 'static:
    let i = 5;
    print_it(i);

    // oops, &i only has the lifetime defined by the scope of main(), so it's not 'static:
    print_it(&i); //  `i` does not live long enough
    print_it1(&i); //  `i` does not live long enough

    // but this one WORKS !
    print_it2(&i);
}
```

Box&lt;dyn Trait&gt; 等效于 Box&lt;dyn Trait + 'static&gt;， &amp;'a Box &lt;dyn Trait&gt; 等效于 &amp;'a Box&lt;dyn Trait +
'static&gt;, 参考：
<https://doc.rust-lang.org/reference/lifetime-elision.html#default-trait-object-lifetimes>


### <span class="section-num">14.3</span> lifetime elision {#lifetime-elision}

Rust borrow checker 使用 lifetime annotation 来检查所有 borrow，确保所有的 borrow 都是有效的。一般情况下，一个变量的 lifetime 开始于它创建，结束于它被销毁。

在大部分情况下，由于有一些 elision rule，用户不需要显式指定 borrow 变量的 lifetime annotation。

-   非引用类型的参数，由于是 Copy 或 Move 对应 ownership，故不需要 lifetime 定义。

<!--listend-->

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

Rust 编译器 <span class="org-target" id="org-target--lifetime-elision-rules"></span>：

1.  The first rule 针对函数的输入参数：is that the compiler `assigns a lifetime parameter to each
       parameter that’s a reference`. In other words, a function with one parameter gets one lifetime
    parameter: fn foo&lt;'a&gt;(x: &amp;'a i32); a function with two parameters gets two separate lifetime
    parameters: fn foo&lt;'a, 'b&gt;(x: &amp;'a i32, y: &amp;'b i32); and so on.

<!--listend-->

```rust
  struct S<'a, 'b> {
      x: &'a i32,
      y: &'b i32
  }
  fn sum_r_xy(r: &i32, s: S) -> i32 {
      r + s.x + s.y
  }
  // 函数签名等效为：
  fn sum_r_xy<'a, 'b, 'c>(r: &'a i32, s: S<'b, 'c>) -> i32
```

1.  The second rule 针对函数的输出参数：is that, if there is `exactly one` input lifetime parameter,
    that lifetime is assigned to `all output` lifetime parameters: fn foo&lt;'a&gt;(x: &amp;'a i32) -&gt; &amp;'a i32.

<!--listend-->

```rust
  fn first_third(point: &[i32; 3]) -> (&i32, &i32) {
      (&point[0], &point[2])
  }
  // 等效为
  fn first_third<'a>(point: &'a [i32; 3]) -> (&'a i32, &'a i32)
```

1.  The third rule 针对方法：is that, if there are multiple input lifetime parameters, but one of
    them is `&self or &mut self` because this is a method, the lifetime of self is assigned to `all
       output lifetime parameters`. This third rule makes methods much nicer to read and write because
    fewer symbols are necessary. 所以，对于方法函数，一般不需要指定输入&amp;输出参数的声明周期。

<!--listend-->

```rust
  struct StringTable {
      elements: Vec<String>,
  }

  impl StringTable {
      fn find_by_prefix(&self, prefix: &str) -> Option<&String> {
          for i in 0 .. self.elements.len() {
              if self.elements[i].starts_with(prefix) {
                  return Some(&self.elements[i]); // [i] 返回对象本身，这里需要通过 & 获得它的引用
              }
          }
          None
      }
  }

  // 等效为
  fn find_by_prefix<'a, 'b>(&'a self, prefix: &'b str) -> Option<&'a String>
```

如果经过上面 elision rule，还有引用参数的 lifetime 不明确， `Rust 拒绝编译` ：

```rust
fn first_word(s: &str) -> &str { // 正确
// 编译器等效为
fn first_word<'a>(s: &'a str) -> &'a str {

fn longest(x: &str, y: &str) -> &str { // 错误
// 经过  rule 后：
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str { // 输出引用 lifetime 不明确，报错
```

其他函数参数或结果中可以消除 lifetime 的情况：

```rust
fn requires_t_outlives_a<'a, T>(x: &'a T) {} // 隐式：T: 'a

fn requires_t_outlives_a_not_implied<'a, T: 'a>() {}

fn requires_t_outlives_a<'a, T>(x: &'a T) {
    // This compiles, because `T: 'a` is implied by the reference type `&'a T`.
    requires_t_outlives_a_not_implied::<'a, T>();
}
fn not_implied<'a, T>() {
    // This errors, because `T: 'a` is not implied by the function signature.
    requires_t_outlives_a_not_implied::<'a, T>();
}

// 只有 lifetime 会被隐式 bound，trait 还是需要显式指定的。
use std::fmt::Debug;
struct IsDebug<T: Debug>(T);
// error[E0277]: `T` doesn't implement `Debug`
fn doesnt_specify_t_debug<T>(x: IsDebug<T>) {}
```

对于 trait object 有特殊的 lifetime bound。参考：[18.5](#org-target--trait-object)

1.  Box&lt;dyn Trait&gt; 默认等效于 `Box<dyn Trait + 'static>` ;
2.  &amp;'x Box&lt;dyn Trait&gt; 等效于 &amp;'x Box&lt;dyn Trait + 'static&gt;;
    -   'x 可能是编译器自动加的, 所以即使没有明确指定, &amp;Box&lt;dyn Trait&gt; 等效于 &amp;Box&lt;dyn Trait+'static&gt;;
3.  &amp;'r Ref&lt;'q, dyn Trait&gt; 等效于 &amp;'r Ref&lt;'q, dyn Trait+'q&gt;;

<https://doc.rust-lang.org/reference/lifetime-elision.html#default-trait-object-lifetimes>


## <span class="section-num">15</span> flow control {#flow-control}

Rust 控制流结构包括 if 表达式、match 表达式和循环（loop、while、for）。

Rust 是表达式语言，程序 block 由 分号 结尾的 statement 来组成：

-   如果 expression 不以分号结尾，则它作为 block 的返回值，否则返回 unit type 值 ();

<!--listend-->

```rust
fn main() {
    let x = 5u32;

    let y = {
        let x_squared = x * x;
        let x_cube = x_squared * x;

        // This expression will be assigned to `y`
        x_cube + x_squared + x
    };

    let z = {
        // The semicolon suppresses this expression and `()` is assigned to `z`
        2 * x;
    };

    println!("x is {:?}", x);
    println!("y is {:?}", y);
    println!("z is {:?}", z);
}
```

if-else，if-let，while-let，match，loop，block 等都是表达式，可以用于变量赋值：

```rust
fn main() {
    let n = 5;
    let big_n = if n < 10 && n > -10 {
        println!(", and is a small number, increase ten-fold");
        10 * n
    } else {
        println!(", and is a big number, halve the number");
        n / 2
    }; // 在 let 变量赋值时，} 右边的分号不能省！
    println!("{} -> {}", n, big_n);
}

let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 {
        break 10; // loop break 可以返回值。
    }
};
```

循环支持嵌套，可以使用 break 'label 来跳出循环：label 的格式和 lifetime 一样，都是 'label 格式；'label
必须位于 loop/for/while 或 { 之前：

```rust
fn main() {
    'outer2: { // lable 可以位于 block { 之前
        let mut count = 0;
        'outer: loop {
            'inner1: loop {
                if count >= 20 {
                    // This would break only the inner1 loop
                    break 'inner1; // `break` is also works.
                }
                count += 2;
            }

            count += 5;
            'inner2: loop {
                if count >= 30 {
                    // This breaks the outer loop
                    break 'outer;
                }
                // This will continue the outer loop
                continue 'outer;
            }
        }
        break 'outer2;
    }
    println!("Success!");
}
```

if-let 和 while-let 支持模式匹配语法： if/while let pattern = expression {}：

-   pattern 元素的数量必须与 expression 结果的元素数量一致；
-   pattern 中变量有效 scope 是 expression 右边的 block；

<!--listend-->

```rust
fn main() {
    if let (a, 1) = (2, 4) { // pattern 的元素数量必须与右边一致，结构后的变量 scope 是表达式右边的 block；
        println!("a: {a}")
    } else { // if let 不匹配的情况
        println!("not match!")
    }
}
```

match/if-let/while-let 绑定的变量只是表达式右边的 block 内部有效，一般还需要 outer let 表达式来返回值。Rust 1.65（rustc --edition=2021）开始支持 let-else 语法，let-else 的变量 scope 是所在 block：

-   注意：pattern 也用于赋值析构，这时变量 scope 也是所在 block：

<!--listend-->

```rust
use std::str::FromStr;
fn get_count_item(s: &str) -> (u64, &str) {
    let mut it = s.split(' ');
    // 如果匹配，count_str/item 可以在函数中使用。
    let (Some(count_str), Some(item)) = (it.next(), it.next()) else {
        panic!("Can't segment count item pair: '{s}'");
    };

    let Ok(count) = u64::from_str(count_str) else {
        panic!("Can't parse integer: '{count_str}'");
    };
    (count, item)
}
fn main() {
    assert_eq!(get_count_item("3 chairs"), (3, "chairs"));
}


// 对比，使用 match/if-let 的例子
let (count_str, item) = match (it.next(), it.next()) {
    // count_str/item 只在内部有效
    (Some(count_str), Some(item)) => (count_str, item),
    _ => panic!("Can't segment count item pair: '{s}'"),
};
// count 只在下面匹配的 block 内部有效
let count = if let Ok(count) = u64::from_str(count_str) {
    count
} else {
    panic!("Can't parse integer: '{count_str}'");
};

// 变量析构
// `Pair` owns resources: two heap allocated integers
struct Pair(Box<i32>, Box<i32>);
impl Pair {
    // This method "consumes" the resources of the caller object `self` desugars to `self: Self`
    fn destroy(self) {
        // Destructure `self`
        // self 的两个 Box 被转移到 first 和 second 变量，他们的 scope 是 destroy 函数体。
        let Pair(first, second) = self;
        println!("Destroying Pair({}, {})", first, second);
        // `first` and `second` go out of scope and get freed
    }
}
```

while 用于 true/false 循环：

```rust
fn main() {
    // A counter variable
    let mut n = 1;
    // Loop while `n` is less than 101
    while n < 101 {
        if n % 15 == 0 {
            println!("fizzbuzz");
        } else if n % 3 == 0 {
            println!("fizz");
        } else if n % 5 == 0 {
            println!("buzz");
        } else {
            println!("{}", n);
        }
        // Increment counter
        n += 1;
    }
}
```

while-let 主要用于消除 loop-match 循环模式，while-let 没有 else 子句：

```rust
// Make `optional` of type `Option<i32>`
let mut optional = Some(0);
// Repeatedly try this test.
loop {
    match optional {
        // If `optional` destructures, evaluate the block.
        Some(i) => {
            if i > 9 {
                println!("Greater than 9, quit!");
                optional = None;
            } else {
                println!("`i` is `{:?}`. Try again.", i);
                optional = Some(i + 1);
            }
            // ^ Requires 3 indentations!
        },
        // Quit the loop when the destructure fails:
        _ => { break; }
        // ^ Why should this be required? There must be a better way!
    }
}


// 使用 while-let 语句

// Make `optional` of type `Option<i32>`
let mut optional = Some(0);
// This reads: "while `let` destructures `optional` into `Some(i)`, evaluate the block (`{}`). Else
// `break`.
while let Some(i) = optional {
    if i > 9 {
        println!("Greater than 9, quit!");
        optional = None;
    } else {
        println!("`i` is `{:?}`. Try again.", i);
        optional = Some(i + 1);
    }
    // ^ Less rightward drift and doesn't require explicitly handling the failing case.
}
// ^ `if let` had additional optional `else`/`else if` clauses. `while let` does not have these.
```

for 专用于迭代（for-in），有三种迭代方式:

1.  for item in collect; // item 为元素值;
2.  for item in &amp;collect; // item 为元素值引用 &amp;T;
3.  for item in &amp;mut collect; // item 为元素值可变引用 &amp;mut T;

a..b, a..=b, a.. 都是 RangeXX 语法糖, 可以直接用于 index 操作和 for 迭代:

```rust
fn main() {
    for n in 1..=100 {
        if n == 100 {
            panic!("NEVER LET THIS RUN")
        }
    }
    println!("Success!");

    // 使用范围和 `for` 循环进行倒计时
    for number in (1..4).rev() {
        println!("{}!", number);
    }
}
```

注意, 对于 array 的 into_iter() , 2021 和以前的版本有所变化:

1.  在 2021 以前版本, 如 2018, for i in array.into_iter() 等效为 for i (&amp;array).into_iter(), 所以迭代产生的 i 为数组元素的引用 &amp;T；
2.  2021 版本以后， for i in array.into_iter() 迭代产生的值为数组元素本身：

<!--listend-->

```rust
  fn main() {
      let a = [4, 3, 2, 1];
      // 2021 及以后 v 类型是元素本身， 以前版本是 &T;
      for (i, v) in a.into_iter().enumerate() {
          println!("{i} {v}");
      }

      println!("Success!");
  }

```

? 运算法可以用于 Result/Option, 它可以使用 std::ops::Try trait 来自定义:

-   Try trait 用于自定义 The ? operator and try {} blocks.

<!--listend-->

```rust
pub trait Try: FromResidual {
    type Output;
    type Residual;

    // Required methods
    fn from_output(output: Self::Output) -> Self;
    fn branch(self) -> ControlFlow<Self::Residual, Self::Output>;
}
```


## <span class="section-num">16</span> match {#match}

match expression {} 结果是一个表达式，可以用于变量赋值(值类型必须相同）：

-   子句格式： pattern =&gt; {statements;},  如果是单条语句则可以省略大括号，如 pattern =&gt; expression,
-   match block 中各子句用逗号分割;（函数和闭包的返回值用 -&gt; 分割;）

<!--listend-->

```rust
enum Direction {
    East,
    West,
    North,
    South,
}

fn main() {
    let dire = Direction::South;
    let result = match dire { // match express-表达式
        Direction::East => println!("East"), // println!() 返回 ()
        _ => {
            Ok(1); // 也返回 ()
        }
    }; // let 赋值的结尾分号不能省！
    println!("{result}")
}

// pattern 引入了新的变量，可能会 shadow 以前同名的变量。
fn main() {
    let age = Some(30);
    if let Some(age) = age { // Create a new variable with the same name as previous `age`
        assert_eq!(age, 30);
    } // The new variable `age` goes out of scope here

    match age {
        // Match can also introduce a new shadowed variable
        Some(age) =>  println!("age is a new variable, it's value is {}",age),
        _ => ()
    }
}
```

match!() 宏将 express value 和 pattern 进行匹配，结果为 true/false，可以用于表达式中：

```rust
let foo = 'f';
assert!(matches!(foo, 'A'..='Z' | 'a'..='z'));
let bar = Some(4);
assert!(matches!(bar, Some(x) if x > 2));
```

match pattern 支持的语法：

1.  字面量：如 100，字符串， bool，char；
2.  range：如 0..=100, 'a'..='z';
3.  使用 | 来分割多个 pattern;
4.  使用 _ 来匹配任意值；\_ 是一个特殊的模式，它匹配任何值，但不绑定到变量。
5.  使用 @ 来匹配并定义一个变量，如 y@1..10, y@(1|2|3), y@.. 匹配后，y 是一个包含了匹配值的变量；
6.  枚举：如 Some(value), None, Ok(value), Err(err);
7.  变量：如 name, mut name, ref name, ref mut name, 这里的 mut/ref 是用来修饰生成的变量 name 的类型；
8.  tuple：(key, value), (r, g, b), (r, g, 12); // 12 为字面量匹配条件；
9.  array：[a, b, c], [a, b, 1] // 1 为字面量匹配条件；
10. slice：[a, b], [a, \_, b], [a, .., b]
11. struct：必须列出每一个 field，但是可以使用 .. 来忽略部分 field；
12. 匹配引用：&amp;value, &amp;(k, v); // &amp; 用于匹配表达式结果， value/k/v 都代表解了一层引用后的值；
13. guard expression： x if x &lt; 2;

pattern .. 的用法:

-   对于 array/tuple/slice 元素，可以使用 .. 来省略任意数量部分的元素；
-   对于 struct，可以使用 .. 来省略未列出的 field；
-   只能使用一次 ..;

<!--listend-->

```rust
  struct Point {
      x: i32,
      y: i32,
  }
  fn main() {
      // Fill in the blank to let p match the second arm
      let p = Point { x: 3, y: 10};
      match p {
          // struct pattern，y 的值用来做匹配判断
          Point { x, y: 0 } => println!("On the x axis at {}", x),
          // y: yy@(xx) 是将 y 与 xx 匹配， 如果满足， 匹配的值被设置给变量 yy
          Point { x: 0..=5, y: yy@ (10 | 20 | 30) } => println!("On the y axis at {}", yy),
          Point { x, y } => println!("On neither axis: ({}, {})", x, y),
      }
  }

  // struct 析构和匹配
  fn main() {
      struct Foo {
          x: (u32, u32),
          y: u32,
      }

      // Try changing the values in the struct to see what happens
      let foo = Foo { x: (1, 2), y: 3 };
      match foo {
          Foo { x: (1, b), y } => println!("First of x is 1, b = {},  y = {} ", b, y),
          // you can destructure structs and rename the variables, the order is not important
          Foo { y: 2, x: i } => println!("y is 2, i = {:?}", i),
          // and you can also ignore some variables:
          Foo { y, .. } => println!("y = {}, we don't care about x", y),

          // this will give an error: pattern does not mention field `x`
          //Foo { y } => println!("y = {}", y),
      }

      let faa = Foo { x: (1, 2), y: 3 };
      // You do not need a match block to destructure structs:
      let Foo { x : x0, y: y0 } = faa;
      println!("Outside: x0 = {x0:?}, y0 = {y0}");
      // Destructuring works with nested structs as well:
      struct Bar {
          foo: Foo,
      }

      let bar = Bar { foo: faa };
      let Bar { foo: Foo { x: nested_x, y: nested_y } } = bar;
      println!("Nested: nested_x = {nested_x:?}, nested_y = {nested_y:?}");
  }

  // match guard
  let num = Some(4);
  let split = 5;
  match num {
      // if match guard
      Some(x) if num < split => assert!(x < split),
      Some(x) => assert!(x >= split),
      None => (),
  }


  // (xx) 和 [xx] 都可以作为 pattern：
  let numbers = (2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048);
  match numbers {
      // ERROR: pattern 中最多只能包含一个 ..
      // (first, .., 16, .., 1024, last) => {
      //     assert_eq!(first, 2);
      //     assert_eq!(last, 2048);
      // }

      // OK
      (first, .., 1024, last) => {
          assert_eq!(first, 2);
          assert_eq!(last, 2048);
      }

      // OK
      (first, .., last) => {
          assert_eq!(first, 2);
          assert_eq!(last, 2048);
      }
  }

  // array/slice 也可以解构，使用 [xxx] pattern:
  fn main() {
      // Try changing the values in the array, or make it a slice!
      let array = [1, -2, 6];

      match array {
          // Binds the second and the third elements to the respective variables
          [0, second, third] =>
                  println!("array[0] = 0, array[1] = {}, array[2] = {}", second, third),

          // Single values can be ignored with _
          [1, _, third] => println!(
              "array[0] = 1, array[2] = {} and array[1] was ignored",
              third
          ),

          // You can also bind some and ignore the rest
          [-1, second, ..] => println!(
              "array[0] = -1, array[1] = {} and all the other ones were ignored",
              second
          ),
          // The code below would not compile
          // [-1, second] => ...

          // Or store them in another array/slice (the type depends on that of the value that is being
          // matched against)
          [3, second, tail @ ..] => println!( // 匹配后，tail 是一个包含匹配值的变量。
              "array[0] = 3, array[1] = {} and the other elements were {:?}",
              second, tail
          ),

          // Combining these patterns, we can, for example, bind the first and last values, and store
          // the rest of them in a single array
          [first, middle @ .., last] => println!(
              "array[0] = {}, middle = {:?}, array[2] = {}",
              first, middle, last
          ),
      }
  }

  // enum 解构
  enum Message {
      Hello { id: i32 },
  }
  fn main() {
      let msg = Message::Hello { id: 5 };
      match msg {
          Message::Hello {
              id:  3..=7,
          } => println!("Found an id in range [3, 7]: {}", id),
          // error[E0408]: variable `newid` is not bound in all patterns
          Message::Hello { id: newid@10 | 11 | 12 } => { // 修复： newid@(10 | 11 | 12)
              println!("Found an id in another range [10, 12]: {}", newid)
          }
          Message::Hello { id } => println!("Found some other id: {}", id),
      }
  }

  // 在 pattern 中 & 不能用于 field value:
  if let Person { name: &person_name, age: 18..=150 } = value { }  // 错误
  if let Person {name: ref person_name, age: 18..=150 } = value { } // 正确

  // & 和 * 匹配
  fn main() {
      // Assign a reference of type `i32`. The `&` signifies there is a reference being assigned.
      let reference = &4;

      match reference {
          // If `reference` is pattern matched against `&val`, it results
          // in a comparison like:
          // `&i32`
          // `&val`
          // ^ We see that if the matching `&`s are dropped, then the `i32`
          // should be assigned to `val`.
          &val => println!("Got a value via destructuring: {:?}", val),
      }

      // To avoid the `&`, you dereference before matching.
      match *reference {
          val => println!("Got a value via dereferencing: {:?}", val),
      }

      // What if you don't start with a reference? `reference` was a `&` because the right side was
      // already a reference. This is not a reference because the right side is not one.
      let _not_a_reference = 3;

      // Rust provides `ref` for exactly this purpose. It modifies the assignment so that a reference
      // is created for the element; this reference is assigned.
      let ref _is_a_reference = 3;

      // Accordingly, by defining 2 values without references, references can be retrieved via `ref`
      // and `ref mut`.
      let value = 5;
      let mut mut_value = 6;

      // Use `ref` keyword to create a reference.
      match value {
          ref r => println!("Got a reference to a value: {:?}", r),
      }

      // Use `ref mut` similarly.
      match mut_value {
          ref mut m => {
              // Got a reference. Gotta dereference it before we can add anything to it.
              *m += 10;
              println!("We added 10. `mut_value`: {:?}", m);
          },
      }
  }
```

模式除了用于 if let/while let/match/match! 匹配场景，也用于 tuple/slice/struct/enum 等复杂数据类型值的 `赋值解构` 场景：

-   赋值析构的变量 scope 是所在 block，对于被析构的对象，新的变量 by-ref/by-mov/by-copy 对应的值。

<!--listend-->

```rust
#[derive(PartialEq, PartialOrd)]
struct Centimeters(f64);

// `Inches`, a tuple struct that can be printed
#[derive(Debug)]
struct Inches(i32);
impl Inches {
    fn to_centimeters(&self) -> Centimeters {
        let &Inches(inches) = self;  // & 用于匹配应用类型，这样 inches 是 i32 值。
        Centimeters(inches as f64 * 2.54)
    }
}
```

模式匹配中, &amp; 引用匹配 reference, 而 ref/ref mut 不是用来匹配而是表示绑定的变量类型.

-   变量先匹配再绑定, 绑定时默认是 copy 或 move, 通过指定 ref 或 ref mut 表示变量是引用或可变引用类型;

<!--listend-->

```rust
match struct_value {
    Struct{a: 10, b: 'X', c: false} => (),
    Struct{a: 10, b: 'X', ref c} => (),
    Struct{a: 10, b: 'X', ref mut c} => (),
    Struct{a: 10, b: 'X', c: _} => (),
    Struct{a: _, b: _, c: _} => (),
}

match a {
    None => (),
    Some(value) => (),  // value 被 Copy 或 Moved
}
match a {
    None => (),
    Some(ref value) => (), // value 是引用类型
}

// `name` is moved from person and `age` referenced
let Person { name, ref age } = person;
```

注意:

1.  对于 enum 类型是在枚举 variant 值外部而非内部类匹配 &amp; 或 &amp;mut 的：
2.  对于 tuple 类型, 也是在 tuple 外部匹配 &amp; 或 &amp;mut 的:
3.  &amp;/&amp;mut 用来匹配共享引用和可变引用, &amp;&amp; 或 &amp;&amp;mut 来匹配间接引用: Reference patterns dereference the
    pointers that are being matched and, thus, borrow them.

<!--listend-->

```rust
  let x: &Option<i32> = &Some(3);

  // OK: 等效为 Some(ref y), y 的类型是 &i32
  if let Some(y) = x {
  }
  // OK: 在 variant 外指定 &， y 的类型是 i32
  if let &Some(y) = x {
  }

  // ERROR: 不能在 variant 内指定 &，expected `i32`, found `&_`
  if let Some(&y) = x {
  }

  // 另一个例子
  enum MyEnum {
      A { name: String, x: u8 },
      B { name: String },
  }

  fn a_to_b(e: &mut MyEnum) {
      if let MyEnum::A {
          name,  // &mut String 类型
          x: 0,
      } = e
          {
              *e = MyEnum::B {
                  name: std::mem::take(name), // take 参数类型是 &mut T, 而 name 类型是 &mut String 故满足
              }
          }

      // OK: name 是 String 类型
      // if let &mut MyEnum::A {
      //     name,
      //     x: 0,
      // } = e
  }


  fn main() {
      let (a, b ) = &(1, 2); // a 和 b 都是 &i32 类型
      println!("Results: {a} {b}");
      let &(c, d ) = &(1, 2); // c 和 d 都是 i32 类型
      println!("Results: {c} {d}");

      let (&c, d ) = &(1, 2); // 报错
  }


  let int_reference = &3; // rust 字面量也支持引用
  let a = match *int_reference { 0 => "zero", _ => "some" };
  let b = match int_reference { &0 => "zero", _ => "some" }; // int_reference 是引用类型
  assert_eq!(a, b);
  let int_reference = &3;
  match int_reference {
      &(0..=5) => (),
      _ => (),
  }
```

对于实现了 Deref&lt;Target=U&gt; 的类型 T 值, `&*T 可以返回 &U`:

-   let (a, b) = &amp;v; 这时 a 和 b 都是引用类型, 正确!
-   let &amp;(a, b) = &amp;v; 这时 a 和 b 都是 move 语义. `rust 不允许从引用类型(不管是共享还是可变)值 move 其中的内容` , 所以如果对应值没有实现 copy 则出错.

<!--listend-->

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::thread;

let pair = Arc::new((Mutex::new(false), Condvar::new()));
let pair2 = Arc::clone(&pair);

// Inside of our lock, spawn a new thread, and then wait for it to start.
thread::spawn(move|| {
    // 这里 &*pair2 返回的是 &(Mutex::new(false), Condvar::new()), 赋值解构后, lock 是 &Mutex, cvar
    // 是 &Convar不能使用 let &(lock, cvar) = &*pair2; 这会导致 pair2 中的值发生了移动( lock 是
    // Mutex, cvar 是 Convar), 由于 pair2 和 pair 是共享底层的对象, 所以是不允许移动的.
    let (lock, cvar) = &*pair2;
    let mut started = lock.lock().unwrap();
    *started = true;
    // We notify the condvar that the value has changed.
    cvar.notify_one();
});

// Wait for the thread to start up.
let (lock, cvar) = &*pair;
let mut started = lock.lock().unwrap();
while !*started {
    started = cvar.wait(started).unwrap();
}
```

pattern match 可能会产生 partial move：

-   struct 的 field 可以被 partial move，但是 Vec 等容器不能；
-   struct 被 partial move 的字段后续不能再访问，未被 partial move 的字段还可以访问；

<!--listend-->

```rust
// https://practice.course.rs/ownership/ownership.html
fn main() {
    #[derive(Debug)]
    struct Person {
        name: String,
        age: Box<u8>,
    }

    let person = Person {
        name: String::from("Alice"),
        age: Box::new(20),
    };

    // `name` is moved out of person, but `age` is referenced
    let Person { name, ref age } = person;

    println!("The person's age is {}", age);

    println!("The person's name is {}", name);

    // Error! borrow of partially moved value: `person` partial move occurs
    //println!("The person struct is {:?}", person);

    // `person` cannot be used but `person.age` can be used as it is not moved
    println!("The person's age from person struct is {}", person.age);
```

Vec 等容器不支持 partial move（如果元素实现了 Copy，则不是 move），解决办法：

-   元素 clone();
-   如果能获得元素的 &amp;mut 引用，这可以使用 std::mem:take()/std:mem::replace()  来拥有对应元素；
-   另外 `Vec 不支持解构` ，需要 slice 操作生成 &amp;[T] 后才能解构：

<!--listend-->

```rust
  fn main() {
      let mut data = vec!["abc".to_string()];
      // let e = data[0]; // move occurs because value has type `String`, which does not implement the `Copy` trait

      // Vec 的元素不能被解构， 需要转换为 &[T] 后才能解构。
      //let [a] = data; // pattern cannot match with input type `Vec<String>`

      // if let [a] = data[..] { // move occurs because `a` has type `String`, which does not implement the `Copy` trait
      //     println!("Results: {a:?}");
      // }

      // if let &[a] = &data[..] { // move occurs because `a` has type `String`, which does not implement the `Copy` trait
      //     println!("Results: {a:?}");
      // }

      if let [a] = &data[..] {  // OK: a 是 &String 类型, 引用的是 data 中的元素。
          println!("Results: {a:?}");
      }
  }
```


## <span class="section-num">17</span> function/method/closure {#function-method-closure}


### <span class="section-num">17.1</span> function {#function}

Rust 函数是具有特定名称和参数列表的代码块，可用于执行一个任务或计算一个值。函数的基本语法遵循以下结构：

-   函数声明的顺序没有关系。

<!--listend-->

```rust
fn function_name(parameter1: Type1, parameter2: Type2, ...) -> ReturnType { // 只能有一个返回值类型
    // 函数体，包含执行的代码
}

// 无参数、无返回值的函数
fn greet_world() {
    println!("Hello, world!");
}

// 接受一个参数、无返回值的函数
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

// 接受两个参数、有返回值的函数
fn add(a: i32, b: i32) -> i32 {
    a + b // 表达式的值将作为函数的返回值
}
```

函数使用 fn 关键字声明, 使用 -&gt; 来指定返回值类型(没有指定的默认为 unit type ()), 函数体中最后一个表达式(不以分号结尾)作为函数的返回值, 也可以使用 return 语句提前返回.

```rust
// Unlike C/C++, there's no restriction on the order of function definitions
fn main() {
    // We can use this function here, and define it somewhere later
    fizzbuzz_to(100);
}

// Function that returns a boolean value
fn is_divisible_by(lhs: u32, rhs: u32) -> bool {
    // Corner case, early return
    if rhs == 0 {
        return false;
    }

    // This is an expression, the `return` keyword is not necessary here
    lhs % rhs == 0
}

// Functions that "don't" return a value, actually return the unit type `()`
fn fizzbuzz(n: u32) -> () {
    if is_divisible_by(n, 15) {
        println!("fizzbuzz");
    } else if is_divisible_by(n, 3) {
        println!("fizz");
    } else if is_divisible_by(n, 5) {
        println!("buzz");
    } else {
        println!("{}", n);
    }
}

// When a function returns `()`, the return type can be omitted from the signature
fn fizzbuzz_to(n: u32) {
    for n in 1..=n {
        fizzbuzz(n);
    }
}
```

Rust 2018 版本开始, main 函数支持返回 Result, 当结果是 Err 时会打印错误消息, 程序退出。这意味着可以在 main 函数中使用 ? 来简化错误处理;

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 代码逻辑
    Ok(())
}
```

struct/enum/union 等自定义类型内部不能定义函数，但是可以实现方法，Associated functions 和 Method 都是和类型相关的函数，都需要在类型的 impl 中声明：

-   第一个参数名为 self 时（&amp;self，&amp;mut self 等）是 Method，否则为 Associated functions；
-   &amp;self 等效为 &amp;self: Self; &amp;mut self 等效为 &amp;mut Self, 其中 Self 等效为 impl XX 中的 XX 类型, 当 XX
    类型比较复杂(如泛型)时 Self 可以简洁的代替.
-   这种组织代码的方式让结构体和相关的方法定义分离，同时仍保持了它们在逻辑上的关联。这也是 Rust 实现面向对象编程概念的方式之一。

<!--listend-->

```rust
struct Point {
    x: f64,
    y: f64,
}

// Implementation block, all `Point` associated functions & methods go in here
impl Point {
    // This is an "associated function" because this function is associated with
    // a particular type, that is, Point.
    //
    // Associated functions don't need to be called with an instance.
    // These functions are generally used like constructors.
    fn origin() -> Point {
        Point { x: 0.0, y: 0.0 }
    }

    // Another associated function, taking two arguments:
    fn new(x: f64, y: f64) -> Point {
        Point { x: x, y: y }
    }
}

struct Rectangle {
    p1: Point,
    p2: Point,
}

impl Rectangle {
    // This is a method `&self` is sugar for `self: &Self`, where `Self` is the type of the caller
    // object. In this case `Self` = `Rectangle`
    fn area(&self) -> f64 {
        // `self` gives access to the struct fields via the dot operator
        let Point { x: x1, y: y1 } = self.p1;
        let Point { x: x2, y: y2 } = self.p2;

        // `abs` is a `f64` method that returns the absolute value of the caller
        ((x1 - x2) * (y1 - y2)).abs()
    }

    fn perimeter(&self) -> f64 {
        let Point { x: x1, y: y1 } = self.p1;
        let Point { x: x2, y: y2 } = self.p2;

        2.0 * ((x1 - x2).abs() + (y1 - y2).abs())
    }

    // This method requires the caller object to be mutable `&mut self` desugars to `self: &mut
    // Self`
    fn translate(&mut self, x: f64, y: f64) {
        self.p1.x += x;
        self.p2.x += x;

        self.p1.y += y;
        self.p2.y += y;
    }
}

// `Pair` owns resources: two heap allocated integers
struct Pair(Box<i32>, Box<i32>);
impl Pair {
    // This method "consumes" the resources of the caller object `self` desugars to `self: Self`
    fn destroy(self) {
        // Destructure `self`
        let Pair(first, second) = self; // self 的两个 Box 被转移到 first 和 second 变量；
        println!("Destroying Pair({}, {})", first, second);
        // `first` and `second` go out of scope and get freed
    }
}

fn main() {
    let rectangle = Rectangle {
        // Associated functions are called using double colons
        p1: Point::origin(),
        p2: Point::new(3.0, 4.0),
    };

    // Methods are called using the dot operator Note that the first argument `&self` is implicitly
    // passed, i.e. `rectangle.perimeter()` === `Rectangle::perimeter(&rectangle)`
    println!("Rectangle perimeter: {}", rectangle.perimeter());
    println!("Rectangle area: {}", rectangle.area());

    let mut square = Rectangle {
        p1: Point::origin(),
        p2: Point::new(1.0, 1.0),
    };

    // Error! `rectangle` is immutable, but this method requires a mutable object
    //rectangle.translate(1.0, 0.0);

    // Okay! Mutable objects can call mutable methods
    square.translate(1.0, 1.0);
    let pair = Pair(Box::new(1), Box::new(2));
    pair.destroy();

    // Error! Previous `destroy` call "consumed" `pair`
    //pair.destroy();
}
```

方法的 self 默认类型是 Self，也可以指定 self 的类型，如 Box&lt;T&gt;，这时只能使用该类型的对象来调用方法：

```rust
impl<T> [T]

pub fn into_vec<A>(self: Box<[T], A>) -> Vec<T, A> where A: Allocator

let s: Box<[i32]> = Box::new([10, 40, 30]);
let x = s.into_vec();
// `s` cannot be used anymore because it has been converted into `x`.
assert_eq!(x, vec![10, 40, 30]);
```


### <span class="section-num">17.2</span> method lookup {#method-lookup}

The Dot Operator: <https://doc.rust-lang.org/stable/nomicon/dot-operator.html>

The `dot operator` will perform a lot of magic to `convert types`. It will perform `auto-referencing`,
`auto-dereferencing`, and `coercion` until types match. The detailed mechanics of method lookup are
defined here, but here is a brief overview that outlines the main steps.

Suppose we have a function `foo` that has a receiver (a self, &amp;self or &amp;mut self parameter). If we
call `value.foo()`, the compiler needs to determine what type Self is before it can call the correct
implementation of the function. For this example, we will say that value has type T. We will use
fully-qualified syntax to be more clear about exactly which type we are calling a function on.

1.  First, the compiler checks if it can `call T::foo(value) directly`. This is called `a "by value"`
    method call. `即先看 T 是否直接实现方法 foo();`
2.  If it can't call this function (for example, if the function has the wrong type or a trait isn't
    implemented for Self), then the compiler `tries to add in an automatic reference`. This means that
    the compiler tries `<&T>::foo(value) and <&mut T>::foo(value)` . This is called `an "autoref" method
       call`. `再看 T 的 &T 和 &mut T 类型是否实现了方法 foo();`
3.  If none of these candidates worked, it `dereferences T and tries again`. This uses the `Deref
       trait` - if T: Deref&lt;Target = U&gt; then it tries again with type U instead of T.
    -   如果 T 是引用类型, 则解引用它;
    -   如果 T 不是引用类型, 但是实现了 Deref trait, 则 \* 解引用它获得 U 类型, 然后对 U 重新执行 1-2 步骤.

例如：标准库为所有 &amp;T 或 &amp;mut T 实现了 Deref&lt;Target = T&gt; trait, 这意味 &amp;T 可以调用 T 上定义的方法：

```rust
// https://doc.rust-lang.org/src/core/ops/deref.rs.html#84
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> Deref for &T {
    type Target = T;

    #[rustc_diagnostic_item = "noop_method_deref"]
    fn deref(&self) -> &T {
        *self
    }
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> !DerefMut for &T {}

#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> Deref for &mut T {
    type Target = T;

    fn deref(&self) -> &T {
        *self
    }
}
```

If it can't dereference T, it can also `try unsizing T`. This just means that if T has a size
parameter known at compile time, it "forgets" it for the purpose of resolving methods. For instance,
this unsizing step can `convert [i32; 2] into [i32]` by "forgetting" the size of the array.

-   unsize Trait 是 [rust 语言定义的](https://doc.rust-lang.org/stable/std/marker/trait.Unsize.html), 用于将 size 类型转换为 unsize 类型, 例如：[i32; 2] into [i32]，
    myType into dyn Display。
-   Rust array [T; N] 会被 unsizing 转换为 [T]（非 Deref 转换）, 所以 slice 上定义的方法都可以被
    arrary 使用！

<!--listend-->

```rust
trait Tst {
    fn tst(&self);
}

impl Tst for &str {
    fn tst(&self) {
        println!("receiver_type=&&str {{{}}}", self);
    }
}
// tst 方法的 self 类型是 &&str

fn main() {
    let x = "hello world";
    x.tst(); // 根据 x 的类型来查找 self 匹配的 tst() 方法.(不止匹配 x 类型)

   let y = String::from("hello world");
   y.tst(); // 报错, 尝试了 self 的类型 String, &String, &mut String, str, &str, &mut str 都不匹配上面 tst() 的签名.
   (&y).tst() // 正确
}
```

示例：

```rust
let array: Rc<Box<[T; 3]>> = ...;
let first_entry = array[0]; // 等效为 *array.index(0)
```

1.  Then, the compiler checks if Rc&lt;Box&lt;[T; 3]&gt;&gt; implements Index, but it does not, and neither do
    &amp;Rc&lt;Box&lt;[T; 3]&gt;&gt; or &amp;mut Rc&lt;Box&lt;[T; 3]&gt;&gt;.
2.  Since none of these worked, the compiler dereferences the `Rc<Box<[T; 3]>> into Box<[T; 3]>` and
    tries again.
3.  Box&lt;[T; 3]&gt;, &amp;Box&lt;[T; 3]&gt;, and &amp;mut Box&lt;[T; 3]&gt; do not implement Index, so it dereferences again.
4.  `[T; 3]` and its autorefs also do not implement Index.
5.  It can't dereference [T; 3], so the compiler `unsizes it`, giving [T]. Finally, `[T] implements
       Index` , so it can now call the actual index function.


### <span class="section-num">17.3</span> closure {#closure}

Rust 中的闭包是一种匿名函数，可以将它们保存在变量中或作为参数传递给其他函数。闭包能够捕获并使用其定义作用域内的变量，这是它们名称的由来——它们"封闭"并包围了周围的环境。

A closure expression produces a closure value with a unique, anonymous type that cannot be written out. A closure type is approximately equivalent to a struct which contains the captured variables. For instance, the following closure:

闭包在定义时（而非调用时）capture 外围环境中的对象，这种 capture 是 closures 函数内部的行为，不体现在闭包的函数参数中。

-   使用 || 而非 () 来定义输入参数；
-   如果是单行表达式，可以忽略大括号，否则需要使用大括号；
-   可以省略返回值声明，默认根据表达式自动推导；
-   闭包的输入和输出参数一旦被自动推导后，就不能再变化，后续多次调用时传的或返回的值类型必须相同；
-   对比：普通函数 fn 的参数类型必须指定，普通函数 fn 不能捕获上下文中的变量对象；

<!--listend-->

```rust
|val| val + x

fn main() {
    let outer_var = 42;

    // A regular function can't refer to variables in the enclosing environment
    //fn function(i: i32) -> i32 { i + outer_var }

    // Closures are anonymous, here we are binding them to references.  Annotation is identical to
    // function annotation but is optional as are the `{}` wrapping the body. These nameless
    // functions are assigned to appropriately named variables.
    let closure_annotated = |i: i32| -> i32 { i + outer_var }; // block 中 return 或最后一个表达式值作为返回
    let closure_inferred  = |i     |          i + outer_var  ; // 单行表达式的结果作为值返回

    // Call the closures.
    println!("closure_annotated: {}", closure_annotated(1));
    println!("closure_inferred: {}", closure_inferred(1));

    // Once closure's type has been inferred, it cannot be inferred again with another type.
    //println!("cannot reuse closure_inferred with another type: {}", closure_inferred(42i64));

    // A closure taking no arguments which returns an `i32`.  The return type is inferred.
    let one = || 1;
    println!("closure returning one: {}", one());
}

fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 }; // 闭包是一个表达式，可以赋值
let add_one_v4 = |x|               x + 1  ;

let color = String::from("green");
let print = move || println!("`color`: {}", color); // move

// 两次函数调用的推导类型不一致，编译失败。
let example_closure = |x| x;
let s = example_closure(String::from("hello"));
let n = example_closure(5);

fn main() {
    // Increment via closures and functions.
    fn function(i: i32) -> i32 { i + 1 }

    // Closures are anonymous, here we are binding them to references
    //
    // These nameless functions are assigned to appropriately named variables.
    let closure_annotated = |i: i32| -> i32 { i + 1 };
    let closure_inferred  = |i     |          i + 1  ;

    let i = 1;
    // Call the function and closures.
    println!("function: {}", function(i));
    println!("closure_annotated: {}", closure_annotated(i));
    println!("closure_inferred: {}", closure_inferred(i));

    // A closure taking no arguments which returns an `i32`.
    // The return type is inferred.
    let one = || 1;
    println!("closure returning one: {}", one());

}
```

闭包是一个表达式，也可以作为函数返回类型，struct/enum 的成员类型：

```rust
struct Cacher<T,E> where T: Fn(E) -> E, E: Copy
  {
      query: T,
      value: Option<E>,
  }

impl<T,E> Cacher<T,E> where T: Fn(E) -> E, E: Copy
  {
      fn new(query: T) -> Cacher<T,E> {
          Cacher {
              query,
              value: None,
          }
      }

      fn value(&mut self, arg: E) -> E {
          match self.value {
              Some(v) => v,
              None => {
                  let v = (self.query)(arg);
                  self.value = Some(v);
                  v
              }
          }
      }
  }

#[test]
fn call_with_different_values() {
    let mut c = Cacher::new(|a| a);
    let v1 = c.value(1);
    let v2 = c.value(2);
    assert_eq!(v2, 1);
}
```

在闭包函数中使用外部环境的对象时，Rust 编译器会分析闭包使用的方式来确定如何捕获该对象：

1.  Imut Refer：不可变引用外围环境中的对象，优先选择该类型。
2.  Mut Refer：可变引用外围环境中的对象；
3.  Move：外围对象的所有权移动到闭包函数中；
    -   例如闭包内部需要 drop() non-copy 对象或者需要返回对象（转移所有权到接收方）；
    -   更常见的情况是多线程场景，使用 move || 来指示闭包获得上下文对象的所有权；

<!--listend-->

```rust
let s = String::from("coolshell");
let take_str = || s; // s 作为闭包返回值，所以是 Move 语义。
println!("{}", s); // s 已经被 move 进闭包，所以不能再访问。
println!("{}",  take_str()); // OK

// 在定义闭包时捕获环境中对象
fn main() {
    let mut count = 0;
    let mut inc = || {
        count += 1;  // Mut refer
        println!("`count`: {}", count);
    };
    inc();
    assert_eq!(count, 1); // 闭包后续不再使用，故还可以继续访问 count
}

// 错误的情况
fn main() {
    let mut count = 0;
    let mut inc = || {
        count += 1;  // Mut refer
        println!("`count`: {}", count);
    };
    inc();
    assert_eq!(count, 1);  // inc() 继续有效的情况下，count 还是保持 &mut，所以这里 cannot borrow
    // `count` as immutable because it is also borrowed as mutable
    inc();
}

fn main() {
    use std::mem;

    let color = String::from("green");

    // A closure to print `color` which immediately borrows (`&`) `color` and stores the borrow and
    // closure in the `print` variable. It will remain borrowed until `print` is used the last time.
    //
    // `println!` only requires arguments by immutable reference so it doesn't impose anything more
    // restrictive.
    let print = || println!("`color`: {}", color);  // print 在有效的情况下，一致保有 color 的共享引用

    // Call the closure using the borrow.
    print();

    // `color` can be borrowed immutably again, because the closure only holds an immutable
    // reference to `color`.
    let _reborrow = &color;
    print();

    // A move or reborrow is allowed after the final use of `print`
    let _color_moved = color; // print 后续不再使用，所以可以 move color


    let mut count = 0;
    // A closure to increment `count` could take either `&mut count` or `count` but `&mut count` is
    // less restrictive so it takes that. Immediately borrows `count`.
    //
    // A `mut` is required on `inc` because a `&mut` is stored inside. Thus, calling the closure
    // mutates `count` which requires a `mut`.
    let mut inc = || {
        count += 1;
        println!("`count`: {}", count);
    };

    // Call the closure using a mutable borrow.
    inc();

    // The closure still mutably borrows `count` because it is called later.  An attempt to reborrow
    // will lead to an error.
    // let _reborrow = &count;   // inc 还有效（因为后面还在调用），所以 count 一直处于 inc 的 &mut 状态

    inc();

    // The closure no longer needs to borrow `&mut count`. Therefore, it is possible to reborrow
    // without an error
    let _count_reborrowed = &mut count;


    // A non-copy type.
    let movable = Box::new(3);

    // `mem::drop` requires `T` so this must take by value. A copy type would copy into the closure
    // leaving the original untouched.  A non-copy must move and so `movable` immediately moves into
    // the closure.
    let consume = || {
        println!("`movable`: {:?}", movable);
        mem::drop(movable);
    };

    // `consume` consumes the variable so this can only be called once.
    consume();
    // consume();
}
```

如果要明确的外围环境中的对象 ownership 移动到闭包中，需要使用 move，这时闭包相当于具有了 'static
lifetime。这在多线程场景非常常见：因为多线程场景的闭包函数在另一个 thread 中运行，编译器不能推断闭包引用对象的声明周期是否有效，所以需要 move，这样确保闭包捕获的对象是一直有效的。

```rust
use std::thread;

fn main() {
    let list = vec![1, 2, 3];
    println!("Before defining closure: {:?}", list);

    thread::spawn(move || println!("From thread: {:?}", list)) // 闭包函数使用 ownership 接管而非引用来捕获外界对象。
        .join()
        .unwrap();
}
```

对于 move 到闭包中的变量对象，闭包外不能再使用（借用）该变量：

```rust
// OK
fn main() {
    let color = String::from("green");
    let print = move || println!("`color`: {}", color);
    print();

    let _reborrow = &color; // error[E0382]: borrow of moved value: `color`
    println!("{}",_reborrow);
}

// OK
fn main() {
    let movable = Box::new(3);
    let consume = move || {
        println!("`movable`: {:?}", movable);
    };
    consume();
    consume(); // OK， consume 保持对 movable 变量的 move，所以可以多次调用。
}

// Error
fn main() {
    let movable = Box::new(3);

    // consume 只能调用一次，因为它内部将 movable 变量 move 走了。
    let consume = || {
        println!("`movable`: {:?}", movable);
        take(movable);
    };
    consume();

    consume(); //  // error[E0382]: use of moved value: `consume`。closure cannot be invoked more
    //  than once because it moves the variable `movable` out of its environment
}
fn take<T>(_v: T) {}
```

外围对象被闭包捕获后，不允许再修改它的值：

-   实际上，一旦对象被借用（不管是共享还是可变借用），只要该借用还有效，对象都不能被修改或 move。

<!--listend-->

```rust
fn main() {
    let mut a = 123;
    let ar = &a;
    a = 456; // Error：cannot assign to `a` because it is borrowed
    println!("{ar}")
}

fn main() {
    let mut x = 4;
    let add_to_x = |y| y + x; // x 被共享借用

    let result = add_to_x(3);
    println!("The result is {}", result); // 输出：The result is 7

    x = x + 3; // 在被共享借用的有效情况下, 不能修改其值
    let result2 = add_to_x(3);
    println!("The result2 is {}", result2);
}
```

编译器为闭包表达式创建一个匿名的闭包类型，该类型类似于 struct，可以捕获（优先 &amp;，其次是 &amp;mut 或移动语义）环境中的对象。编译器根据闭包中使用环境对象的方式，来确定：

1.  捕获环境对象的方式：ref，mut ref or move；
2.  匿名闭包类型该实现哪一个 trait：Fn/FnMut/FnOnce。

<!--listend-->

```rust
fn f<F : FnOnce() -> String> (g: F) {
    println!("{}", g());
}
let mut s = String::from("foo");
let t = String::from("bar");
f(|| {
    s += &t;
    s
});
// Prints "foobar".

// generates a closure type roughly like the following:
struct Closure<'a> {
    s : String,
    t : &'a String,
}

impl<'a> FnOnce<()> for Closure<'a> {
    type Output = String;
    fn call_once(self) -> String {
        self.s += &*self.t;
        self.s
    }
}

// so that the call to f works as if it were:
f(Closure{s: s, t: &t});
```

匿名闭包类型实现了哪一个 trait：Fn/FnMut/FnOnce，当闭包作为 `函数输入参数 或限界 Bound` 需要指定该类型。

1.  FnOnce 类型：只能调用一次，该闭包接管了外界环境中的值；
2.  FnMut 类型：可以调用多次，该闭包使用 &amp;mut 来捕获外界值；
3.  Fn 类型：可以调用多次，该闭包使用 &amp;T 来捕获外界值；
4.  Fn 是 FnMut 子类型, FnMut 是 FnOnce 子类型；

<!--listend-->

```rust
//   std::ops::FnOnce
pub trait FnOnce<Args> where Args: Tuple,
          {
              type Output;
              // Required method
              extern "rust-call" fn call_once(self, args: Args) -> Self::Output; // 传入 self
          }

// std::ops::FnMut
pub trait FnMut<Args>: FnOnce<Args> where Args: Tuple, // FnMut 是 FnOnce 子类型
          {
              // Required method
              extern "rust-call" fn call_mut( &mut self, args: Args ) -> Self::Output; // 传入 &mut self
          }

pub trait Fn<Args>: FnMut<Args> where Args: Tuple, // Fn 是 FnMut 子类型
  {
      // Required method
      extern "rust-call" fn call(&self, args: Args) -> Self::Output; // 传入 &self
  }

// 例子 1 `F` must implement `Fn` for a closure which takes no inputs and returns nothing - exactly
// what is required for `print`.
fn apply<F>(f: F) where F: Fn() {
    f();
}
fn main() {
    let x = 7;

    // Capture `x` into an anonymous type and implement `Fn` for it. Store it in `print`.
    let print = || println!("{}", x);

    apply(print);
}

// 例子2
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}
fn main() {
    let mut list = [
        Rectangle { width: 10, height: 1 },
        Rectangle { width: 3, height: 5 },
        Rectangle { width: 7, height: 12 },
    ];
    list.sort_by_key(|r| r.width); // 编译器推断该闭包符合 FnMutt 要求，虽然它没有捕获外围任何对象
    println!("{:#?}", list);
}

impl<T> [T] {
    pub fn sort_by_key<K, F>(&mut self, mut f: F)
    where
        F: FnMut(&T) -> K, // F 是 FnMutt 类型，且输入是 &T
        K: Ord,
    {
        stable_sort(self, |a, b| f(a).lt(&f(b)));
    }
}

// 例子 3
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let mut list = [
        Rectangle { width: 10, height: 1 },
        Rectangle { width: 3, height: 5 },
        Rectangle { width: 7, height: 12 },
    ];

    let mut sort_operations = vec![];
    let value = String::from("by key called");

    list.sort_by_key(|r| { // 编译器推断该闭包为 FnOnce 类型，不符合 sort_by_key() 方法的要求 FnMut
        sort_operations.push(value);
        r.width
    });
    println!("{:#?}", list);
}
```

对于没有捕获环境中值的闭包，可以自动转换为 fn 指针：

```rust
let add = |x, y| x + y;
let mut x = add(5,7);
type Binop = fn(i32, i32) -> i32;
let bo: Binop = add;
x = bo(5,7);
```

如果一个函数输入参数类型是闭包，则也可以传入函数指针（fn 类型对象），反之则不行。

闭包作为匿名类型，可以作为 `函数输入参数、限界 Bound、函数的返回值` ，与其他类型一样，也可以实现
Send/Sync/Copy/Clone trait，具体取决于捕获的对象类型，例如：如果所有捕获的对象都实现了 Send，则闭包也实现了 Send。

闭包也可以作为函数返回值返回，但 Fn/FnMut/FnOnce 都是 trait，所以需要使用 `impl Trait` 或 `&dyn Trait` 或
`Box<dyn Trait>` 语法。另外，必须使用 move 关键字，否则当函数返回时，闭包捕获的引用值会失效。

-   impl trait 是编译时静态确定的唯一类型；&amp;dyn trait 和 Box&lt;dyn trait&gt; 是运行时动态分发的类型；

<!--listend-->

```rust
fn create_fn() -> impl Fn() {
    let text = "Fn".to_owned();
    move || println!("This is a: {}", text)
}

fn create_fnmut() -> impl FnMut() {
    let text = "FnMut".to_owned();
    move || println!("This is a: {}", text)
}

fn create_fnonce() -> impl FnOnce() {
    let text = "FnOnce".to_owned();
    move || println!("This is a: {}", text)
}

fn main() {
    let fn_plain = create_fn();
    let mut fn_mut = create_fnmut();
    let fn_once = create_fnonce();
    fn_plain();
    fn_mut();
    fn_once();
}
```

闭包 lifetime：闭包函数返回引用时可能会遇到 lifetime 问题（[14.3](#org-target--lifetime-elision-rules) 并不适合闭包）：

```rust
fn fn_elision(x: &i32) -> &i32 { x } // OK

let closure_elision = |x: &i32| -> &i32 { x }; // Error
|     let closure = |x: &i32| -> &i32 { x }; // fails
|                       -        -      ^ returning this value requires that `'1` must outlive `'2`
|                       |        |
|                       |        let's call the lifetime of this reference `'2`
|                       let's call the lifetime of this reference `'1`



```

解决办法：

1.  使用 nightly toolchain 和开启 #\![feature(closure_lifetime_binder)]，这样可以为闭包函数指定 for
    &lt;'a&gt; 语法的 lifetime：<https://github.com/rust-lang/rust/issues/97362>
2.  或者，定义一个 helper 函数，该函数可以指定闭包输入、输出参数所需的 lifetime，然后内部调用闭包；
3.  或者，将闭包转换为 fn 函数指针类型，函数指针[支持使用 for&lt;'a&gt;](https://doc.rust-lang.org/reference/types/function-pointer.html)来定义 lifetime 参数；
    <https://stackoverflow.com/a/60906558>

<!--listend-->

```rust
// 解决办法1:
fn main() {
    // let clouse_test = |input: &String| input; // error: lifetime may not live long enough
    //let clouse_test = |input: &String| -> &String {input}; // error: lifetime may not live long enough
    // let clouse_test = |input: &'a String| ->&'a String {input}; // error[E0261]: use of undeclared lifetime name `'a`

    let clouse_test = for <'a> |input: &'a String| ->&'a String {input}; // 需要使用 nightly toolchain 和开启 #![feature(closure_lifetime_binder)]
    println!("Results:")
}

// 解决办法2:
fn testStr<'a> (input: &'a String) -> &'a String {
    let closure_test = |input: &'a String | -> &'a String {input}; // 闭包使用外围 helper 函数定义的 lifetime
    return closure_test(input);
}

fn main() {
    // let clouse_test = |input: &String| input; // error: lifetime may not live long enough

    //let clouse_test = |input: &String| -> &String {input}; // error: lifetime may not live long enough

    // let clouse_test = |input: &'a String| ->&'a String {input}; // error[E0261]: use of undeclared lifetime name `'a`

    // let clouse_test = for <'a> |input: &'a String| ->&'a String {input};

    println!("Results:{}", testStr(&"asdfab".to_string()));
}

// 解决办法3:
// 将闭包转换为 fn 函数指针，函数指针支持使用 for<'a> 来定义高阶函数，而且编译期间大小是已知的。
// 但是不能使用 Fn/FnMut/FnOnce 等 trait 类型。
let test_fn: for<'a> fn(&'a _) -> &'a _ = |p: &String| p;
println!("Results:{}", test_fn(&"asdfab".to_string()));

// 其他例子：https://github.com/rust-lang/rust/pull/56746/files
#![allow(unused)]

fn willy_no_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x| { p }; // no type annotation at all
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn willy_ret_type_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x| -> &str { p }; // type annotation on the return type
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn willy_ret_region_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x| -> &'w str { p }; // type+region annotation on return type
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn willy_arg_type_ret_type_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x: &str| -> &str { p }; // type annotation on arg and return types
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn willy_arg_type_ret_region_annot<'w>(p: &'w str, q: &str) -> &'w str {
    let free_dumb = |_x: &str| -> &'w str { p }; // fully annotated
    let hello = format!("Hello");
    free_dumb(&hello)
}

fn main() {
    let world = format!("World");
    let w1: &str = {
        let hello = format!("He11o");
        willy_no_annot(&world, &hello)
    };
    let w2: &str = {
        let hello = format!("He22o");
        willy_ret_type_annot(&world, &hello)
    };
    let w3: &str = {
        let hello = format!("He33o");
        willy_ret_region_annot(&world, &hello)
    };
    let w4: &str = {
        let hello = format!("He44o");
        willy_arg_type_ret_type_annot(&world, &hello)
    };
    let w5: &str = {
        let hello = format!("He55o");
        willy_arg_type_ret_region_annot(&world, &hello)
    };
    assert_eq!((w1, w2, w3, w4, w5),
        ("World","World","World","World","World"));
}
}
```

如果要给闭包 move 传参， 比如 clone 后的值或 refer 值， 则建议使用 Use variable rebinding in a
separate scope for that.

```rust
// https://rust-unofficial.github.io/patterns/idioms/pass-var-to-closure.html
use std::rc::Rc;
let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);
let closure = {
    // `num1` is moved
    let num2 = num2.clone();  // `num2` is cloned
    let num3 = num3.as_ref();  // `num3` is borrowed
    move || {
        *num1 + *num2 + *num3;
    }
};

// 而非，因为下面的 num2_cloned/num3_borrowed 的作用域在闭包之外还有效，但是它们已经 move 到闭包了
use std::rc::Rc;
let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);
let num2_cloned = num2.clone();
let num3_borrowed = num3.as_ref();
let closure = move || {
    *num1 + *num2_cloned + *num3_borrowed;
};
```


### <span class="section-num">17.4</span> HRTB Fn/fn {#hrtb-fn-fn}

HRTB 一般只会在 Fn 作为 Bound 时会使用到, 下面没有加 lifetime 的代码是可以正常编译的：

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
      where F: Fn(&(u8, u16)) -> &u8,
  {
      fn call(&self) -> &u8 {
          (self.func)(&self.data)
      }
  }

fn do_it(data: &(u8, u16)) -> &u8 { &data.0 }

fn main() {
    let clo = Closure { data: (0, 1), func: do_it };
    println!("{}", clo.call());
}
```

如果要给上面的代码添加 lifetime bound 则会遇到 F 的 lifetime 该如何指定的问题：

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
            // where F: Fn(&'??? (u8, u16)) -> &'??? u8,
        {
            fn call<'a>(&'a self) -> &'a u8 {
                (self.func)(&self.data)
            }
        }

fn do_it<'b>(data: &'b (u8, u16)) -> &'b u8 { &'b data.0 }

fn main() {
    'x: {
        let clo = Closure { data: (0, 1), func: do_it };
        println!("{}", clo.call());
    }
}


// Error1:
impl<'a, F> Closure<F>
             where F: Fn(&'a (u8, u16)) -> &'a u8,
        {
            fn call<'a>(&'a self) -> &'a u8 { // lifetime name `'a` shadows a lifetime name that is
                // already in scope
                (self.func)(&self.data)
            }
        }

// Error2:
impl<'a, F> Closure<F>
             where F: Fn(&'a (u8, u16)) -> &'a u8,
        {
            fn call<'b>(&'b self) -> &'b u8 { //  method was supposed to return data with lifetime `'a`
                //  but it is returning data with lifetime `'b`
                (self.func)(&self.data)
            }
        }

// Error3:
impl<'a, F> Closure<F>
          where F: Fn(&'a (u8, u16)) -> &'a  u8,
      {
          fn call(& self) -> & u8 { // rustc 自动为 &self 添加 liefitime 如 '1: method was supposed to
              // return data with lifetime `'1` but it is returning data with
              // lifetime `'a`
              (self.func)(&self.data)
          }
      }

// Error4: 可以编译过，但是要求 Closure 的 liefitime 和传入的 Fn 的参数 lifetime 一致，不符合预期语
// 义（Fn 的函数有自己独立的 lifetime，和 Closure 对象 lifetime 无关）。
impl<'a, F> Closure<F>
        where F: Fn(&'a (u8, u16)) -> &'a  u8,
    {
        fn call(&'a self) -> &'a u8 {
            (self.func)(&self.data)
        }
    }

// OK：HRTB
impl<F> Closure<F> // 1. 泛型参数中没有 'a lifetitme
      where F: for <'a> Fn(&'a (u8, u16)) -> &'a u8, // 2. 在 F 的 Bound 中使用 for <'a> 来声明 'a lifetime
  {
      fn call<'a>(&'a self) -> &'a u8 { // 'a 和上面的 for <'a> 没有任何关系，是 call() 方法自己的
          // lifetime。由于 rustc 会自动加 lifetime，所以不指定：fn
          // call(&self) -> &u8
          (self.func)(&self.data)
      }
  }

```

可见 HRTB 语法 for &lt;'a&gt; ：一般 `只在 Fn Bound 中使用，for <'a> Fn 表示 Fn 满足任意 'a liftime` ，所以
Fn 也满足 call() 方法的lifetime 要求，可以在 call() 方法中使用。

另外，fn 函数指针也可以使用 for &lt;'a&gt; 来指定 lifetime：

```rust
let test_fn: for<'a> fn(&'a _) -> &'a _ = |p: &String| p; // OK
let test_fn: for<'a> Fn(&'a _) -> &'a _ = |p: &String| p; // Error，因为 Fn 是 trait
```

HRTB 有两种等效语法：

```rust
where F: for<'a> Fn(&'a (u8, u16)) -> &'a u8,
// 等效为
where for <'a> F: Fn(&'a (u8, u16)) -> &'a u8,
```


### <span class="section-num">17.5</span> Diverging functions {#diverging-functions}

发散函数 Diverging functions 指的是不返回的函数,它使用 ! 来表示: 发散函数可以用在需要表示永不返回的逻辑，例如无限循环、调用 panic! 宏或者退出程序。

```rust
  fn foo() -> ! {
      panic!("This call never returns.");
  }

  fn loop_forever() -> ! {
      loop {
          println!("I will loop forever!");
      }
  }
```

对比: unit type 值 () 还是有唯一的值 ():

```rust
fn some_fn() { // 未指定返回值,等效为返回 ();
    ()
}

fn main() {
    let _a: () = some_fn();
    println!("This function returns and you can see this line.");
}
```

! 也可以作为 loop {} 或 std::os::exit() 函数的返回值.

! 也可以作为类型:

```rust
#![feature(never_type)]

fn main() {
    let x: ! = panic!("This call never returns.");
    println!("You will never see this line!");
}
```

为什么要使用发散函数来作为一个无限循环的返回，正常的单元函数返回，然后指定一个不可退出的循环条件不也可以实现么。这里返回一个! 而不是 () 有其特定的用途，主要取决于类型系统和编译器的行为。


### <span class="section-num">17.6</span> 高阶函数 {#高阶函数}

Rust 中的高阶函数是指那些可以接受一个或多个函数作为参数，或者返回一个函数的函数。这些函数广泛应用于迭代器、闭包和函数式编程模式中。

map变换: map 方法可以接受一个闭包，对迭代器中的每一个元素应用这个闭包，然后生成一个新的迭代器。

```rust
fn main() {
    let vec = vec![1, 2, 3];
    let doubled: Vec<i32> = vec.iter().map(|x| x * 2).collect();
    println!("{:?}", doubled); // 输出：[2, 4, 6]
}
```

filter过滤: filter 方法接受一个闭包，该闭包返回布尔值，用于决定是否保留元素。

```rust
fn main() {
    let vec = vec![1, 2, 3, 4, 5, 6];
    let even: Vec<i32> = vec.into_iter().filter(|x| x % 2 == 0).collect();
    println!("{:?}", even); // 输出：[2, 4, 6]
}
```

fold累计: fold 方法接受一个初始累加值和一个闭包，闭包将当前元素和上一次累加的结果结合生成新的累加结果。

```rust
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    let sum: i32 = vec.iter().fold(0, |acc, x| acc + x);
    println!("{}", sum); // 输出：15
}
```

find查找元素: find 方法接受一个闭包，返回第一个满足闭包条件的元素的引用。

```rust
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    let first_even = vec.iter().find(|&&x| x % 2 == 0);
    match first_even {
        Some(&n) => println!("{}", n), // 输出：2
        None => println!("No even numbers"),
    }
}
```

iter 和 for_each 进行迭代: iter 方法创建一个不可变引用的迭代器，而 for_each 方法接受一个闭包来对每个元素执行一些操作。

```rust
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    vec.iter().for_each(|x| println!("{}", x));
    // 输出：
    // 1
    // 2
    // 3
    // 4
    // 5
}
```

闭包作为返回值: 可以定义一个函数返回一个实现了特定功能的闭包：

```rust
fn make_multiplier(factor: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |n| n * factor)
}

fn main() {
    let doubler = make_multiplier(2);
    println!("Double 5 is {}", doubler(5)); // 输出：Double 5 is 10
}
```

在这个例子中，make_multiplier 函数返回一个闭包，该闭包捕获了 factor 变量，并用于乘以传入的参数。


## <span class="section-num">18</span> generic/trait {#generic-trait}

泛型类型/函数/方法的泛型参数使用 &lt;CamelCase, ...&gt; 表示：

```rust
struct A;          // Concrete type `A`.
struct S(A);       // Concrete type `S`.
struct SGen<T>(T); // 泛型函数，自带泛型类型参数。

// 具体函数
fn reg_fn(_s: S) {}
fn gen_spec_t(_s: SGen<A>) {} // 实例化后的 具体类型(concrete type) SGen<A>
fn gen_spec_i32(_s: SGen<i32>) {}
// 泛型函数, 类型参数位于函数名后的 <xx> 中。
fn generic<T>(_s: SGen<T>) {}
fn main() {
    // Using the non-generic functions
    reg_fn(__);          // Concrete type.
    gen_spec_t(__);   // Implicitly specified type parameter `A`.
    gen_spec_i32(__); // Implicitly specified type parameter `i32`.
    // 调用泛型方法时，使用比目鱼语法： method::<type>()
    // Explicitly specified type parameter `char` to `generic()`.
    generic::<char>(__);
    println!("Success!");
}


// 泛型类型 struct
struct Point<T> { x: T, y: T, }
fn main() {
    let p = Point{x: "5".to_string(), y : "hello".to_string()}; // 自动推导
    println!("Success!");
}

// 泛型类型和方法, 类型名称位于类型名后的 <xx> 中。
struct Val<T> { val: T, }
// 为泛型类型实现关联函数或方法时，impl 后面需要列出泛型类型所需的所有泛型参数
impl<T> Val<T> {
    fn value(&self) -> &T { // 方法可以使用 impl 后的泛型参数类型。
        &self.val
    }
}
fn main() {
    let x = Val{ val: 3.0 }; // x 是 Val<f64> 类型
    let y = Val{ val: "hello".to_string()}; // y 是 Val<String> 类型
    println!("{}, {}", x.value(), y.value()); // 3.0, hello
}

// 函数和方法也可以定义自己的泛型参数, 它们不需要在 impl 后列出.
use std::fmt::Debug;
struct Val<T> {
    val: T,
}
impl<T> Val<T> {
    fn value(&self) -> &T {
        &self.val
    }
    fn add<O: Debug>(&self, other: O) { // 泛型方法有自己的类型参数 O
        println!("{:?}", other);
    }

}
fn main() {
    let x = Val { val: 3.0 };
    let y = Val {
        val: "hello".to_string(),
    };
    println!("{}, {}", x.value(), y.value());

    x.add(123); // 推断
    x.add::<i32>(123); // 为泛型方法手动指定类型，使用比目鱼语法。
}
```

impl&lt;T&gt; 语法:

1.  定义泛型类型的方法或关联函数：impl&lt;T&gt; MyStruct&lt;T&gt; {}
2.  或为类型定义泛型 trait 的实现: impl&lt;T&gt; MyTrait for MyStruct&lt;T&gt; {}

<!--listend-->

```rust
struct S; // Concrete type `S`
struct GenericVal<T>(T); // Generic type `GenericVal`

// impl of GenericVal where we explicitly specify type parameters:
impl GenericVal<f32> {} // Specify `f32`
impl GenericVal<S> {} // Specify `S` as defined above

// `<T>` Must precede the type to remain generic
impl<T> GenericVal<T> {}

// 示例
struct Val {
    val: f64,
}
struct GenVal<T> {
    gen_val: T,
}
// impl of Val
impl Val {
    fn value(&self) -> &f64 {
        &self.val
    }
}
// impl of GenVal for a generic type `T`
impl<T> GenVal<T> {
    fn value(&self) -> &T {
        &self.gen_val
    }
}
fn main() {
    let x = Val { val: 3.0 };
    let y = GenVal { gen_val: 3i32 };

    println!("{}, {}", x.value(), y.value());
}

// 另一个例子
// Non-copyable types.
struct Empty;
struct Null;

// A trait generic over `T`.
trait DoubleDrop<T> {
    // Define a method on the caller type which takes an
    // additional single parameter `T` and does nothing with it.
    fn double_drop(self, _: T);
}

// Implement `DoubleDrop<T>` for any generic parameter `T` and
// caller `U`.
impl<T, U> DoubleDrop<T> for U {
    // This method takes ownership of both passed arguments,
    // deallocating both.
    fn double_drop(self, _: T) {}
}

fn main() {
    let empty = Empty;
    let null  = Null;

    // Deallocate `empty` and `null`.
    empty.double_drop(null);

    //empty;
    //null;
    // ^ TODO: Try uncommenting these lines.
}
```

trait 支持关联类型, 即需要是 impl trait 时指定的 type

-   关联类型不需要作为 trait 的泛型参数来指定；
-   关联类型可以指定缺省值，在实现该 trait 或将 trait 作为 bound 约束时，可以指定关联类型的具体类型。
-   后续也可以对关联类型进行限界.

<!--listend-->

```rust
  // `A` and `B` are defined in the trait via the `type` keyword.  (Note: `type` in this context is
  // different from `type` when used for aliases).
  trait Contains {
      type A;
      type B;

      // Updated syntax to refer to these new types generically.
      fn contains(&self, _: &Self::A, _: &Self::B) -> bool;
  }


  // Without using associated types, 在 impl 时需要明确指定 A/B/C 类型
  fn difference<A, B, C>(container: &C) -> i32 where
      C: Contains<A, B> { ... }
  // Using associated types
  fn difference<C: Contains>(container: &C) -> i32 { ... }

  trait Iterator {
      type Item;
      fn next(&mut self) -> Option<Self::Item>;
  }

  /// 一个只输出偶数的示例
  struct EvenNumbers {
      count: usize,
      limit: usize,
  }
  impl Iterator for EvenNumbers {
      type Item = usize; // 实现时指定 Item 类型

      fn next(&mut self) -> Option<Self::Item> {
          if self.count > self.limit {
              return None;
          }
          let ret = self.count * 2;
          self.count += 1;
          Some(ret)
      }
  }
  fn main() {
      let nums = EvenNumbers { count: 1, limit: 5 };
      for n in nums {
          println!("{}", n);
      }
  }
  // 依次输出  2 4 6 8 10
```

对于有缺省参数 + 关联类型的泛型 trait, 可以同时指定参数+关联类型, 例如, 在使用 Add 进行限界时可以使用 Add&lt;&amp;T, Output=T&gt; , 但不能使用 Add&lt;Rhs=&amp;T, Output=T&gt;, Rust 报错没有 Rhs 关联类型;

```rust
pub trait Add<Rhs = Self> { // 泛型 trait，类型参数有缺省值
    type Output = Self; // 有缺省值的关联类型，实现该 trait 时需要具体指定类型。

    // Required method
    fn add(self, rhs: Rhs) -> Self::Output;
}


use std::ops::Add;
// 定义一个包含 Add trait 作为泛型限界的结构体
struct MyStruct<T> {
    value: T,
}
// 定义一个包含 Add trait 作为泛型限界的函数，同时指定 Rhs 类型和关联类型
fn add_values<T, Rhs>(a: T, b: Rhs) -> T
    where
        T: Add<Rhs, Output = T>, // 不能使用 Rhs = Rhs, 否则 Rust 会报错没有 Rhs 关联类型;
    {
        a + b
    }

fn main() {
    let struct_a = MyStruct { value: 10 };
    let scalar_b = 20;

    // 在这里，指定 Rhs 类型为 i32，确保加法操作的结果类型是相同的
    let result = add_values(struct_a.value, scalar_b);

    println!("Result: {}", result);
}

// 可以在实现 trait 时实例化泛型 trait 的参数。
impl ops::Add<Bar> for Foo {
    type Output = FooBar;
    fn add(self, _rhs: Bar) -> FooBar {
        FooBar
    }
}
```

trait 也支持常量参数（例如 `<const N: usize>` ），这样可以在泛型类型/函数/方法中使用 Array：

-   实例化常量参数时，可以使用 {XXX} 常量表达式。

<!--listend-->

```rust
  struct ArrayPair<T, const N: usize> {
      left: [T; N],
      right: [T; N],
  }

  impl<T: Debug, const N: usize> Debug for ArrayPair<T, N> {
      // ...
  }

  // 实例化常量参数
  fn foo<const N: usize>() {}
  fn bar<T, const M: usize>() {
      foo::<M>(); // Okay: `M` is a const parameter
      foo::<2021>(); // Okay: `2021` is a literal
      foo::<{20 * 100 + 20 * 10 + 1}>(); // Okay: const expression contains no generic parameters

      foo::<{ M + 1 }>(); // Error: const expression contains the generic parameter `M`
      foo::<{ std::mem::size_of::<T>() }>(); // Error: const expression contains the generic parameter `T`

      let _: [u8; M]; // Okay: `M` is a const parameter
      let _: [u8; std::mem::size_of::<T>()]; // Error: const expression contains the generic parameter `T`
  }

  fn main() {}
```

其他 trait bound 的格式：

1.  Bounds written after declaring a generic parameter: fn f&lt;A: Copy&gt;() {} is the same as fn f&lt;A&gt;()
    where A: Copy {}.
2.  In trait declarations as supertraits: trait Circle : Shape {} is equivalent to trait Circle where
    Self : Shape {}.
3.  In trait declarations as bounds on associated types: trait A { type B: Copy; } is equivalent to
    trait A where Self::B: Copy { type B; }.


### <span class="section-num">18.1</span> 泛型 trait 和 blanket impl {#泛型-trait-和-blanket-impl}

trait 作为一种类型类型，本身也可以是泛型定义的。trait 也可以作为泛型类型参数的限界（bound）约束，也可以作为函数的参数或返回值（impl Trait）：

-   匿名函数也可以作为类型参数的 bound 约束，例如函数指针 fn() -&gt; u32, 或闭包 Fn() -&gt; u32;

<!--listend-->

```rust
struct Pair<T> {
    x: T,
    y: T,
}
impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y, }
    }
}

// trait 作为泛型参数的 bound
impl<T: std::fmt::Debug + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {:?}", self.x);
        } else {
            println!("The largest member is y = {:?}", self.y);
        }
    }
}

trait Contains<A, B> {
    fn contains(&self, _: &A, _: &B) -> bool;
    fn first(&self) -> i32;
    fn last(&self) -> i32;
}
impl Contains<i32, i32> for Container { // 实现泛型 trait 时，可以直接指定具体的泛型参数类型。
    fn contains(&self, number_1: &i32, number_2: &i32) -> bool {
        (&self.0 == number_1) && (&self.1 == number_2)
    }
    // Grab the first number.
    fn first(&self) -> i32 { self.0 }

    // Grab the last number.
    fn last(&self) -> i32 { self.1 }
}

// 泛型参数可以使用多个 trait 作为限界,同时也支持 lifetime 作为限界
use std::fmt::{Debug, Display};
fn compare_prints<T: Debug + Display>(t: &T) {
    println!("Debug: `{:?}`", t);
    println!("Display: `{}`", t);
}
fn compare_types<T: Debug, U: Debug>(t: &T, u: &U) {
    println!("t: `{:?}`", t);
    println!("u: `{:?}`", u);
}
fn main() {
    let string = "words";
    let array = [1, 2, 3];
    let vec = vec![1, 2, 3];
    compare_prints(&string);
    //compare_prints(&array);
    // TODO ^ Try uncommenting this.
    compare_types(&array, &vec);
}

// 对于复杂的限界,可以使用 where 来定义, where 必须位于 { 前, 用逗号分割
impl <A: TraitB + TraitC, D: TraitE + TraitF> MyTrait<A, D> for YourType {}
// Expressing bounds with a `where` clause
impl <A, D> MyTrait<A, D> for YourType
      where
           A: TraitB + TraitC,
           D: TraitE + TraitF {}

// 有些场景,必须使用 where 来限界
use std::fmt::Debug;
trait PrintInOption {
    fn print_in_option(self);
}
// Because we would otherwise have to express this as `T: Debug` or
// use another method of indirect approach, this requires a `where` clause:
impl<T> PrintInOption for T where Option<T>: Debug {
    // We want `Option<T>: Debug` as our bound because that is what's
    // being printed. Doing otherwise would be using the wrong bound.
    fn print_in_option(self) {
        println!("{:?}", Some(self));
    }
}

fn main() {
    let vec = vec![1, 2, 3];

    vec.print_in_option();
}
```

可以为泛型类型 T 实现泛型 Trait，这样可以批量对已知或未知的类型实现 trait, 例如为所有 &amp;T 或 &amp;mut T
实现 Deref&lt;Target = T&gt; trait, 这意味 &amp;T 调用 T 上定义的方法。

```rust
// https://doc.rust-lang.org/src/core/ops/deref.rs.html#84

#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> Deref for &T {
    type Target = T;

    #[rustc_diagnostic_item = "noop_method_deref"]
    fn deref(&self) -> &T {
        *self
    }
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> !DerefMut for &T {}

#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> Deref for &mut T {
    type Target = T;

    fn deref(&self) -> &T {
        *self
    }
}
```

一般情况下定义函数或方法使用的 trait 或 type 必须是本 crate package 定义的，称为 `Orphan Rule` ，这是为了避免破坏开发者原来的定义。通常的解决办法是使用 `newtype pattern` 来解决这个问题：

```rust
use std::fmt;
sturct Pretty(String) // a newtype Pretty， 相当于为 String 实现 Display

impl fmt::Display for Pretty {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "\"{}\"", self.0.clone() + ", world")
    }
}
fn main() {
    let w = Pretty("hello".to_string());
    println!("w = {}", w);
}
```

但是对于泛型 Trait 则放宽了这个条件限制：可以在本 crate package 没有定义 trait 或 type 时，给它们实现方向函数或方法，这种实现称为 `blanket impl` ，也就是给实现某个 trait 的类型定义自动定义其他 trait 实现：

-   例如为任意 T 类型实现 Borrow&lt;T&gt; trait：
-   balnket impl：It is an implement of a trait either for all types, or for all types that match some
    condition.

<!--listend-->

```rust
// https://doc.rust-lang.org/src/core/borrow.rs.html#207
impl<T: ?Sized> Borrow<T> for T {
    #[rustc_diagnostic_item = "noop_method_borrow"]
    fn borrow(&self) -> &T {
        self
    }
}

// https://doc.rust-lang.org/src/core/convert/mod.rs.html#217
pub trait AsRef<T: ?Sized> {
    /// Converts this type into a shared reference of the (usually inferred) input type.
    #[stable(feature = "rust1", since = "1.0.0")]
    fn as_ref(&self) -> &T;
}

// 在其他 create 中实现 AsRef 泛型

// OsStr 是当前 crate 的类型
// https://doc.rust-lang.org/src/std/ffi/os_str.rs.html#1452
impl AsRef<OsStr> for String {
    #[inline]
    fn as_ref(&self) -> &OsStr {
        (&**self).as_ref()
    }
}

// str 是当前 crate 的类型
// https://doc.rust-lang.org/src/alloc/string.rs.html#2580
impl AsRef<str> for String {
    #[inline]
    fn as_ref(&self) -> &str {
        self // self 为 &String 类型，编译器通过 Defref 转换为 &str
    }
}
```

如果明确指定了泛型 trait 的类型参数, 则 impl 该 trait 时不需要指定泛型参数:

```rust
struct Container(i32, i32);

// A trait which checks if 2 items are stored inside of container.
// Also retrieves first or last value.
trait Contains<A, B> {
    fn contains(&self, _: &A, _: &B) -> bool; // Explicitly requires `A` and `B`.
    fn first(&self) -> i32; // Doesn't explicitly require `A` or `B`.
    fn last(&self) -> i32;  // Doesn't explicitly require `A` or `B`.
}

impl Contains<i32, i32> for Container {
    // True if the numbers stored are equal.
    fn contains(&self, number_1: &i32, number_2: &i32) -> bool {
        (&self.0 == number_1) && (&self.1 == number_2)
    }

    // Grab the first number.
    fn first(&self) -> i32 { self.0 }

    // Grab the last number.
    fn last(&self) -> i32 { self.1 }
}
```


### <span class="section-num">18.2</span> supertrait {#supertrait}

trait 可以定义父子关系，即一个 trait 可以有多个父 trait（称为 supertrait），某个类型在实现该 trait
的同时也必须实现这些父 trait：

-   In trait declarations as supertraits: `trait Circle : Shape {}` is equivalent to `trait Circle where
      Self : Shape {}` .

<!--listend-->

```rust
trait Person {
    fn name(&self) -> String;
}

// Person is a supertrait of Student.
// Implementing Student requires you to also impl Person.
trait Student: Person {
    fn university(&self) -> String;
}

trait Programmer {
    fn fav_language(&self) -> String;
}

// CompSciStudent (computer science student) is a subtrait of both Programmer and
// Student. Implementing CompSciStudent requires you to impl both supertraits.
trait CompSciStudent: Programmer + Student {
    fn git_username(&self) -> String;
}
```


### <span class="section-num">18.3</span> 完全限定方法调用 {#完全限定方法调用}

由于一个对象可以实现不同 crate package 定义的同名 trait（如 Write，它们可以有相同或不同的方法），所以在 VecK&lt;u8&gt; 上调用 trait 定义的方法时，必须确保对应的 trait 被引入到作用域，否则 rust 不知道该调用哪一个 trait 的方法实现：

-   Cone/Iterator 等 trait 是 rust 自动导入的标准库 std preledge trait，不需要手动引入。

<!--listend-->

```rust
use std::io::Write

let mut buf: Vec<u8> = vec![];
buf.write_all(b"hello")?; // Vec<u8L> 实现了 Write trait，调用 Write trait 方法 write_all 时必须引入 Write trait 定义。
```

完全限定方法调用: Rust 中的完全限定语法（Fully Qualified Syntax）可以解决命名冲突或混淆的问题。这是因为你明确地指出了你想要调用特定 trait 的方法，即使你的类型对多个 trait 实现了同名的方法。

```rust
trait CalSum {
    fn sum(&self, a: i32) -> i32;
}

trait CalSumV2 {
    fn sum(&self, a: i32) -> i32;
}

#[derive(Clone, Copy, Debug)]
struct S1(i32);

impl CalSum for S1 {
    fn sum(&self, a: i32) -> i32 {
        self.0 + a
    }
}

impl CalSumV2 for S1 {
    fn sum(&self, a: i32) -> i32 {
        self.0 + a
    }
}

fn main() {
    let s1 = S1(0);

    // 调用指定 trait 的方法
    println!("Results: {}", CalSum::sum(&s1, 1)); // OK
    println!("Results: {}", CalSumV2::sum(&s1, 1)); // OK

    // ERROR: S1 同时实现了 CalSum 和 CalSumV2, 它们都提供了 sum() 方法, rust 不确定该调用哪一个.
    //println!("Results: {}", s1.sum(1)); // Error:  multiple `sum` found

    // 使用完全限定方法调用.
    println!("Results: {}", <S1 as CalSum>::sum(&s1, 1));// OK
}
```

使用完全限定的语法 `<Type as Trait>::function` 来清楚地调用特定的 'sum' 实现, 如 &lt;S1 as
CalSum&gt;::sum(&amp;s1, 1) 明确表示我们要调用 S1 实现的 CalSum 的 sum() 方法。在没有歧义的情况下, 可以直接调用对象实现的 trait 的方法, 而不需要使用完全限定调用的方式:

```rust
   let i: i8 = Default::default();
   let (x, y): (Option<String>, f64) = Default::default();
   let (a, b, (c, d)): (i32, u32, (bool, bool)) = Default::default();

   #[derive(Default)]
   struct SomeOptions {
       foo: i32,
       bar: f32,
   }

   fn main() {
       // 如果 SomeOptions 没有实现除 Default 外的其它 trait 的 default() 方法, 则
       // <SomeOptions as Default>::default() 可以简写为 Default::default().
       let options: SomeOptions = Default::default();
       let options = SomeOptions { foo: 42, ..Default::default() };
   }
```


### <span class="section-num">18.4</span> trait 使用及原理分析 {#trait-使用及原理分析}

<https://liujiacai.net/blog/2021/04/27/trait-usage/>

在 Rust 设计目标中，零成本抽象是非常重要的一条，它让 Rust 具备高级语言表达能力的同时，又不会带来性能损耗。零成本的基石是泛型与 trait，它们可以在编译期把高级语法编译成与高效的底层代码，从而实现运行时的高效。这篇文章就来介绍 trait，包括使用方式与三个常见问题的分析，在问题探究的过程中来阐述其实现原理。

基本用法

Trait 的主要作用是用来抽象行为，类似于其他编程语言中的「接口」，这里举一示例阐述 trait 的基本使用方式：

```rust
trait Greeting {
    fn greeting(&self) -> &str;
}

struct Cat;
impl Greeting for Cat {
    fn greeting(&self) -> &str {
        "Meow!"
    }
}

struct Dog;
impl Greeting for Dog {
    fn greeting(&self) -> &str {
        "Woof!"
    }
}
```

在上述代码中，定义了一个 trait Greeting，两个 struct 实现了它，根据函数调用方式，主要两种使用方式：

-   基于泛型的 `编译时静态派发`
-   基于 trait object 的 `运行时动态派发`

泛型的概念比较常见，这里着重介绍下 trait object：A trait object is an opaque value of another type
that implements a set of traits. The set of traits is made up of an object safe base trait plus any
number of auto traits.

比较重要的一点是 trait object 属于 `Dynamically Sized Types（DST，类似于 str ）` ，在编译期无法确定大小，只能通过指针来间接访问，常见的形式有 `Box<dyn trait>, &dyn trait` 等。

```rust
fn print_greeting_static<G: Greeting>(g: G) {
    println!("{}", g.greeting());
}
fn print_greeting_dynamic(g: Box<dyn Greeting>) {
    println!("{}", g.greeting());
}

print_greeting_static(Cat);
print_greeting_static(Dog);

print_greeting_dynamic(Box::new(Cat));
print_greeting_dynamic(Box::new(Dog));
```

静态派发：在 Rust 中， `泛型的实现采用的是单态化（monomorphization）` ，会针对不同类型的调用者，在编译时生成不同版本的函数，所以泛型也被称为类型参数。好处是没有虚函数调用的开销，缺点是最终的二进制文件膨胀。在上面的例子中， print_greeting_static 会编译成下面这两个版本：

```rust
print_greeting_static_cat(Cat);
print_greeting_static_dog(Dog);
```

动态派发：是所有函数的调用都能在编译期确定调用者类型，一个常见的场景是 GUI 编程中事件响应的 callback，一般来说一个事件可能对应多个 callback 函数，而这些 callback 函数都是 `在编译期不确定的` ，因此泛型在这里就不适用了，需要采用动态派发的方式：

```rust
trait ClickCallback {
    fn on_click(&self, x: i64, y: i64);
}

struct Button {
    listeners: Vec<Box<dyn ClickCallback>>,
}
```

impl trait：在 Rust 1.26 版本中，引入了一种新的 trait 使用方式，即：impl trait，可以用在两个地方：=
函数参数与返回值= 。 该方式主要是简化复杂 trait 的使用(声明语法)，算是泛型的特例版，因为在使用 impl
trait 的地方，也是 `静态派发` ，而且作为函数返回值时， `数据类型只能有一种` ，这一点要尤为注意！

```rust
fn print_greeting_impl(g: impl Greeting) {
    println!("{}", g.greeting());
}
print_greeting_impl(Cat);
print_greeting_impl(Dog);

// 下面代码会编译报错
fn return_greeting_impl(i: i32) -> impl Greeting {
    if i > 10 {
        return Cat;
    }
    Dog
}

// | fn return_greeting_impl(i: i32) -> impl Greeting {
// |                                    ------------- expected because this return type...
// |     if i > 10 {
// |         return Cat;
// |                --- ...is found to be `Cat` here
// |     }
// |     Dog
// |     ^^^ expected struct `Cat`, found struct `Dog`
```

关联类型：在上面介绍的基本用法中，trait 中方法的参数或返回值类型都是确定的， `Rust 提供了类型「惰性绑定」的机制` ，即关联类型（associated type），这样就能 `在实现 trait 时再来确定类型` ，一个常见的例子是标准库中的Iterator，next 的返回值为 Self::Item ：

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

/// 一个只输出偶数的示例
struct EvenNumbers {
    count: usize,
    limit: usize,
}
impl Iterator for EvenNumbers {
    type Item = usize; // 实现时指定 Item 类型

    fn next(&mut self) -> Option<Self::Item> {
        if self.count > self.limit {
            return None;
        }
        let ret = self.count * 2;
        self.count += 1;
        Some(ret)
    }
}
fn main() {
    let nums = EvenNumbers { count: 1, limit: 5 };
    for n in nums {
        println!("{}", n);
    }
}
// 依次输出  2 4 6 8 10
```

关联类型的使用和泛型相似，Iterator 也可使用泛型来定义：

```rust
pub trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```

它们的区别主要在于：

-   一个特定类型（比如上文中的 Cat）可以 `多次实现泛型 trait` 。比如对于 From&lt;T&gt;，可以有 impl From&lt;&amp;str&gt;
    for Cat 也可以有 impl From&lt;String&gt; for Cat
-   但是对于关联类型的 trait， `只能实现一次` 。比如对于 FromStr，只能有 impl FromStr for Cat ，类似的
    trait 还有 Iterator Deref

Derive： Rust 中，可以使用 derive 属性来实现一些常用的 trait，比如：Debug/Clone 等，对于用户自定义的
trait，也可以实现过程宏支持 derive，具体可参考：How to write a custom derive macro? ，这里不再赘述。

常见问题

不支持向上转型（upcast）

对于 trait SubTrait: Base (SubTrait 是 Base 的子 trait) ，在目前的 Rust 版本中，是无法将 &amp;dyn
SubTrait 转换到 &amp;dyn Base 的。这个限制与 trait object 的内存结构有关。

在 Exploring Rust fat pointers 一文中，该作者通过 transmute 将 trait object 的引用转为两个 usize，并且验证它们是指向数据与函数虚表的指针：

```rust
use std::mem::transmute;
use std::fmt::Debug;

fn main() {
    let v = vec![1, 2, 3, 4];
    let a: &Vec<u64> = &v;
    // 转为 trait object
    let b: &dyn Debug = &v;
    println!("a: {}", a as *const _ as usize);
    println!("b: {:?}", unsafe { transmute::<_, (usize, usize)>(b) });
}

// a: 140735227204568
// b: (140735227204568, 94484672107880)
```

从这里可以看出：Rust 使用 fat pointer（除了一个指针外，还包含另外一些信息，类似于 str ） 来表示
trait object 的引用，分别指向 data 与 vtable，这和 Go 中的 interface 十分类似。可以用下面的伪代码来表示：

![](https://img.alicdn.com/imgextra/i2/581166664/O1CN01esAA7q1z6A3inQpnF_!!581166664.jpg)
trait object reference

```rust
pub struct TraitObjectReference {
    pub data: *mut (),
    pub vtable: *mut (),
}

struct Vtable {
    destructor: fn(*mut ()),
    size: usize,
    align: usize,
    method: fn(*const ()) -> String,
}
```

尽管 fat pointer 导致指针体积变大（无法使用 Atomic 之类指令），但是好处是更明显的：

-   由于 vtable 与原始对象是分开的，所以可以为已有类型实现新的 trait（比如 blanket implementations）
-   调用虚表中的函数时，只需要引用一次，而在 C++ 中，vtable 是存在对象内部的，导致每一次函数调用都需要两次引用，如下图所示：

![](https://img.alicdn.com/imgextra/i2/581166664/O1CN01u6ms841z6A3cHRdJw_!!581166664.jpg)
cpp vtable two-level indirect

如果 trait 有继承关系时，vtable 是怎么存储不同 trait 的方法的呢？在目前的实现中，是依次存放在一个
vtable 中的，如下图：

![](https://img.alicdn.com/imgextra/i4/581166664/O1CN01x8adaQ1z6A3bkyKqY_!!581166664.png)
多 trait 时 vtable 示意图

可以看到， `所有 trait 的方法是顺序放在一起，并没有区分方法属于哪个 trait，这样也就导致无法进行upcast`
，社区内有 RFC 2765 在追踪这个问题，感兴趣的读者可参考，这里就不讨论解决方案了，介绍一种比较通用的解决方案，通过引入一个 AsBase 的 trait 来解决：

```rust
trait Base {
    fn base(&self) {
        println!("base...");
    }
}

trait AsBase {
    fn as_base(&self) -> &dyn Base;
}

// blanket implementation
impl<T: Base> AsBase for T {
    fn as_base(&self) -> &dyn Base {
        self
    }
}

trait Foo: AsBase {
    fn foo(&self) {
        println!("foo..");
    }
}

#[derive(Debug)]
struct MyStruct;

impl Foo for MyStruct {}
impl Base for MyStruct {}

fn main() {
    let s = MyStruct;
    let foo: &dyn Foo = &s;
    foo.foo();
    let base: &dyn Base = foo.as_base();
    base.base();
}
```

向下转型（downcast）

向下转型是指把一个 trait object 再转为之前的具体类型，Rust 提供了 `Any` 这个 trait 来实现这个功能。

```rust
pub trait Any: 'static {
    fn type_id(&self) -> TypeId;
}
```

大多数类型都实现了 Any，只有那些包含非 'static 引用的类型没有实现。通过 type_id 就能够在运行时判断类型，下面看一示例：

```rust
use std::any::Any;
trait Greeting {
    fn greeting(&self) -> &str;
    fn as_any(&self) -> &dyn Any;
}

struct Cat;
impl Greeting for Cat {
    fn greeting(&self) -> &str {
        "Meow!"
    }
    fn as_any(&self) -> &dyn Any {
        self
    }
}

fn main() {
    let cat = Cat;
    let g: &dyn Greeting = &cat;
    println!("greeting {}", g.greeting());

    // &Cat 类型
    let downcast_cat = g.as_any().downcast_ref::<Cat>().unwrap();
    println!("greeting {}", downcast_cat.greeting());
}
```

上面的代码重点在 downcast_ref，其实现为：

```rust
pub fn downcast_ref<T: Any>(&self) -> Option<&T> {
    if self.is::<T>() {
        unsafe { Some(&*(self as *const dyn Any as *const T)) }
    } else {
        None
    }
}
```

可以看到，在类型一致时，通过 unsafe 代码把 trait object 引用的第一个指针（即 data 指针）转为了指向具体类型的引用。

Object safety
在 Rust 中，并不是所有的 trait 都可用作 trait object，需要满足一定的条件，称之为 object safety 属性。

主要有以下几点：

-   函数返回类型不能是 Self（即当前类型）。这主要因为把一个对象转为 trait object 后，原始类型信息就丢失了，所以这里的 Self 也就无法确定了。
-   函数中不允许有泛型参数。主要原因在于单态化时会生成大量的函数，很容易导致 trait 内的方法膨胀。比如

<!--listend-->

```rust
trait Trait {
 fn foo<T>(&self, on: T);
 // more methods
}

// 10 implementations
fn call_foo(thing: Box<Trait>) {
 thing.foo(true); // this could be any one of the 10 types above
 thing.foo(1);
 thing.foo("hello");
}

// 总共会有 10 * 3 = 30 个实现
// Trait 不能继承 Sized。这是由于 Rust 会默认为 trait object 实现该 trait，生成类似下面的代码：

trait Foo {
 fn method1(&self);
 fn method2(&mut self, x: i32, y: String) -> usize;
}

// autogenerated impl
impl Foo for TraitObject {
 fn method1(&self) {
     // `self` is an `&Foo` trait object.

     // load the right function pointer and call it with the opaque data pointer
     (self.vtable.method1)(self.data)
 }
 fn method2(&mut self, x: i32, y: String) -> usize {
     // `self` is an `&mut Foo` trait object

     // as above, passing along the other arguments
     (self.vtable.method2)(self.data, x, y)
 }
}
```

如果 Foo 继承了 Sized，那么就要求 trait object 也是 Sized，而 trait object 是 DST 类型，属于 ?Sized
，所以 trait 不能继承 Sized。更多参考：rust-blog/sizedness-in-rust: Trait Objects

总结

本文开篇就介绍了 trait 是实现零成本抽象的基础，通过 trait 可以为已有类型增加新方法，这其实解决了表达式问题，可以进行运算符重载，可以进行面向接口编程等。

希望通过本文的分析，可以让读者更好的驾驭 trait 的使用，对于非 safe 的 trait，能修改成 safe 是最好的方案，否则可以尝试泛型的方式。


### <span class="section-num">18.5</span> trait object {#trait-object}

<span class="org-target" id="org-target--trait-object"></span>

`dyn TraitName` 称为 trait object，可用于表示实现 TraitName 的任意对象类型。由于这些类型的 size 是不确定的，所以一般使用 `&dyn TraitName` 或 `Box<dyn TraitName>` 来转换为可以固定大小的类型。

-   特殊的 trait Any 一般用于 downcase() 到具体的类型.

dyn 语法：dyn TypeParamBounds，其中 TypeParamBounds 是一系列具有如下限制的 trait bound：

1.  All traits except the first trait must be auto traits；
    -   [auto trait](https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits) 包括：Send, Sync, Unpin, UnwindSafe 和  RefUnwindSafe。
2.  there may not be more than one lifetime
3.  opt-out bounds (e.g. ?Sized) are not allowed
4.  paths to traits may be parenthesized.

注:

1.  Rust 2021 前版本 dyn 关键字是可选的;
2.  dyn 优先级低，后面的 Bound 才是一个语法元素.

示例：

```rust
dyn Trait
dyn Trait + Send
dyn Trait + Send + Sync
dyn Trait + 'static
dyn Trait + Send + 'static
dyn Trait +
dyn 'static + Trait.
dyn (Trait)
dyn Trait + 'a // 可以为 trait object 指定 lifetime
dyn eviction::EvictionManager + Sync  // 可以使用 path 来完整指定 trait
```

两个 dyn trait 的 trait、lifetime 如果相同的话，则类型互为别名，例如：dyn Trait + Send + UnwindSafe
is the same as dyn Trait + UnwindSafe + Send.

dyn trait 是编译时未知大小，故一般和指针联合使用，如 &amp;dyn TraitName 或 Box&lt;dyn TraitName&gt;。它们都是胖指针，包含两个字段：指向动态对象内存的指针 + 该对象实现 trait 定义的各种方法的虚表（vtable) 的指针。
&lt;= 运行时动态派发

当使用 &amp;dyn Trait 时，如果要指定 Send/lifetime 等，则需要使用 `&(dyn Trait + Send + 'a) 格式` ，否则
Rust 报错 + 号有歧义：

-   当参数类型是 &amp;dyn Trait 时，需要传入实现该 Trait 的借用类型值，如 MyStruct 实现了 MyTrait, 则需要使用 &amp;dyn Trait 的地方需要传入 &amp;MyStruct 的值。
-   当参数类型是 &amp;Box&lt;dyn Trait&gt; 时，不能传入 &amp;Box::new(my_struct)， Rust 报错类型不匹配，两个解决办法：
    -   将参数类型修改为 Box&lt;dyn Trait&gt;, 后续可以直接传入 Box::new(my_struct)；
    -   或者，先创建一个 Box&lt;dyn Triat&gt; 的对象 b，然后再传入 &amp;b;
-   lifetime 都是在声明时指定，在实际 borrow 时不需要指定 lifetime，否则 Rust 报错。

<!--listend-->

```rust
trait MyTrait {
    fn say_hello(&self);
}

struct MyStruct(String);

impl MyTrait for MyStruct {
    fn say_hello(&self) {
        println!("hello {}", self.0);
    }
}

fn main() {
    // let b = &'a dyn MyTrait + Send + 'static; // error: expected expression, found keyword `dyn`
    // let b = &'a(dyn MyTrait + Send + 'static); // error: borrow expressions cannot be annotated with lifetimes

    let my_struct = MyStruct("abc".to_string());

    //printf_hello(Some(my_struct)); // expected `&dyn MyTrait`, found `MyStruct`

    // printf_hello(Some(&'static my_struct)); // error: borrow expressions cannot be annotated with lifetimes: help: remove the lifetime annotation
    printf_hello(Some(&my_struct));

    // printf_hellov2(Some(&Box::new(my_struct))); // error[E0308]: mismatched types，expected `&Box<dyn MyTrait + Send>`, found `&Box<MyStruct>`
    // 解决办法：先创建一个 Box 对象，然后再借用。
    let st1: Box<dyn MyTrait + Send + 'static> = Box::new(my_struct);
    printf_hellov2(Some(&st1));

    let my_struct2 = MyStruct("def".to_string());
    printf_hellov3(Some(Box::new(my_struct2)));
}

//fn printf_hello<'a>(say_hello: Option<&'a dyn MyTrait + Send>) { // error: ambiguous `+` in a type: use parentheses to disambiguate: `(dyn MyTrait + Send)`
fn printf_hello(say_hello: Option<&(dyn MyTrait + Send)>) {
    if let Some(my_trait) = say_hello {
        my_trait.say_hello();
    }
}

fn printf_hellov2(say_hello: Option<&Box<dyn MyTrait + Send + 'static>>) {
    // if let Some( &my_trait) = say_hello { // error[E0507]: cannot move out of `*say_hello` as enum variant `Some` which is behind a shared reference
    if let Some(my_trait) = say_hello {
        my_trait.say_hello();
    }
}

fn printf_hellov3(say_hello: Option<Box<dyn MyTrait + Send + 'static>>) {
    if let Some(my_trait) = say_hello {
        my_trait.say_hello();
    }
}
```

dyn Trait 的 lifetime 规则:

1.  `Box<dyn Trait>` 等效于 `Box<dyn Trait + 'static>` ;
2.  `&'x Box<dyn Trait>` 等效于 `&'x Box<dyn Trait + 'static>`;
3.  `&'a dyn Foo` 等效于 `&'a (dyn Foo + 'a)` ;
    -   因为 &amp;'a T 隐式声明了 T: 'a;
    -   &amp;'a (dyn Foo + 'a) 表示 dyn Foo 的 trait object 的 lifetime 要比 'a 长，同时它的引用也要比'a
        长；
4.  &amp;'r Ref&lt;'q, dyn Trait&gt; 等效于 &amp;'r Ref&lt;'q, dyn Trait+'q&gt;;

<https://doc.rust-lang.org/reference/lifetime-elision.html#default-trait-object-lifetimes>

```rust
// For the following trait...
trait Foo { }

// These two are the same because Box<T> has no lifetime bound on T
type T1 = Box<dyn Foo>;
type T2 = Box<dyn Foo + 'static>;

// ...and so are these:
impl dyn Foo {}
impl dyn Foo + 'static {}

// ...so are these, because &'a T requires T: 'a
type T3<'a> = &'a dyn Foo;
type T4<'a> = &'a (dyn Foo + 'a);

// std::cell::Ref<'a, T> also requires T: 'a, so these are the same
type T5<'a> = std::cell::Ref<'a, dyn Foo>;
type T6<'a> = std::cell::Ref<'a, dyn Foo + 'a>;

// This is an example of an error.
struct TwoBounds<'a, 'b, T: ?Sized + 'a + 'b> {
    f1: &'a i32,
    f2: &'b i32,
    f3: T,
}
type T7<'a, 'b> = TwoBounds<'a, 'b, dyn Foo>;
//                                  ^^^^^^^
// Error: the lifetime bound for this object type cannot be deduced from context


// For the following trait...
trait Bar<'a>: 'a { }
// ...these two are the same:
type T1<'a> = Box<dyn Bar<'a>>;
type T2<'a> = Box<dyn Bar<'a> + 'a>;
// ...and so are these:
impl<'a> dyn Bar<'a> {}
impl<'a> dyn Bar<'a> + 'a {}
```

理解： Box&lt;dyn Trait + 'static&gt;

```rust
// 表示 Parent 的生命周期可以任意长，但是当 parent 被 drop 时，child 也会被 drop，从而释放 dyn
// OtherTrait 对象。
struct Parent {
   child: Box<dyn OtherTrait + 'static>,
}
```

One thing to realize is that lifetimes in Rust are never talking about the lifetime of a value,
instead they are `upper bounds on how long a value can live`. If a type is annotated with a lifetime,
then it `must go out of scope before that lifetime ends`, but there's no requirement that it lives for
the entire lifetime.

trait object 和泛型参数的差别：

1.  对于使用 Trait bound 的泛型参数，在编译阶段 Rust 编译器是可以推断出一种实际类型，所以不能实例化出一类实现 trait 的类型。例如 Vec&lt;T: Display&gt; , 在编译器间实例化后只能保存一种实现 Display trait 的实例；
2.  对于 trait object，在编译期间是不能实例化为单一类型，而是在运行时来根据调用方法的对象指针（类型）来查找编译期间构造的 vtab 来调用对应对象的方法。所以 Vec&lt;&amp;dyn Display&gt; 是可以保存各种实现了
    Display trait 的多种对象。

使用 Box 定义 trait object 时, 一般有三种标准格式, rust 标准库为这三种格式定义了对应的方法和函数:

-   Box&lt;dyn traitName + 'static&gt;
-   Box&lt;dyn traitName + Send + 'static&gt;
-   Box&lt;dyn traitName + Sync + Send + 'static&gt;

当参数类型是 &amp;Box&lt;dyn Trait&gt; 时，不能传入 &amp;Box::new(my_struct)， Rust 报错类型不匹配，两个解决办法：

-   将参数类型修改为 Box&lt;dyn Trait&gt;, 后续可以直接传入 Box::new(my_struct)；
-   或者，先创建一个 Box&lt;dyn Triat&gt; 的对象 b，然后再传入 &amp;b;

<!--listend-->

-   traitName 还可以是闭包函数 trait, 例如: Box&lt;dyn Fn(int32) -&gt; int32&gt;;

<!--listend-->

```rust
struct Sheep {}
struct Cow {}

trait Animal {
    // Instance method signature
    fn noise(&self) -> &'static str;
}

// Implement the `Animal` trait for `Sheep`.
impl Animal for Sheep {
    fn noise(&self) -> &'static str {
        "baaaaah!"
    }
}

// Implement the `Animal` trait for `Cow`.
impl Animal for Cow {
    fn noise(&self) -> &'static str {
        "moooooo!"
    }
}

// Returns some struct that implements Animal, but we don't know which one at compile time.
fn random_animal(random_number: f64) -> Box<dyn Animal> {
    if random_number < 0.5 {
        Box::new(Sheep {})
    } else {
        Box::new(Cow {})
    }
}
fn main() {
    let random_number = 0.234;
    let animal = random_animal(random_number);
    println!("You've randomly chosen an animal, and it says {}", animal.noise());
}

trait MyTrait {
    fn say_hello(&self);
}

struct MyStruct(String);

impl MyTrait for MyStruct {
    fn say_hello(&self) {
        println!("hello {}", self.0);
    }
}

fn main() {
    // printf_hellov2(Some(&Box::new(my_struct))); // error[E0308]: mismatched types，expected `&Box<dyn MyTrait + Send>`, found `&Box<MyStruct>`
    // 解决办法：先创建一个 Box 对象，然后再借用。
    let st1: Box<dyn MyTrait + Send + 'static> = Box::new(my_struct);
    printf_hellov2(Some(&st1));
}

fn printf_hellov2(say_hello: Option<&Box<dyn MyTrait + Send + 'static>>) {
    // if let Some( &my_trait) = say_hello { // error[E0507]: cannot move out of `*say_hello` as enum variant `Some` which is behind a shared reference
    if let Some(my_trait) = say_hello {
        my_trait.say_hello();
    }
}
```

当将一个对象被转移给函数时，该对象满足函数的 'static 要求（因为转移给函数后，函数可以自由决定对象的生命周期）：

-   如果对象有借用，则不能转移给函数，

<!--listend-->

```rust
use std::collections::HashMap;

trait App {}

struct AppRegistry {
    apps: HashMap<String, Box<dyn App>>,
}

impl AppRegistry {
    fn new() -> AppRegistry {
        Self { apps: HashMap::new() }
    }

    fn add(&mut self, name: &str, app: impl App + 'static) {
        self.apps.insert(name.to_string(), Box::new(app));
    }
}

struct MyApp {}

impl App for MyApp {}

fn main() {
    let mut registry = AppRegistry::new();
    registry.add("thing", MyApp {}); // paas by value 传递对象，这时 add 方法相当于拥有了这个对象，所以满足 'static
    println!("apps: {}", registry.apps.len());
}
```

On-Stack Dynamic Dispatch: We can dynamically dispatch over multiple values, however, to do so, we
need to declare multiple variables to bind differently-typed objects. `To extend the lifetime as
necessary` , we can use deferred conditional initialization, as seen below:

```rust
// https://rust-unofficial.github.io/patterns/idioms/on-stack-dyn-dispatch.html

use std::io;
use std::fs;

// These must live longer than `readable`, and thus are declared first:
let (mut stdin_read, mut file_read);

// We need to describe the type to get dynamic dispatch.
let readable: &mut dyn io::Read = if arg == "-" {
    stdin_read = io::stdin();
    &mut stdin_read
} else {
    file_read = fs::File::open(arg)?;
    &mut file_read
};

// Read from `readable` here.

// 使用 Box 更简洁，Box 拥有了 dyn io::Read, 它的声明周期和它一致。
// We still need to ascribe the type for dynamic dispatch.
let readable: Box<dyn io::Read> = if arg == "-" {
    Box::new(io::stdin())
} else {
    Box::new(fs::File::open(arg)?)
};
// Read from `readable` here.
```


### <span class="section-num">18.6</span> impl trait {#impl-trait}

`impl TraitName（注意：impl 前能加 &）` 和泛型参数类似, 也是在编译时实例化为一种特定类型, 不支持运行时动态派发。impl trait 可以作为泛型参数 Bound、函数输入参数和返回值类型；

```rust
// impl trait 作为函数参数的类型时，不需要引入泛型函数。
fn print_it1(input: impl Debug + 'static ) {
    println!( "'static value passed in is: {:?}", input );
}
// 等效为
fn print_it<T: Debug + 'static>(input: T) {
    println!( "'static value passed in is: {:?}", input );
}

// 另一个例子
// 泛型参数
fn parse_csv_document<R: std::io::BufRead>(src: R) -> std::io::Result<Vec<Vec<String>>> {}
// impl Trait 省去泛型参数
fn parse_csv_document(src: impl std::io::BufRead) -> std::io::Result<Vec<Vec<String>>> {}

// 返回值类型的例子，可以简化值类型声明
use std::iter;
use std::vec::IntoIter;
// This function combines two `Vec<i32>` and returns an iterator over it.  Look how complicated its
// return type is!
fn combine_vecs_explicit_return_type(
    v: Vec<i32>,
    u: Vec<i32>,
) -> iter::Cycle<iter::Chain<IntoIter<i32>, IntoIter<i32>>> {
    v.into_iter().chain(u.into_iter()).cycle()
}
// This is the exact same function, but its return type uses `impl Trait`.  Look how much simpler it
// is!
fn combine_vecs(
    v: Vec<i32>,
    u: Vec<i32>,
) -> impl Iterator<Item=i32> {
    v.into_iter().chain(u.into_iter()).cycle()
}
fn main() {
    let v1 = vec![1, 2, 3];
    let v2 = vec![4, 5];
    let mut v3 = combine_vecs(v1, v2);
    assert_eq!(Some(1), v3.next());
    assert_eq!(Some(2), v3.next());
    assert_eq!(Some(3), v3.next());
    assert_eq!(Some(4), v3.next());
    assert_eq!(Some(5), v3.next());
    println!("all done");
}
```

由于闭包 Fn 也是 trait，所以函数可以返回 Box&lt;dyn Fn() -&gt; i32&gt; 或 impl Fn() -&gt; i32:

```rust
// Returns a function that adds `y` to its input
fn make_adder_function(y: i32) -> impl Fn(i32) -> i32 {
    let closure = move |x: i32| { x + y };
    closure
}

fn main() {
    let plus_one = make_adder_function(1);
    assert_eq!(plus_one(2), 3);
}

fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1) // 堆分配空间，性能差些
}

fn returns_closure() -> impl Fn(i32) -> i32 {
    |x| x + 1 // 栈分配，性能好
}
```


### <span class="section-num">18.7</span> 运算符重载 {#运算符重载}

类型实现 std::ops 下的各 trait， 以实现运算符重载：

```rust
use std::ops;

struct Foo;
struct Bar;

#[derive(Debug)]
struct FooBar;

#[derive(Debug)]
struct BarFoo;

// The `std::ops::Add` trait is used to specify the functionality of `+`.  Here, we make `Add<Bar>`
// - the trait for addition with a RHS of type `Bar`.  The following block implements the operation:
// Foo + Bar = FooBar
impl ops::Add<Bar> for Foo {
    type Output = FooBar;

    fn add(self, _rhs: Bar) -> FooBar {
        println!("> Foo.add(Bar) was called");
        FooBar
    }
}
// By reversing the types, we end up implementing non-commutative addition.  Here, we make
// `Add<Foo>` - the trait for addition with a RHS of type `Foo`.  This block implements the
// operation: Bar + Foo = BarFoo
impl ops::Add<Foo> for Bar {
    type Output = BarFoo;

    fn add(self, _rhs: Foo) -> BarFoo {
        println!("> Bar.add(Foo) was called");

        BarFoo
    }
}

fn main() {
    println!("Foo + Bar = {:?}", Foo + Bar);
    println!("Bar + Foo = {:?}", Bar + Foo);
}
```


### <span class="section-num">18.8</span> PhantomData {#phantomdata}

A phantom type parameter is one that doesn't show up at runtime, but is `checked statically (and
only) at compile time` .

-   PhantomData 是一个 zero-sized 泛型 struct, 唯一的一个值是 PhantomData;

<!--listend-->

```rust
pub struct PhantomData<T> where T: ?Sized;
```

使用举例:

1.  lifetime:

<!--listend-->

```rust
  // Error:  没有 field 使用 'a
  struct Slice<'a, T> {
      start: *const T,
      end: *const T,
  }

  // OK: 用 0 长字段来使用 'a
  use std::marker::PhantomData;

  struct Slice<'a, T> {
      start: *const T,
      end: *const T,
      phantom: PhantomData<&'a T>, // 表示 T: 'a, 即 T 对象的引用的生命周期要比 'a 长。
  }
```

1.  隐藏数据:

<!--listend-->

```rust
use std::marker::PhantomData;

// A phantom tuple struct which is generic over `A` with hidden parameter `B`.
#[derive(PartialEq)] // Allow equality test for this type.
struct PhantomTuple<A, B>(A, PhantomData<B>);

// A phantom type struct which is generic over `A` with hidden parameter `B`.
#[derive(PartialEq)] // Allow equality test for this type.
struct PhantomStruct<A, B> { first: A, phantom: PhantomData<B> }

// Note: Storage is allocated for generic type `A`, but not for `B`.
//       Therefore, `B` cannot be used in computations.

fn main() {
    // Here, `f32` and `f64` are the hidden parameters.
    // PhantomTuple type specified as `<char, f32>`.
    let _tuple1: PhantomTuple<char, f32> = PhantomTuple('Q', PhantomData);
    // PhantomTuple type specified as `<char, f64>`.
    let _tuple2: PhantomTuple<char, f64> = PhantomTuple('Q', PhantomData);

    // Type specified as `<char, f32>`.
    let _struct1: PhantomStruct<char, f32> = PhantomStruct { first: 'Q', phantom: PhantomData, };
    // Type specified as `<char, f64>`.
    let _struct2: PhantomStruct<char, f64> = PhantomStruct { first: 'Q', phantom: PhantomData, };

    // Compile-time Error! Type mismatch so these cannot be compared:
    // println!("_tuple1 == _tuple2 yields: {}",
    //           _tuple1 == _tuple2);

    // Compile-time Error! Type mismatch so these cannot be compared:
    // println!("_struct1 == _struct2 yields: {}",
    //           _struct1 == _struct2);
}
```

1.  编译时类型检查:

<!--listend-->

```rust
use std::ops::Add;
use std::marker::PhantomData;

/// Create void enumerations to define unit types.
#[derive(Debug, Clone, Copy)]
enum Inch {}
#[derive(Debug, Clone, Copy)]
enum Mm {}

/// `Length` is a type with phantom type parameter `Unit`,
/// and is not generic over the length type (that is `f64`).
///
/// `f64` already implements the `Clone` and `Copy` traits.
#[derive(Debug, Clone, Copy)]
struct Length<Unit>(f64, PhantomData<Unit>);

/// The `Add` trait defines the behavior of the `+` operator.
impl<Unit> Add for Length<Unit> {
    type Output = Length<Unit>;

    // add() returns a new `Length` struct containing the sum.
    fn add(self, rhs: Length<Unit>) -> Length<Unit> {
        // `+` calls the `Add` implementation for `f64`.
        Length(self.0 + rhs.0, PhantomData)
    }
}

fn main() {
    // Specifies `one_foot` to have phantom type parameter `Inch`.
    let one_foot:  Length<Inch> = Length(12.0, PhantomData);
    // `one_meter` has phantom type parameter `Mm`.
    let one_meter: Length<Mm>   = Length(1000.0, PhantomData);

    // `+` calls the `add()` method we implemented for `Length<Unit>`.
    //
    // Since `Length` implements `Copy`, `add()` does not consume
    // `one_foot` and `one_meter` but copies them into `self` and `rhs`.
    let two_feet = one_foot + one_foot;
    let two_meters = one_meter + one_meter;

    // Addition works.
    println!("one foot + one_foot = {:?} in", two_feet.0);
    println!("one meter + one_meter = {:?} mm", two_meters.0);

    // Nonsensical operations fail as they should:
    // Compile-time Error: type mismatch.
    //let one_feter = one_foot + one_meter;
}
```


### <span class="section-num">18.9</span> Copy/Clone {#copy-clone}

Copy 是 Clone 的 子 trait, 所以在实现 Copy 的同时也必须实现 Clone. 另外 Copy 是一个 marker trait, 不需要实现特定的方法.

```rust
pub trait Copy: Clone { }
```

Clone 和 Copy 的差别:

1.  Copy 是 rustc 隐式调用的(如变量赋值,传参等), 不能被重载或控制, 做的是 simple bit-wise copy.
2.  Clone 是需要明确调用的方法, 如 x.clone(), 在具体实现时, 可能会 copy the point-to buffer in the
    heap, 而不向 Copy 那样可能值会做栈空间的拷贝(如指针拷贝).

默认实现 Copy 的情况:

1.  基本类型, 如整型, 浮点型, bool, char,
2.  Option&lt;T&gt;, Result&lt;T,E&gt;, Bound&lt;T&gt;, 共享引用 &amp;T, 如 &amp;str;
3.  数组 [T;N],  元组 (T, T): 前提: 元素类型 T 实现了 Copy;

没有实现 Copy 的情况:

1.  自定义的 Struct/Enum;
2.  std::collections 中的各种容器类型, 如 Vec/HashMap/HashSet; (但都实现了 Clone)
3.  &amp;mut T 没有实现 Copy, 但是 &amp;T 实现了 Copy.
4.  任何实现了 Drop 的对象也没有实现 Copy.
5.  各种智能指针，如 Box/Rc/Arc/Ref 都没有实现 Copy。

实现 Copy 方式: derive(rustc 自动实现) 或手动实现.

```rust
// OK: i32 实现了 Copy
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

// 错误: Vec<T> 没有实现 Copy
// the trait `Copy` cannot be implemented for this type; field `points` does not implement `Copy`
#[derive(Copy, Clone)]
struct PointList {
    points: Vec<Point>,
}

// OK: 因为 &T 实现了 Copy, 即使 T 没有实现 Copy, &T 也实现了 Copy
#[derive(Copy, Clone)]
struct PointListWrapper<'a> {
    point_list_ref: &'a PointList,
}

// 或者
struct MyStruct;

impl Copy for MyStruct { }

impl Clone for MyStruct {
    fn clone(&self) -> MyStruct {
        *self
    }
}
```


### <span class="section-num">18.10</span> Default {#default}

<https://doc.rust-lang.org/stable/std/default/trait.Default.html>

按照惯例, 各类型使用 new() 关联函数来作为类型的 constructor. Rust 并没有强制所有类型都实现该方法, 但是它提供了 Default trait, 各类型可以实现该 trait 提供的 default() -&gt; Self 方法:

```rust
/// Time in seconds.
///
/// # Example
///
/// ```
/// let s = Second::default();
/// assert_eq!(0, s.value());
/// ```
pub struct Second {
    value: u64,
}

impl Second {
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}

impl Default for Second {
    fn default() -> Self {
        Self { value: 0 }
    }
}
```

Default can also be derived if all types of all fields implement Default, like they do with Second:

```rust
use std::{path::PathBuf, time::Duration};

// note that we can simply auto-derive Default here.
#[derive(Default, Debug, PartialEq)]
struct MyConfiguration {
    // Option defaults to None
    output: Option<PathBuf>,
    // Vecs default to empty vector
    search_path: Vec<PathBuf>,
    // Duration defaults to zero time
    timeout: Duration,
    // bool defaults to false
    check: bool,
}

impl MyConfiguration {
    // add setters here
}

fn main() {
    // construct a new instance with default values
    let mut conf = MyConfiguration::default(); // 调用 Default trait 的 default() 方法来创建对象.
    // do something with conf here
    conf.check = true;
    println!("conf = {conf:#?}");

    // partial initialization with default values, creates the same instance
    let conf1 = MyConfiguration {
        check: true,
        ..Default::default()  // struct 对象部分输出化, 没有指定的 field 都使用 default() 方法的值来填充.
    };
    assert_eq!(conf, conf1);
}

// 另一个例子
#[derive(Default)]
struct SomeOptions {
    foo: i32,
    bar: f32,
}

fn main() {
    let options: SomeOptions = Default::default();
    // 自动根据目标类型调用对应实现的方法, 等效于 <SomeOptions as Default>::default()
}
```

对于 enum 类型, 需要明确定义哪一个是 default:

```rust
#[derive(Default)]
enum Kind {
    #[default]
    A,
    B,
    C,
}

// 或者
enum Kind {
    A,
    B,
    C,
}

impl Default for Kind {
    fn default() -> Self { Kind::A }
}
```

rust 基本类型和 &amp;str, &amp;CStr, &amp;OsStr, Box&lt;str&gt;, Box&lt;CStr&gt;, Box&lt;OsStr&gt;, CString, OsString, Error,
PathBuf, String, &amp;[T], Option&lt;T&gt;, [T; N], Rc&lt;T&gt;, Arc&lt;T&gt;, Vec&lt;T&gt;, HashSet&lt;T, S&gt; 等都实现了 Default.

Default trait 作为泛型类型的定界, 在标准库中得到广泛使用, 例如:

1.  很多 Option::unwrap_or_default() 方法;
2.  std::mem::take(&amp;mut T) 将 T 的值 move 出来，然后将原始的 T 的值用他的 Default 填充；

<!--listend-->

```rust
let x: Option<u32> = None;
let y: Option<u32> = Some(12);

assert_eq!(x.unwrap_or_default(), 0);
assert_eq!(y.unwrap_or_default(), 12);
```


### <span class="section-num">18.11</span> Drop {#drop}

Drop trait 为对象提供了自定义析构能力， 一般在对象离开 scope 时由编译器自动调用来释放资源：

-   例如：Box, Vec, String, File, and Process 都实现了 Drop；
-   也可以使用的 drop(obj) 或 std::mem::drop/forget 来手动释放对象；

<!--listend-->

```rust
  fn bar() -> Result<(), ()> {
      // These don't need to be defined inside the function.
      struct Foo;

      // Implement a destructor for Foo.
      impl Drop for Foo {
          fn drop(&mut self) {
              println!("exit");
          }
      }

      // The dtor of _exit will run however the function `bar` is exited.
      let _exit = Foo;
      // Implicit return with `?` operator.
      baz()?;
      // Normal return.
      Ok(())
  }
```

Drop 在程序 panic 时也会被执行: Code in destructors will (nearly) always be run - copes with panics,
early returns, etc.


### <span class="section-num">18.12</span> From/Into： {#from-into}

自定义类型一般只需实现 From，rust 会自动生成相反方向的 Into。

```rust
use std::convert::From;

#[derive(Debug)]
struct Number {
    value: i32,
}

impl From<i32> for Number {
    fn from(item: i32) -> Self {
        Number { value: item }
    }
}

fn main() {
    let num = Number::from(30);
    println!("My number is {:?}", num);
}
```

Into 一般作用在泛型函数的限界场景中，例如：

```rust
pub fn stdout<T: Into<Stdio>>(&mut self, cfg: T) -> &mut Command
```

Rust 中实现了 \`From\` trait 的类型，会在以下场景自动调用：

1.  `Into trait` 规定了任何实现了 `From<T>` 的类型，自动实现 `Into<U>` 。

<!--listend-->

```rust
let from_type: FromType = FromType::new();
let into_type: IntoType = from_type.into(); // 自动调用 From<FromType> for IntoType 的实现
```

1.  使用 \`?\` 运算符处理 \`Result\` 或 \`Option\` 错误在返回错误时，将错误类型转换为函数返回的错误类型。

<!--listend-->

```rust
fn from_type() -> Result<IntoType, IntoError> {
    let result: Result<FromType, FromError> = do_something_may_fail();
    let from_type = result?; // 如果出错，自动调用 From<FromError> for IntoError 的实现并返回
}
```

the standard library gives us `a blanket From impl` for converting from anything that `implements Error
to a Box<dyn Error>, which ? automatically uses` :

```rust
impl<'a, E: Error + Send + Sync + 'a> From<E> for Box<dyn Error + Send + Sync + 'a> {
    fn from(err: E) -> Box<dyn Error + Send + Sync + 'a> {
        Box::new(err)
    }
}

type GenericError = Box<dyn std::error::Error + Send + Sync + 'static>;
type GenericResult<T> = Result<T, GenericError>;
fn parse_i32_bytes(b: &[u8]) -> GenericResult<i32> {
    Ok(std::str::from_utf8(b)?.parse::<i32>()?)
}
```

1.  \`collect\` 方法将 \`Iterator&lt;Item=T&gt;\` 转换为其他容器类型。

<!--listend-->

```rust
let vec: Vec<FromType> = get_some_from_types();
let result: Result<Vec<IntoType>, _> = vec.into_iter().collect(); // 自动调用 From<FromType> for IntoType 的实现
```

1.  \`std::convert::from\` 函数会显式调用 \`From\` trait。

<!--listend-->

```rust
let from_type = FromType::new();
let into_type:IntoType = std::convert::From::from(from_type); // 显式调用 From<FromType> for IntoType 的实现
```

变量赋值和函数返回值: 如 &amp;str 实现了 Into Box&lt;dyn std::error::Error&gt; 的 trait, 则可以直接调用 into()
 方法:

-   如果 impl From&lt;T&gt; for U, 则可以 let u: U = U::from(T) 或 let u:U = T.into().

<!--listend-->

```rust
fn main() {
    if let Err(e) = run_app() {
        eprintln!("Application error: {}", e);
        std::process::exit(1);
    }
}
fn run_app() -> Result<(), Box<dyn std::error::Error>> {
    // 代码逻辑
    // Ok(()) 执行这行将会正常返回0
    return Err("main will return 1".into());
}

trait Into<t>: Sized {
    fn into(self) -> T; // ownership 方法，将之声转换为 T 对象
}

trait From<T>: Sized {
    fn from(other: T) -> Self; // 关联函数，ownership 传入的 T 对象
}
use std::net::Ipv4Addr;
fn ping<A>(address: A) -> std::io::Result<bool> where A: Into<Ipv4Addr> // A 是任意能转换为 Ipv4Addr 的类型
{
    let ipv4_address = address.into();
    // ...
}
```

其他情况下，rust 并不会自动调用 From/Into trait：

Unlike `From/Into`, `TryFrom` and `TryInto` are used for fallible conversions and return a `Result` instead
of a plain value.


### <span class="section-num">18.13</span> FromStr/ToString {#fromstr-tostring}

从 string 来生成各种类型的值：

-   一般被 &amp;str.parse::&lt;T&gt;() 方法隐式调用。
-   rust 的基本类型，如整数、浮点数、bool、char、String、PathBuf、IpAddr、SocketAddr、Ipv4Addr、
    Ipv6Addr 都实现了该 trait。

<!--listend-->

```rust
pub trait FromStr: Sized {
    type Err;

    // Required method
    fn from_str(s: &str) -> Result<Self, Self::Err>;
}
```

str 的 parse::&lt;T&gt;() 方法用于将字符串转换为其它对象：

```rust
  pub fn parse<F>(&self) -> Result<F, <F as FromStr>::Err>
  where
      F: FromStr,
```

在使用 &amp;str.parse() 方法时一般需要指定目标对象类型，否则 rustc 不支持该调用哪个类型的 FromStr 实现：

-   注意：只支持目标类型本身，而不是它的 &amp;T 或 &amp;mut T；

<!--listend-->

```rust
let four: u32 = "4".parse().unwrap();
assert_eq!(4, four);

let four = "4".parse::<u32>();
assert_eq!(Ok(4), four);

// Error
let nope = "j".parse::<u32>();
assert!(nope.is_err());
```

如果类型实现了 ToString trait，则可以用它生成一个 String。rust 对于任意实现了 Display trait 的类型自动实现了 ToString trait。

```rust
use std::fmt;

struct Circle {
    radius: i32
}

impl fmt::Display for Circle {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Circle of radius {}", self.radius)
    }
}

fn main() {
    let circle = Circle { radius: 6 };
    println!("{}", circle.to_string());
}
```


### <span class="section-num">18.14</span> FromIterator/IntoIterator {#fromiterator-intoiterator}

FromIterator: 从一个实现了 IntoIterator&lt;Item=A&gt; 的迭代器创建一个 Self 类型对象：

```rust
pub trait FromIterator<A>: Sized {
    // Required method
    fn from_iter<T>(iter: T) -> Self
       where T: IntoIterator<Item = A>;
}
```

如果一个类型 B 实现了 FromIterator&lt;Iterm=A&gt;， 则可以从能迭代生产 A 的迭代器生产 B。FromIterator 的典型使用场景是 IntoIterator 的 collect::&lt;TargetType&gt;() 方法，用于将前序迭代器对象的值生产一个
TargetType。

IntoIterator 是 for item in type 迭代语句对 type 的要求，即 type 要实现 IntoIterator 才能被 for 迭代：

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;

    // Required method
    fn into_iter(self) -> Self::IntoIter;
}
```


### <span class="section-num">18.15</span> Try {#try}

? 运算法可以用于 Result/Option, 它可以使用 std::ops::Try trait 来自定义:

-   Try trait 用于自定义 The ? operator and try {} blocks.

<!--listend-->

```rust
pub trait Try: FromResidual {
    type Output;
    type Residual;

    // Required methods
    fn from_output(output: Self::Output) -> Self;
    fn branch(self) -> ControlFlow<Self::Residual, Self::Output>;
}
```


### <span class="section-num">18.16</span> AsRef/AsMut {#asref-asmut}

例如 std::fs:<:open> 函数的声明：open 的实现依赖于 &amp;Path，通过限定 P 实现了 AsRef&lt;Path&gt;，在 open
内部就可以通过 P.as_ref() 方法调用返回 &amp;Path 的类型对象。

```rust
fn open<P: AsRef<Path>>(path: P) -> Result<file>
```

标准库各种类型，如 String/str/OsString/OsStr/PathBuf/Path 等都实现了 AsRef&lt;Path&gt;, 所以都可以作为
open 的参数输入：

```rust
// https://doc.rust-lang.org/src/std/path.rs.html#3170
impl AsRef<Path> for String {
    #[inline]
    fn as_ref(&self) -> &Path {
        Path::new(self)
    }
}
```


### <span class="section-num">18.17</span> Index/IndexMut {#index-indexmut}

a 可以实现 Index trait 和 IndexMutt trait, 前者的 index() 返回 &amp;Self::Output, 后者的 index_mut() 返回 &amp;mut Self::Output;

a[i] 是 \*a.index(i) 的简写形式，它返回一个对象而非引用，这是由于 index() 方法返回的是对象的不可变引用。返回一个对象的好处是，可以作为左值使用，例如 a[i] = 3;

rust 根据 a[xx] 操作中的 xx 类型，自动生成对应的 RangeXX struct 类型：

-   Range: m..n
-   RangeFrom: m..
-   RangeFull: ..
-   RangeInclusive: m..=n
-   RangeTo: ..n
-   RangeToInclusive: ..=n

然后创建一个 SliceIndex&lt;str&gt; 或 SliceIndex&lt;T&gt; 类型，调用他的 index() 方法返回 或 &amp;str 或 &amp;T 对象，再解引用获得 &amp;str 或 T 对象。

rust 编译器根据 a[i] 索引表达式出现的上下文来自动选择 index() 或 index_mut() 方法。

对 &amp;[T] 使用 index 操作时，返回的是 &amp;[T] 类型。

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<I> ops::Index<I> for str
where
    I: SliceIndex<str>,
{
    type Output = I::Output;

    #[inline]
    fn index(&self, index: I) -> &I::Output {
        index.index(self)
    }
}

// 然后查看那些类型实现了 SliceIndex<str>，然后获得他的 Output 关联类型。在 SliceIndex 的文档页面的
// Implementors 部分可以看到实现 SliceIndex<str> 的类型列表：
// std/slice/trait.SliceIndex.html#implementors
impl SliceIndex<str> for (Bound<usize>, Bound<usize>)
type Output = str

impl SliceIndex<str> for Range<usize>
type Output = str

impl SliceIndex<str> for RangeFrom<usize>
type Output = str

impl SliceIndex<str> for RangeFull
type Output = str

impl SliceIndex<str> for RangeInclusive<usize>
type Output = str

impl SliceIndex<str> for RangeTo<usize>
type Output = str

impl SliceIndex<str> for RangeToInclusive<usize>
type Output = str
```


### <span class="section-num">18.18</span> Borrow/ToOwned/Cow {#borrow-toowned-cow}

Borrow 和 BorrowMut 和 AsRef/AsMut 类似，Borrow&lt;T&gt; 是从自身创建一个 &amp;T 的借用，但是它要求 &amp;T 必须和
Self 能以相同的方式进行哈希和比较时，Self 才应该实现 Borrow&lt;T&gt;

-   Rust 编译器并不会强制该限制，但是 Borrow 有这种约定的意图。

String 实现了 AsRef&lt;str&gt;, AsRef&lt;[u8]&gt;, AsRef&lt;Path&gt;, 但是只有 String 和 &amp;str 才能保证做相同的 hash，所以 String 只实现按了 Borrow&lt;str&gt;.

Borrow 用于解决泛型哈希表和其他关联集合类型的情况：

-   K 和 Q 都是 Eq + Hash 语义；
-   K: Borrow&lt;Q&gt; 表示可以从 K 对象生成 &amp;Q 的引用。
-   所以可以给 HashMap 的 get() 方法传入任何满足上面两个约束的引用对象，在 get() 方法的内部实现中，会从自身的 K.borrow() 生成 &amp;Q 对象，然后用 K 的 Eq+Hash 值于 Q 的 Eq+Hash 值进行比较。

<!--listend-->

```rust
impl<K,V> HashMap<K, V> where K: Eq + Hash
{
  fn get<Q: ?Sized>(&self, key: &Q) -> Option<&V>
    where K: Borrow<Q>,
          Q: Eq + Hash
// ...
}
```

例如： String 实现了 Borrow&lt;str&gt; 和 Borrow&lt;String&gt;, 所以 HashMap&lt;String, i32&gt; 的 get 方法可以传入
&amp;String 或 &amp;str 类型的 key。

Borrow/AsRef trait: 都是将 &amp;self 转换为 `其他类型` 的 ref:

-   self 一般是 owned 类型, 如 String/PathBuf/OsString/Box/Arc/Rc 等, borrow 的结果是对应的 unsized 类型, 如 &amp;str/&amp;Path/&amp;OsStr/&amp;T;

<!--listend-->

```rust
pub trait AsRef<T>
where
    T: ?Sized,
{
    // Required method
    fn as_ref(&self) -> &T;
}

pub trait Borrow<Borrowed>
where
    Borrowed: ?Sized,
{
    // Required method
    fn borrow(&self) -> &Borrowed;
}
```

相比 AsRef, Borrow trait 主要是增加了 Eq/Ord/Hash 的限制, 也就是 borrowed 生成的 value 和 owned
value 的 Eq/Ord/Hash 语义都是一致的:

-   String 实现了 Borrow&lt;str&gt;, 所以 String 和 str 的 Eq/Ord/Hash 的语义是一致的.

<!--listend-->

```rust
use std::borrow::Borrow;
use std::hash::Hash;

pub struct HashMap<K, V> {
    // fields omitted
}

impl<K, V> HashMap<K, V> {
    pub fn insert(&self, key: K, value: V) -> Option<V>
    where K: Hash + Eq
    {
        // ...
    }

    pub fn get<Q>(&self, k: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq + ?Sized
    {
        // ...
    }
}
```

实现 Borrow 的类型如下:

```rust
// 从 Ownerd 类型生成 unsized 的类型引用
impl Borrow<str> for String  // &String -> &str
impl Borrow<CStr> for CString // &CString -> &CStr
impl Borrow<OsStr> for OsString
impl Borrow<Path> for PathBuf

impl<'a, B> Borrow<B> for Cow<'a, B>
where
    B: ToOwned + ?Sized,

impl<T> Borrow<T> for &T
where
    T: ?Sized,

impl<T> Borrow<T> for &mut T
where
    T: ?Sized,

impl<T> Borrow<T> for T
where
    T: ?Sized,

impl<T, A> Borrow<[T]> for Vec<T, A>  // &Vec<T>  -> &[T]
where
    A: Allocator,

impl<T, A> Borrow<T> for Box<T, A> // &Box<T> -> &T
where
    A: Allocator,
    T: ?Sized,

impl<T, A> Borrow<T> for Rc<T, A> // &Rc<T> -> &T
where
    A: Allocator,
    T: ?Sized,

impl<T, A> Borrow<T> for Arc<T, A> // &Arc<T> -> &T
where
    A: Allocator,
    T: ?Sized,

impl<T, const N: usize> Borrow<[T]> for [T; N]  // &[T;N] -> &[T]
```

反过来, 由 Borrow trait 创建的类型,如 str, 也实现了 ToOwned trait, 将 &amp;T 转换为 Owned 类型, 如:

-   &amp;str -&gt; String
-   &amp;CStr -&gt; CString
-   &amp;OsStr -&gt; OsString
-   &amp;Path -&gt; PathBuf
-   &amp;[T] -&gt; Vec&lt;T&gt;
-   &amp;T -&gt; T

<!--listend-->

```rust
pub trait ToOwned {
    type Owned: Borrow<Self>; // Ownerd 是实现了 Borrow<Self> 的任意类型

    // Required method
    fn to_owned(&self) -> Self::Owned;

    // Provided method
    fn clone_into(&self, target: &mut Self::Owned) { ... }
}

// 实现 ToOwned 的类型都是 unsized 类型, 它们的 Owned 类型都是 sized 版本
impl ToOwned for str type Owned = String
impl ToOwned for CStr type Owned = CString
impl ToOwned for OsStr type Owned = OsString
impl ToOwned for Path type Owned = PathBuf
impl<T> ToOwned for [T] where T: Clone, type Owned = Vec<T>
impl<T> ToOwned for T where T: Clone, type Owned = T
```

示例:

```rust
let s: &str = "a";
let ss: String = s.to_owned();

let v: &[i32] = &[1, 2];
let vv: Vec<i32> = v.to_owned();
```

Clone 和 ToOwned 的区别:

1.  相同点: 两者都可以从 &amp;self -&gt; Self, 也即 &amp;T -&gt; T;
2.  不同点:  Clone trait 只能实现 &amp;T -&gt; T 的转换, 而 ToOwned trait 可以更灵活, 转换后的对象不局限于 T.
    -   只要 Owned 类型实现了 Borrow&lt;Self&gt; 就可以转换为该 Owned 类型. 例如 String 实现了 Borrow&lt;str&gt;, 则
        &amp;str 就可以转换为 Owned 类型为 String 的对象.

std::borrow::Cow 是一个同时可以保存 &amp;B 和 B 的 owned 类型的枚举对象:

-   &lt;B as ToOwned&gt;::Owned&gt; 的含义为 B 实现的 ToOwned trait 指定的 Owned 类型, 也就是可以生成 &amp;B 的
    owned 类型.

<!--listend-->

```rust
pub enum Cow<'a, B>
where
    B: 'a + ToOwned + ?Sized,
{
    Borrowed(&'a B), // &B 引用
    Owned(<B as ToOwned>::Owned), // 可以生成 &B 的 Owned 类型对象
}
```

`Cow<'a, B> 实现了 Deref<Target=B>`, 所以可以直接调用 &amp;B 的方法, 如果需要 mutation, 则可以调用他的
to_mut(&amp;mut self) 方法, 它返回一个 Owned 类型的 &amp;mut 引用(如果 Cow 当前保存的是 Borrowed(&amp;B), 则会自动调用 &amp;B 的 ToOwned trait 来 clone 生成 B 的 Owned 类型, 然后返回它的 &amp;mut 类型):

Cow 示例, 对于实现了 From trait 的类型, rust 自动为它生成 Into trait:

-   impl&lt;'a, T&gt; From&lt;&amp;'a [T]&gt; for Cow&lt;'a, [T]&gt; where T: Clone,
-   impl&lt;'a, T&gt; From&lt;&amp;'a Vec&lt;T&gt;&gt; for Cow&lt;'a, [T]&gt; where T: Clone,
-   impl&lt;'a, T&gt; From&lt;Vec&lt;T&gt;&gt; for Cow&lt;'a, [T]&gt; where T: Clone,

<!--listend-->

```rust
// file:///Users/zhangjun/.rustup/toolchains/nightly-x86_64-apple-darwin/share/doc/rust/html/std/borrow/enum.Cow.html#method.to_mut
use std::borrow::Cow;

// 函数参数是 Cow<'_, [i32> ], 其中可以保存 &[i32] 或他的 Owned 类型 Vec[i32] 或 [i32; N]
fn abs_all(input: &mut Cow<'_, [i32]>) {
    // Cow 实现了 Deref<Target=B>, 所以可以调用 &B 的方法.
    for i in 0..input.len() {
        let v = input[i];
        if v < 0 {
            // Clones into a vector if not already owned.
	      // 这里是通过 &[T] 实现的 ToOwned trait 来生成一个 vec[T] 来实现的.
            input.to_mut()[i] = -v;
        }
    }
}

// No clone occurs because `input` doesn't need to be mutated.
let slice = [0, 1, 2];
let mut input = Cow::from(&slice[..]); // 保存 &[i32]
abs_all(&mut input);

// Clone occurs because `input` needs to be mutated.
let slice = [-1, 0, 1];
let mut input = Cow::from(&slice[..]);
abs_all(&mut input);

// No clone occurs because `input` is already owned.
let mut input = Cow::from(vec![-1, 0, 1]); // 保存 Owned 类型 vec[i32]
abs_all(&mut input);


// Another example showing how to keep Cow in a struct:
use std::borrow::Cow;

struct Items<'a, X> where [X]: ToOwned<Owned = Vec<X>> {
    values: Cow<'a, [X]>,
}

impl<'a, X: Clone + 'a> Items<'a, X> where [X]: ToOwned<Owned = Vec<X>> {
    fn new(v: Cow<'a, [X]>) -> Self {
        Items { values: v }
    }
}

// Creates a container from borrowed values of a slice
let readonly = [1, 2];
let borrowed = Items::new((&readonly[..]).into());
match borrowed {
    Items { values: Cow::Borrowed(b) } => println!("borrowed {b:?}"),
    _ => panic!("expect borrowed value"),
}

let mut clone_on_write = borrowed;
// Mutates the data from slice into owned vec and pushes a new value on top
clone_on_write.values.to_mut().push(3);
println!("clone_on_write = {:?}", clone_on_write.values);

// The data was mutated. Let's check it out.
match clone_on_write {
    Items { values: Cow::Owned(_) } => println!("clone_on_write contains owned data"),
    _ => panic!("expect owned data"),
}
```

Cow 的重要方法:

1.  pub fn into_owned(self) -&gt; &lt;B as ToOwned&gt;::Owned   // 消耗 Cow, 生成 Owned 对象
2.  pub fn is_borrowed(&amp;self) -&gt; bool
3.  pub fn is_owned(&amp;self) -&gt; bool
4.  pub fn to_mut(&amp;mut self) -&gt; &amp;mut &lt;B as ToOwned&gt;::Owned // 返回 Owned 对象的 &amp;mut 引用;

如果 Cow 实际保存的是 Borrow&lt;B&gt; 对象, 则在调用 into_owned()/to_mut() 方法时,会调用 &amp;B 实现的 ToOwned
trait 来生成 Owned 对象, 也就是实现了 Copy on Write 的特性.

Cow 语义看成『potentially owned』，即可能拥有所有权，可以用来避免一些不必须的拷贝，一种使用场景保存函数内临时对应的引用：

```rust
fn foo(s: &str, some_condition: bool) -> &str {
    if some_condition {
        &s.replace("foo", "bar") // 临时对象的引用，在 block 结束前，临时对象被释放，引用无效
    } else {
        s
    }
}
```

上面的示例看起来没问题，但是会有编译错误：

```rust
   |         &s.replace("foo", "bar")
   |         ^-----------------------
   |         ||
   |         |temporary value created here
   |         returns a reference to data owned by the current function
```

如果把返回值改成 String，那么在 else 分支会有一次额外的拷贝，这时，Cow 就可以派上用场了：

```rust
fn foo(s: &str, some_condition: bool) -> Cow<str> {
    if some_condition {
        Cow::from(s.replace("foo", "bar")) // 保存 Owned 的 String 对象
    } else {
        Cow::from(s) // 保存 &str
    }
}
```

另一个类似的例子（playground）：

```rust
struct MyString<'a, F>(&'a str, F);

impl<'a, F> MyString<'a, F>
where
    F: Fn(&'a str),
{
    fn foo(&self, some_condition: bool) {
        if some_condition {
            (self.1)(self.0)
        } else {
            (self.1)(&self.0.replace("foo", "bar")) // 创建临时对象的引用，生命周期非 'static，所以报错
        }
    }
}
fn main() {
    // 第一个参数是字符串字面量， 'a 对应的是 'static, 所以传给闭包的 s 也应该是 'static 。
    let ss = MyString("foo", |s| println!("Results: {}", s));

    ss.foo(true);
    ss.foo(false)
}
```

在上面这个例子有，结构体的第一个属性的生命周期是 'a ，第二个属性是个闭包，参数的生命周期也是 'a ，直接编译会报下面的错误：

```rust
error[E0716]: temporary value dropped while borrowed
  --> src/main.rs:11:23
   |
3  | impl<'a, F> MyString<'a, F>
   |      -- lifetime `'a` defined here
...
11 |             (self.1)(&self.0.replace("foo", "bar"))
   |             ----------^^^^^^^^^^^^^^^^^^^^^^^^^^^^-
   |             |         |
   |             |         creates a temporary which is freed while still in use
   |             argument requires that borrow lasts for `'a` // 'a 是 'static
12 |         }
   |         - temporary value is freed at the end of this statement
```

和第一个例子的报错类似，改用 Cow 同样可以在尽量不拷贝的前提下解决这个问题：

```rust
impl<'a, F> MyString<'a, F>
where
    F: Fn(Cow<'a, str>),
{
    fn foo(&self, some_condition: bool) {
        if some_condition {
            (self.1)(Cow::from(self.0))
        } else {
            (self.1)(Cow::from(self.0.replace("foo", "bar")))
        }
    }
}
```


### <span class="section-num">18.19</span> Deref/DerefMut {#deref-derefmut}

Deref 从 &amp;self 引用返回一个 Targe 对象的不可变引用：

```rust
pub trait Deref {
    type Target: ?Sized;

    // Required method
    fn deref(&self) -> &Self::Target;
}

pub trait DerefMut: Deref {
    // Required method
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

Defer 主要使用场景是智能指针，因为智能指针类型如 t=Box&lt;U&gt; 有 \*t 和 &amp;t 的语义：

1.  \*t 返回 U，等效于 \*Deref.deref(&amp;t);
2.  &amp;t 返回 &amp;U, 等效于 Deref.deref(&amp;t);
3.  t 实现了 U 的所有非可变方法；

在需要获取对象的 &amp;mut 引用时，如方法调用或变量赋值场景，Rust 会看对象是否实现了 DerefMut trait，如果实现了则 `自动调用(被称为 mutable deref coercion) 它来实现转换` ：

`所以，*v 作为左值的场景，rust 使用 DerefMut<Target=U> 来对 * 操作符进行了重载，相当于用生成 Target U
对象来进行赋值。`

对于实现了 Deref&lt;Target=U&gt; 的类型 T 值, `&*T 返回 &U`:

-   Deref 重载了 \* 运算符, 所以 `*T == *t.deref()`, 返回 U 对象, 为了获得 &amp;U, 一般使用 `&*T`;
-   let (a, b) = &amp;v; 这时 a 和 b 都是引用类型, 正确!
-   let &amp;(a, b) = &amp;v; 这时 a 和 b 都是 move 语义, 如果对应值没有实现 copy,  则不允许从引用类型 move 值出来.

<!--listend-->

```rust
*v = 1;  // 引用变量赋值
let mut data = data.lock().unwrap(); // 变量赋值时类型转换

let mut v = xx;
v.setXX(xxx);  // 自动获得 v 的 &mut 引用来调用 setXX() 方法.


use std::sync::{Arc, Mutex, Condvar};
use std::thread;

let pair = Arc::new((Mutex::new(false), Condvar::new()));
let pair2 = Arc::clone(&pair);

// Inside of our lock, spawn a new thread, and then wait for it to start.
thread::spawn(move|| {
    // pair2 实现了 Deref trait, 会重载 * 运算符, 所以 &*pair2 等效为 &*pair2.deref();
    // 这里 &*pair2 返回的是 &(Mutex::new(false), Condvar::new()), 赋值解构后, lock 是 &Mutex, cvar
    // 是 &Convar不能使用 let &(lock, cvar) = &*pair2; 这会导致 pair2 中的值发生了移动( lock 是
    // Mutex, cvar 是 Convar), 由于 pair2 和 pair 是共享底层的对象, 所以是不允许移动的.
    let (lock, cvar) = &*pair2;
    let mut started = lock.lock().unwrap();
    *started = true;
    // We notify the condvar that the value has changed.
    cvar.notify_one();
});

// Wait for the thread to start up.
let (lock, cvar) = &*pair;
let mut started = lock.lock().unwrap();
while !*started {
    started = cvar.wait(started).unwrap();
}
```

例如 String 实现了到 str 的 Deref，则编译器自动运行如下场景：

1.  在需要 &amp;str 的函数参数场景，可以传入 &amp;String;
2.  String 可以调用 str 类型的（immutable）方法；

`*expression` 是 Rust 中的解引用表达式，当它作用于指针类型（主要包括：引用 &amp;, &amp;mut 、原生指针 \*const,
\*mut ）时， `表示指向的内容` ，这点与 C/C++ 中一致. 但当它作用于非指针类型时，它表示
`*std::ops::Deref::deref(&x)` 。比如 String 的 Deref 实现如下：

```rust
impl ops::Deref for String {
    type Target = str;

    #[inline]
    fn deref(&self) -> &str {
        unsafe { str::from_utf8_unchecked(&self.vec) }
    }
}
```

可以通过 &amp;\*V 来得到 &amp;str 类型：

```rust
let addr_string = "192.168.0.1:3000".to_string();
let addr_ref: &str = &*addr_string; // *addr_string = str，再加上外面的 & 即为 &str
```

类型自动转化是 Rust 为了追求语言简洁，使用 Deref/DerefMut 实现的另一种隐式操作，比如下面的赋值都是正确的：

```rust
let s: String = "hello".to_string();
let s1: &String = &s;
let s2: &str = s1;
let s3: &str = &&s;
let s4: &str = &&&s;
let s5: &str = &&&&s;
```

可以看到， `无论有多少个 & ，Rust 都能正确的将其转为 &str 类型` ，究其原因，是因为 `deref coercions` ，它允许在 T: Deref&lt;U&gt; 时， &amp;T 可以自动转为 &amp;U 。

因为 s2 的赋值能成功就是利用了 deref coercion 的原理，那么 s3/s4/s5 呢？如果稍微有些 Rust 经验的话，当一个类型可以调用一个不知道哪里定义的方法时， `大概率是 trait 的通用实现（blanket implementation）`
起作用的，这里就是这种情况：

```rust
impl<T: ?Sized> const Deref for &T {
    type Target = T;

    fn deref(&self) -> &T {
        *self
    }
}
```


### <span class="section-num">18.20</span> smart pointer {#smart-pointer}

实际上只要实现了 Deref trait 的类型都可以称为 smart pointer。Deref 文档列出了标准库中实现 Deref
trait 的所有类型列表：<https://doc.rust-lang.org/std/ops/trait.Deref.html>

```rust
impl Deref for String type Target = str

impl<B> Deref for Cow<'_, B> where B: ToOwned + ?Sized, <B as ToOwned>::Owned: Borrow<B> type Target = B

impl<P> Deref for Pin<P> where P: Deref, type Target = <P as Deref>::Target

impl<T> Deref for &T where T: ?Sized, type Target = T

impl<T> Deref for &mut T where T: ?Sized, type Target = T

impl<T> Deref for Ref<'_, T> where T: ?Sized, type Target = T

impl<T> Deref for RefMut<'_, T> where T: ?Sized, type Target = T

impl<T, A> Deref for Box<T, A> where A: Allocator, T: ?Sized, type Target = T

impl<T, A> Deref for Rc<T, A> where A: Allocator, T: ?Sized, type Target = T

impl<T, A> Deref for Arc<T, A> where A: Allocator, T: ?Sized, type Target = T

impl<T, A> Deref for Vec<T, A> where A: Allocator, type Target = [T]
```

上面各种智能指针都没有实现 Copy trait，所以 ownership 会转移。

Ref&lt;T&gt;/RefMut&lt;T&gt;/Rc&lt;T&gt;/Arc&lt;T&gt;/Box&lt;T&gt; ：

1.  解引用后类型都是 T；
2.  在需要 &amp;T 的地方都可以传入 &amp;Ref/&amp;Rc/&amp;Box 类型；
3.  Ref&lt;T&gt;/Box&lt;T&gt; 可以调用 T 定义的所有方法；

由于智能指针都可以调用 &amp;T 的方法, 为了避免指针自己和 &amp;T 的方法冲突, 在调用只能指针自己的方法或实现的
trait 时,一般使用全限定名称:

```rust
use std::sync::Arc;
let foo = Arc::new(vec![1.0, 2.0, 3.0]);
// The two syntaxes below are equivalent.
let a = foo.clone();
let b = Arc::clone(&foo); // 建议: 全限定名称的方法或关联函数调用
let my_weak = Arc::downgrade(&my_arc); // 全限定语法
// a, b, and foo are all Arcs that point to the same memory location

#![feature(new_uninit)]
#![feature(get_mut_unchecked)]
use std::sync::Arc;
let mut five = Arc::<u32>::new_uninit();
// Deferred initialization:
Arc::get_mut(&mut five).unwrap().write(5); // 全限定语法
let five = unsafe { five.assume_init() };


use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::thread;
let val = Arc::new(AtomicUsize::new(5));
for _ in 0..10 {
    let val = Arc::clone(&val);

    thread::spawn(move || {
        let v = val.fetch_add(1, Ordering::SeqCst);
        println!("{v:?}");
    });
}
```

String/Vec&lt;T&gt; 其实也是 smartpointer, 只不过是专用的，它们占用 3 个机器字栈空间+可变长堆空间:

1.  String 实现了 Deref&lt;Target = str&gt;;
2.  Vec&lt;T&gt; 实现了 Deref&lt;Target = [T]&gt;;

<!--listend-->

```rust
fn read_slice(slice: &[usize]) {
    // ...
}
let v = vec![0, 1];
read_slice(&v); // Deref 自动转换

let u: &[usize] = &v; // Deref 自动转换
// or like this:
let u: &[_] = &v;
```

[Use borrowed types for arguments](https://rust-unofficial.github.io/patterns/idioms/coercion-arguments.html)

-   一般情况下, 使用 borrowed type 而不是 borrowing the owned type, 例如 &amp;str over &amp;String, &amp;[T] over
    &amp;Vec&lt;T&gt;, or &amp;T over &amp;Box&lt;T&gt;.

Box/Rc/Arc 的 lifetime 规则:

1.  Box&lt;Trait&gt; 默认等效于 Box&lt;Trait + 'static&gt;;
2.  &amp;'x Box&lt;Trait&gt; 等效于 &amp;'x Box&lt;Trait+'static&gt;;
    -   'x 可能是编译器自动加的, 所以即使没有明确指定, &amp;Box&lt;Trait&gt; 等效于 &amp;Box&lt;Trait+'static&gt;;
3.  &amp;'r Ref&lt;'q, Trait&gt; 等效于 &amp;'r Ref&lt;'q, Trait+'q&gt;;


### <span class="section-num">18.21</span> Box&lt;T&gt; {#box-t}

Rust 值默认在 stack 上分配. 可以使用 Box&lt;T&gt; 将 T 值在 heap 上分配, Box 是一个 hold T 值的智能指针,在离开 Box scope 时, T 值的解构被调用, heap 上内存被释放.

```rust
  use std::mem;

  #[allow(dead_code)]
  #[derive(Debug, Clone, Copy)]
  struct Point {
      x: f64,
      y: f64,
  }

  // A Rectangle can be specified by where its top left and bottom right corners are in space
  #[allow(dead_code)]
  struct Rectangle {
      top_left: Point,
      bottom_right: Point,
  }

  fn origin() -> Point {
      Point { x: 0.0, y: 0.0 }
  }

  fn boxed_origin() -> Box<Point> {
      // Allocate this point on the heap, and return a pointer to it
      Box::new(Point { x: 0.0, y: 0.0 })
  }

  fn main() {
      // 栈上分配内存
      // (all the type annotations are superfluous) Stack allocated variables
      let point: Point = origin();
      let rectangle: Rectangle = Rectangle {
          top_left: origin(),
          bottom_right: Point { x: 3.0, y: -4.0 }
      };

      // 堆上分配内存
      // Heap allocated rectangle
      let boxed_rectangle: Box<Rectangle> = Box::new(Rectangle {
          top_left: origin(),
          bottom_right: Point { x: 3.0, y: -4.0 },
      });

      // The output of functions can be boxed
      let boxed_point: Box<Point> = Box::new(origin());

      // Double indirection
      let box_in_a_box: Box<Box<Point>> = Box::new(boxed_origin());

      println!("Point occupies {} bytes on the stack", mem::size_of_val(&point));
      println!("Rectangle occupies {} bytes on the stack", mem::size_of_val(&rectangle));

      // box size == pointer size
      println!("Boxed point occupies {} bytes on the stack", mem::size_of_val(&boxed_point));
      println!("Boxed rectangle occupies {} bytes on the stack", mem::size_of_val(&boxed_rectangle));
      println!("Boxed box occupies {} bytes on the stack", mem::size_of_val(&box_in_a_box));

      // Copy the data contained in `boxed_point` into `unboxed_point`
      let unboxed_point: Point = *boxed_point;
      println!("Unboxed point occupies {} bytes on the stack", mem::size_of_val(&unboxed_point));
  }
```

Box&lt;T&gt; 实现了 Deref&lt;Target=T&gt;, 可以使用 \* 来获得 T 值:

```rust
// https://doc.rust-lang.org/src/alloc/boxed.rs.html#1918
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized, A: Allocator> Deref for Box<T, A> {
    type Target = T;

    fn deref(&self) -> &T {
        &**self
    }
}
```

可以向 reference 一样使用 Box&lt;T&gt;:

```rust
fn main() {
    let x = 5;
    let y = Box::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y); // *y 等效为 *(y.deref()), 返回 5。
}
```

自定义对象实现 Deref：

```rust
struct MyBox<T>(T);

use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

示例：deref coercion：

```rust
fn hello(name: &str) {
    println!("Hello, {name}!");
}

fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&m); // Rust 自动进行多级 Defer 解引用，也就是 deref coercion：MyBox<String> ->  String -> str

  // 如果 rust 不做 deref coercion，则需要做如下繁琐操作
    hello(&(*m)[..]);
}
```

v = Box&lt;T&gt; 会 ownership T 的内容，当 v 被 drop 时，堆中的 T 内存也会被 drop。

-   所以直接用 v 赋值或传递到函数时，会转移所有权。

<!--listend-->

```rust
enum List {
    Cons(i32, Box<List>), // 元组类型，由于 Box 没有实现 Copy，所以整个元组没有实现 Copy
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let a = Cons(5, Box::new(Cons(10, Box::new(Nil))));
    let b = Cons(3, Box::new(a)); // a 所有权转移。这里不能传入 &a, 否则只是分配一个指针的空间。
    let c = Cons(4, Box::new(a)); // 编译器报错，解决办法是使用 Rc<T>
}
```

Box&lt;T&gt; 默认没有实现 Copy，在赋值时会被移动。其他智能指针，如 Rc/Arc/Cell/RefCell 类似。


### <span class="section-num">18.22</span> Rc/Arc&lt;T&gt; {#rc-arc-t}

Rc&lt;T&gt; 和 Box&lt;T&gt; 类似, a = Rc::new(T) 都是在堆上为 T 分配内存，并 ownership 它，但是：

1.  Rc::clone(&amp;a) 返回一个新的 Rc&lt;T&gt;, 但增加 a 的引用计数，并不会重新分配堆内存；
2.  clone 后返回的对象都被 drop 后(引用计数为 0)，a 对应的堆内存才会被释放；

<!--listend-->

```rust
impl<T, A> Clone for Rc<T, A>
where
    A: Allocator + Clone,
    T: ?Sized,
fn clone(&self) -> Rc<T, A> // 返回一个 Rc<T> 对象
```

Rc::clone(&amp;a) 返回一个引用计数, 和 a 共享堆空间的新 Rc 对象。

-   Using Rc&lt;T&gt; allows a single value to have `multiple owners`, and the count ensures that the value
    remains valid as long as `any of the owners` still exist.
-   Rc 对象指向的值是共享的，不可变的。

<!--listend-->

```rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}
```

打印引用计数：

```rust
fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!("count after creating a = {}", Rc::strong_count(&a));
    let b = Cons(3, Rc::clone(&a));
    println!("count after creating b = {}", Rc::strong_count(&a));
    {
        let c = Cons(4, Rc::clone(&a));
        println!("count after creating c = {}", Rc::strong_count(&a));
    }
    println!("count after c goes out of scope = {}", Rc::strong_count(&a));
}
```

结果：

```shell
$ cargo run
   Compiling cons-list v0.1.0 (file:///projects/cons-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.45s
     Running `target/debug/cons-list`
count after creating a = 1
count after creating b = 2
count after creating c = 3
count after c goes out of scope = 2
```

使用 Rc/Arc 的一个常见场景是在多线程中共享大的数据，避免内存拷贝。

Rc 提供了 get_mut()/make_mut()/into_inner() 方法来修改对象：

Rc::get_mut() 方法

```text
pub fn get_mut(this: &mut Rc<T, A>) -> Option<&mut T>
```

`当没有其他 Rc 对象引用 this 时`, 也就是 this Rc 对象的引用计数为 1 时, 返回 &amp;mut T, 否则返回 None.

-   而 make_mut() 方法在引用计数 &gt;=1 时, clone 一个新对象来替换 this, 然后返回他的 &amp;mut T;

<!--listend-->

```rust
use std::rc::Rc;

let mut x = Rc::new(3);
*Rc::get_mut(&mut x).unwrap() = 4;
assert_eq!(*x, 4);

let _y = Rc::clone(&x);
assert!(Rc::get_mut(&mut x).is_none());
```

Rc::make_mut() 方法：

```text
pub fn make_mut(this: &mut Rc<T, A>) -> &mut T
```

创建传入 Rc&lt;T&gt; 的 &amp;mutT 可变引用,当调用该方法时, 如果传入的 Rc 对象被 clone 过, 也就是引用计数 &gt;=2,
则该方法会 clone 生成一个新的 Rc 对象并替换 this, 这被称为 clone-on-write;

```rust
use std::rc::Rc;

let mut data = Rc::new(5);
*Rc::make_mut(&mut data) += 1;         // Won't clone anything
let mut other_data = Rc::clone(&data); // Won't clone inner data

// 由于 data 被 clone 过, 再次调用 data 的 make_mut 时会 clone 一个新 Rc 对象并替换 data
// 这是  other_data 的引用计数 -1 变为 1, data 的引用计数也为 1
*Rc::make_mut(&mut data) += 1;         // Clones inner data
// data 是刚才  clone 生成的新对象, 计数为 1, 所以这次不会再 clone
*Rc::make_mut(&mut data) += 1;         // Won't clone anything

// other_data 引用计数为 1, 所以也不会 clone
*Rc::make_mut(&mut other_data) *= 2;   // Won't clone anything

// Now `data` and `other_data` point to different allocations.
assert_eq!(*data, 8);
assert_eq!(*other_data, 12);
```

Rc::into_inner() 方法：

```text
pub fn into_inner(this: Rc<T, A>) -> Option<T>
```

Returns the inner value, if the Rc has `exactly one strong` reference. Otherwise, `None` is returned and
`the Rc is dropped`.

-   传入的是 Rc 对象本身而非引用, 会消耗传入的 Rc(其他类型的 into_inner() 方法都是类似的语义).

This will succeed even if there are outstanding weak references.

If `Rc::into_inner` is called on every clone of this Rc, it is guaranteed that exactly one of the
calls returns the inner value. This means in particular that the inner value is not dropped.

This is equivalent to Rc::try_unwrap(this).ok(). (Note that these are not equivalent for Arc, due to
race conditions that do not apply to Rc.)

BTW, into_inner() 是一个常见的类型方法, 都是从包装类型生成对应的 T, 例如:

-   pub fn into_inner(boxed: Box&lt;T, A&gt;) -&gt; T
-   Cell/RefCell

<!--listend-->

```rust
// pub fn into_inner(self) -> T

use std::cell::Cell;
let c = Cell::new(5);
let five = c.into_inner();
assert_eq!(five, 5);

```


### <span class="section-num">18.23</span> Cell&lt;T&gt;/RefCell&lt;T&gt; {#cell-t-refcell-t}

内部可变性（Interior Mutability）

Cell/RefCell 没有实现 Sync trait，Rust 不允许多线程使用它们。Mutext&lt;T&gt;/RwLock&lt;T&gt;/OnceLock&lt;T&gt;/原子操作类型，提供了线程安全的内部可变性。都可以使用共享引用 &amp;self 来调用它们的 mut 方法，例如 lock().

---

Rc&lt;SpiderRobot&gt; 返回的对象不能对 SpiderRobot 进行变更，即 Rc 是共享的。

在不可变对象中引用一些可变性，称为 interior mutability。

Rust 标准库的 std::cell module 提供了 Cell&lt;T&gt; 和 RefCell&lt;T&gt; 来支持这种场景：在对 Cell 自身没有 mut
access 的情况下，也能 get/set 它的 field。

1.  Cell::new(value)： Creates a new Cell, `moving` the given value into it.
2.  cell.get()： Returns a copy of the value in the cell.
3.  cell.set(value)： Stores the given value in the cell, dropping the previously stored value. This
    method takes `self as a non-mut reference`: fn set(&amp;self, value: T) // note: not \`&amp;mut self\`

cell.set() 方法是 &amp;self 类型而非 &amp;mut self ，但是又能对 self 进行设置操作，这是 Cell&lt;T&gt; 在 Rust 中存在的主要价值。

-   get() 返回地是 &lt;T&gt; 的 copy，需要 T 实现 Copy trait，但是很多 T 类型，尤其是资源型，如 File 是没有实现 Copy trait 的。解决办法是使用 RefCell&lt;T&gt;。

<!--listend-->

```rust
use std::cell::Cell;
pub struct SpiderRobot {
  // ...
  hardware_error_count: Cell<u32>, // 成员是 Cell<T> 类型
  // ...
}


// SpiderRobot 的非 mutt 方法也可以 get/set 对应的 Cell<T> 成员。
impl SpiderRobot {
  /// Increase the error count by 1.
  pub fn add_hardware_error(&self) { // &self 而非 &mut self
    let n = self.hardware_error_count.get();
    self.hardware_error_count.set(n + 1);
  }

  /// True if any hardware errors have been reported.
  pub fn has_hardware_errors(&self) -> bool {
    self.hardware_error_count.get() > 0
  }
}

```

RefCell&lt;T&gt; supports borrowing references to its T value:

1.  RefCell::new(value) Creates a new RefCell, `moving` value into it.
2.  ref_cell.borrow() Returns a `Ref<T>`, which is essentially just a shared reference to the value
    stored in ref_cell. This method panics if the value is already mutably borrowed; see details to
    follow.
3.  ref_cell.borrow_mut() Returns a `RefMut<T>`, essentially a mutable reference to the value in
    ref_cell. This method panics if the value is already borrowed; see details to follow.
4.  ref_cell.try_borrow(), ref_cell.try_borrow_mut() Work just like borrow() and borrow_mut(), but
    return `a Result`. Instead of panicking if the value is already mutably borrowed, they return an
    Err value.

borrow()/borrow_mut() 返回的 Ref&lt;T&gt;/RefMut&lt;T&gt; 是智能指针，实现了 Deref&lt;Target=T&gt;, 可以直接调用 T 的方法.

对于已经通过 r = RefCell&lt;T&gt;.borrow() 的 r 不能调用 borrow_mut(), 否则会 panic:

```rust
use std::cell::RefCell;

let ref_cell: RefCell<String> =  RefCell::new("hello".to_string());

let r = ref_cell.borrow(); // ok, returns a Ref<String>
let count = r.len(); // ok, returns "hello".len()
assert_eq!(count, 5);

let mut w = ref_cell.borrow_mut(); // panic: already borrowed
w.push_str(" world");
```

例子:

```rust
pub struct SpiderRobot {
  // ...
  log_file: RefCell<File>,
  // ...
}

impl SpiderRobot {
  /// Write a line to the log file.
  pub fn log(&self, message: &str) {
    // self 虽然是不可变引用, 但是使用 RefCell<T>.borrow_mut() 来返 RefMut<T>
    let mut file = self.log_file.borrow_mut();
    // `writeln!` is like `println!`, but sends
    // output to the given file.
    writeln!(file, "{}", message).unwrap();
  }
}

```

另一个例子：

```rust
// https://doc.rust-lang.org/book/ch15-05-interior-mutability.html
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: RefCell::new(vec![]),
            }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        // --snip--

        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```


### <span class="section-num">18.24</span> Pin/UnPin {#pin-unpin}

由于 move 机制的存在，导致在 Rust 很难去正确表达『自引用』的结构，比如链表、树等。主要问题：move 只会进行值本身的拷贝，指针的指向则不变。如果被 move 的结构有指向其他字段的指针，那么这个指向被 move 后就是非法的，因为原始指向已经换地址了。

Pin 的常用场景是 std::future::Feature trait 的 poll 方法：由于 Future trait object 必须保存执行上下文中 stack 上的变量，而 Future 下一次被 wake 执行的时机是不定的，所以为了避免 Future 对象上报错的
stack 变量的地址发生变化导致引用出错，需要将 Future 对象设置为 Pin&lt;&amp;mut Self&gt; 类型，表示该对象不能被
move 转移所有权。

```rust
pub trait Future {
    type Output;

    // Required method
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

举例：

```rust
#![feature(noop_waker)]

use std::future::Future;
use std::task;

let waker = task::Waker::noop();
let mut cx = task::Context::from_waker(&waker);

let mut future = Box::pin(async { 10 });
assert_eq!(future.as_mut().poll(&mut cx), task::Poll::Ready(10));
```


## <span class="section-num">19</span> iterator {#iterator}

Iterator trait 定义：

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // 其他都是缺省实现的方法。
}
```

可以直接在自定义类型上实现迭代器，如 impl Iterator for ReadDir，也可以通过类型的方法返回一个实现迭代器的对象（惯例是 Iter/IterMut/IntoIter），惯例的方法名是：

1.  iter(&amp;self): 返回的类型惯例是 Iter，迭代返回的是 &amp;T 类型；
2.  iter_mut(&amp;mut self): 返回的类型惯例是 IterMut，迭代返回的是 &amp;mut T 类型；
3.  into_iter(self): 返回的迭代器对象惯例类型是 IntoIter，转移了被迭代对象的所有权，迭代返回的是 T 类型；

for-in 循环的对象需要实现 std::iter::IntoIterator，也就是对象的 into_iter() 返回一个 Iterator, 但是也可以给被迭代对象加 &amp;obj 或 &amp;mut obj 来让 for-in 使用 iter/iter_mut:

Rust `为实现了 Iterator 的类型自动实现了 IntoIterator` ，所以可以使用 for i in vec.iter() {println!("{i}")};

```rust
#[rustc_const_unstable(feature = "const_intoiterator_identity", issue = "90603")]
#[stable(feature = "rust1", since = "1.0.0")]
impl<I: Iterator> IntoIterator for I {
    type Item = I::Item;
    type IntoIter = I;

    #[inline]
    fn into_iter(self) -> I {
        self
    }
}
```

Rust 同时也为 Vec&lt;T&gt;，[T; N]，HashSet&lt;T&gt; 等实现了 IntoIterator。

```rust
  let mut v = vec![1, 2, 3];

  for i in &v {
      println!("{i}")
  }

  for in in &mut v {
      println!("{i}")
  }

  for i in v {
      println!("{i}");
  }
  // v 不可再访问
```

注意：

1.  如果直接在自定义类型上实现迭代器，如 impl Iterator for ReadDir，则在 ReadDir 上调用一些 self 迭代器方法，如 readDir.take(3) 后，readDir 将失效。
2.  但是如果在 iter()/iter_mut() 返回的迭代器对象上调用 take() 方法，消耗的是迭代器本身，原对象还可以正常方法；
3.  由于 into_iter() 会转移对象所有权，迭代也是直接放回元素本身，所以 take() 返回并消耗对象。

<!--listend-->

```rust
  // https://rust-unofficial.github.io/too-many-lists/second-into-iter.html

  let a = [1, 2, 3];
  let mut iter = a.iter(); // 返回一个 Vec 定义的迭代器对象，迭代式返回 &T
  assert_eq!(Some(&1), iter.next());
  assert_eq!(Some(&2), iter.next());
  assert_eq!(Some(&3), iter.next());


  // IntoIter
  pub struct IntoIter<T>(List<T>);  // List 自定义的实现 Iterator 的迭代器类型

  impl<T> List<T> {
      // into_iter() 方法的输入是 self，会消耗 List 对象本身和元素。
      pub fn into_iter(self) -> IntoIter<T> { // List 的 into_iter() 方法返回该自定义迭代器对象
          IntoIter(self) // self 所有权转义到返回的 IntoIter 对象
      }
  }

  impl<T> Iterator for IntoIter<T> { // List 自定义的 Into
      type Item = T;
      fn next(&mut self) -> Option<Self::Item> {
          // access fields of a tuple struct numerically
          self.0.pop()
      }
  }


  // Iter
  pub struct Iter<'a, T> {
      next: Option<&'a Node<T>>,
  }

  impl<'a, T> List<T> {
      pub fn iter(&'a self) -> Iter<'a, T> {
          Iter { next: self.head.map(|node| &'a node) }
          }
          }

  impl<'a, T> Iterator for Iter<'a, T> {
      type Item = &'a T;
      fn next(&'a mut self) -> Option<Self::Item> {
          self.next.map(|node| {
              self.next = node.next.map(|node| &'a node);
              &'a node.elem
          })
      }
  }


  // IterMut
  pub struct IterMut<'a, T> {
      next: Option<&'a mut Node<T>>,
  }

  impl<T> List<T> {
      pub fn iter_mut(&self) -> IterMut<'_, T> {
          IterMut { next: self.head.as_deref_mut() }
      }
  }

  impl<'a, T> Iterator for IterMut<'a, T> {
      type Item = &'a mut T;

      fn next(&mut self) -> Option<Self::Item> {
          self.next.map(|node| {
              self.next = node.next.as_deref_mut();
              &mut node.elem
          })
      }
  }
```

示例：

```rust
  #[test]
  fn iterator_demonstration() {
      let v1 = vec![1, 2, 3];

      let mut v1_iter = v1.iter(); // iter() 方法返回一个迭代器类型，它的 next() 方法，返回的是 &T 类型。

      assert_eq!(v1_iter.next(), Some(&1));
      assert_eq!(v1_iter.next(), Some(&2));
      assert_eq!(v1_iter.next(), Some(&3));
      assert_eq!(v1_iter.next(), None);
  }
```

消耗迭代的方式：

1.  for-in 循环: 需要对象实现 std::iter::IntoIterator trait；
2.  Iterator trait 定义的泛型方法大部分都是消耗 self，如 iter.sum()/map() 等；
3.  绝大部分迭代器方法使用 self，所以 `调用后迭代器对象所有权发生转移` 。

for-in 循环器默认使用的是 IntoIter 迭代器，它会消耗被迭代对象的元素。可以通过 &amp;obj 或 &amp;mut obj 来使用 Iter 或 IterMut 迭代器；

```rust
let mut array = [1, 2, 3];

// IntoIter
for n in array { // n 为 uint
 println!(n);
}
// 迭代后 array 不能再使用

// Iter
for n in &array { // n 为 &uint
  println!(n);
}
// 迭代后 array 可以使用

// IterMut
for n in &mut array { // n 为 &mut uint
  println!(n);
}
```

另外 for 循环迭代器也可以使用 pattern match 来展开引用：

```rust
let mut array = [(1, 2, 3), (4, 5, 6)];
for (a, b, &c) in &array {
    println!("{} {} {}", a, b, *c);
}
```

迭代器泛型方法：

-   Vec&lt;T&gt; 实现了上面三种类型的迭代器。
-   迭代器适配器（iterator adaptor）是 Iterator trait 上定义的方法，它一般会消耗旧 Iterator （这些适配器方法的输入绝大大部分是 self）并返回新的 Iterator。由于 Iterator 都是 lazy 的，必须被消耗时才能干活，如调用 collect() 方法；

<!--listend-->

```rust
#[derive(PartialEq, Debug)]
struct Shoe {
    size: u32,
    style: String,
}

// 由于返回一个 Vec<Shoe> ，所以需要 into_iter() 返回的 IntoIter 迭代器。
fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}
```

Iterator trait 的 collect() 方法返回一个新的可迭代对象, 定义如下：

-   collect() 返回的是一个实现了 FromIterator&lt;Self::Item&gt; trait 的对象；

<!--listend-->

```rust
pub trait Iterator {
    type Item;

    fn collect<B: FromIterator<Self::Item>>(self) -> B
    where
        Self: Sized,
    {
        FromIterator::from_iter(self)
    }
}

// https://doc.rust-lang.org/std/iter/trait.FromIterator.html
pub trait FromIterator<A>: Sized {
    // Required method
    fn from_iter<T>(iter: T) -> Self
       where T: IntoIterator<Item = A>;
}
```

示例：

-   Vec&lt;T&gt; 实现了 FromIterator&lt;T&gt; trait, 所以 collect() 方法可以返回该类型对象:

<!--listend-->

```rust
 let v1: Vec<i32> = vec![1, 2, 3];
 let v2: Vec<&i32> = v1.iter().map(|x| x + 1).collect(); // iter() 返回 &32, 所以追踪 collect() 返回的是 &i32.
 assert_eq!(v2, vec![2, 3, 4]);
```

另一个例子:

```rust
impl Config {
    pub fn build(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone(); // args[1] 返回的是 String, 必须要 & 或 clone, 否则 rust 不允许从 args 中 takeoff
        let file_path = args[2].clone();

        let ignore_case = env::var("IGNORE_CASE").is_ok();

        Ok(Config {
            query,
            file_path,
            ignore_case,
        })
    }
}



fn main() {
    let args: Vec<String> = env::args().collect(); // args() 返回的是 Vec<String>;
    let config = Config::build(&args).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {err}");
        process::exit(1);
    });
    // --snip--
}
```

用迭代器重写:

```rust
impl Config {
    pub fn build(
        mut args: impl Iterator<Item = String>, // Vec<String> 实现了该 trait
    ) -> Result<Config, &'static str> {
        args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };

        let file_path = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file path"),
        };

        let ignore_case = env::var("IGNORE_CASE").is_ok();

        Ok(Config {
            query,
            file_path,
            ignore_case,
        })
    }
}

fn main() {
    let config = Config::build(env::args()).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {err}");
        process::exit(1);
    });

    // --snip--
}
```

Option/Result 都是 enum 类型，但是也支持迭代（实现了 IntoIterator），效果就如一个或0个元素。

```rust
  let turing = Some("Turing");
  let mut logicians = vec!["Curry", "Kleene", "Markov"];
  logicians.extend(turing); // Option 实现了 IntoIterator, 因此可以传入 .extend() 方法中

// 也可以传入 chain() 方法中
let turing = Some("Turing");
let logicians = vec!["Curry", "Kleene", "Markov"];
for logician in logicians.iter().chain(turing.iter()) {
    println!("{logician} is a logician");
}

```


### <span class="section-num">19.1</span> 迭代器方法 {#迭代器方法}

```rust
pub trait Iterator {
    type Item;

    // Required method
    fn next(&mut self) -> Option<Self::Item>;

    // 返回下一个 N 个元素的数组，N 的数量可以指定或推导
    fn next_chunk<const N: usize>( &mut self ) -> Result<[Self::Item; N], IntoIter<Self::Item, N>> where Self: Sized {  }

    let mut iter = "lorem".chars();
    assert_eq!(iter.next_chunk().unwrap(), ['l', 'o']);              // N is inferred as 2
    assert_eq!(iter.next_chunk().unwrap(), ['r', 'e', 'm']);         // N is inferred as 3
    assert_eq!(iter.next_chunk::<4>().unwrap_err().as_slice(), &[]); // N is explicitly 4
    let quote = "not all those who wander are lost";
    let [first, second, third] = quote.split_whitespace().next_chunk().unwrap(); // 自动推导
    assert_eq!(first, "not");
    assert_eq!(second, "all");
    assert_eq!(third, "those");

    // 返回迭代器中剩下元素的下界和上界
    fn size_hint(&self) -> (usize, Option<usize>) { ... }

    // 返回迭代器中元素数量（消耗）
    fn count(self) -> usize where Self: Sized { ... }

    // 返回最后一个元素（消耗）
    fn last(self) -> Option<Self::Item> where Self: Sized { ... }

    // 前进迭代器 n 个元素
    fn advance_by(&mut self, n: usize) -> Result<(), NonZero<usize>> { ... }
    #![feature(iter_advance_by)]
    use std::num::NonZeroUsize;
    let a = [1, 2, 3, 4];
    let mut iter = a.iter();
    assert_eq!(iter.advance_by(2), Ok(()));
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.advance_by(0), Ok(()));
    assert_eq!(iter.advance_by(100), Err(NonZeroUsize::new(99).unwrap())); // only `&4` was skipped

    // 返回第 n 个元素（0开始）
    fn nth(&mut self, n: usize) -> Option<Self::Item> { ... }
    let a = [1, 2, 3];
    let mut iter = a.iter();
    assert_eq!(iter.nth(1), Some(&2));
    assert_eq!(iter.nth(1), None); // 调用 nth(n) 多次并不重复返回
    assert_eq!(iter.nth(10), None);

    // 返回一个新的迭代器，每次返回原始迭代器的 +step 后的元素
    fn step_by(self, step: usize) -> StepBy<Self> where Self: Sized { ... }
    let a = [0, 1, 2, 3, 4, 5];
    let mut iter = a.iter().step_by(2);
    assert_eq!(iter.next(), Some(&0)); // 第一个元素
    assert_eq!(iter.next(), Some(&2)); // +2
    assert_eq!(iter.next(), Some(&4));
    assert_eq!(iter.next(), None);

    // 将两个迭代器合并为一个，先迭代自身再迭代传入的 other
    fn chain<U>(self, other: U) -> Chain<Self, <U as IntoIterator>::IntoIter> where Self: Sized, U: IntoIterator<Item = Self::Item> { ... }
    let a1 = [1, 2, 3];
    let a2 = [4, 5, 6];
    let mut iter = a1.iter().chain(a2.iter());
    assert_eq!(iter.next(), Some(&1));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&4));
    assert_eq!(iter.next(), Some(&5));
    assert_eq!(iter.next(), Some(&6));
    assert_eq!(iter.next(), None);

    // 从两个迭代器返回一个新的迭代器，每次各返回一个值，直到某个迭代器完成
    fn zip<U>(self, other: U) -> Zip<Self, <U as IntoIterator>::IntoIter> where Self: Sized, U: IntoIterator { ... }
    let a1 = [1, 2, 3];
    let a2 = [4, 5, 6];
    let mut iter = a1.iter().zip(a2.iter());
    assert_eq!(iter.next(), Some((&1, &4)));
    assert_eq!(iter.next(), Some((&2, &5)));
    assert_eq!(iter.next(), Some((&3, &6)));
    assert_eq!(iter.next(), None);
    let enumerate: Vec<_> = "foo".chars().enumerate().collect();
    let zipper: Vec<_> = (0..).zip("foo".chars()).collect();
    assert_eq!((0, 'f'), enumerate[0]);
    assert_eq!((0, 'f'), zipper[0]);
    assert_eq!((1, 'o'), enumerate[1]);
    assert_eq!((1, 'o'), zipper[1]);
    assert_eq!((2, 'o'), enumerate[2]);
    assert_eq!((2, 'o'), zipper[2]);

    // 在迭代器元素间插入一个 separator 元素（需要实现 Clone）
    fn intersperse(self, separator: Self::Item) -> Intersperse<Self> where Self: Sized, Self::Item: Clone { ... }
    #![feature(iter_intersperse)]
    let hello = ["Hello", "World", "!"].iter().copied().intersperse(" ").collect::<String>();
    assert_eq!(hello, "Hello World !");
    let mut a = [0, 1, 2].iter().intersperse(&100);
    assert_eq!(a.next(), Some(&0));   // The first element from `a`.
    assert_eq!(a.next(), Some(&100)); // The separator.
    assert_eq!(a.next(), Some(&1));   // The next element from `a`.
    assert_eq!(a.next(), Some(&100)); // The separator.
    assert_eq!(a.next(), Some(&2));   // The last element from `a`.
    assert_eq!(a.next(), None);       // The iterator is finished.

    // 使用指定的闭包函数插入分隔符
    fn intersperse_with<G>(self, separator: G) -> IntersperseWith<Self, G> where Self: Sized, G: FnMut() -> Self::Item { ... }

    // 使用指定的闭包函数来处理每一个元素（消耗原迭代器），函数结果形成另一个可迭代对象
    fn map<B, F>(self, f: F) -> Map<Self, F> where Self: Sized, F: FnMut(Self::Item) -> B { ... }
    let a = [1, 2, 3];
    let mut iter = a.iter().map(|x| 2 * x);
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(4));
    assert_eq!(iter.next(), Some(6));
    assert_eq!(iter.next(), None);

    // 对每一个元素调用指定的闭包函数（消耗闭包）
    fn for_each<F>(self, f: F) where Self: Sized, F: FnMut(Self::Item) { ... }
    use std::sync::mpsc::channel;
    let (tx, rx) = channel(); (0..5).map(|x| x * 2 + 1).for_each(move |x| tx.send(x).unwrap());
    let v: Vec<_> = rx.iter().collect();
    assert_eq!(v, vec![1, 3, 5, 7, 9]);


// 使用 predicate 过滤元素，返回为 true 的元素的迭代器
fn filter<P>(self, predicate: P) -> Filter<Self, P> where Self: Sized, P: FnMut(&Self::Item) -> bool { ... }
let a = [0, 1, 2];
let mut iter = a.iter().filter(|x| **x > 1); // need two *s! 和 map() 不同，filter 闭包函数的参数是 &T 类型。
assert_eq!(iter.next(), Some(&2));
assert_eq!(iter.next(), None);
let a = [0, 1, 2];
let mut iter = a.iter().filter(|&x| *x > 1); // both & and *
assert_eq!(iter.next(), Some(&2));
assert_eq!(iter.next(), None);
let a = [0, 1, 2];
let mut iter = a.iter().filter(|&&x| x > 1); // two &s
assert_eq!(iter.next(), Some(&2));
assert_eq!(iter.next(), None);

// 对迭代元素值执行 f 闭包，返回结果为 Some(value) 的 value 迭代器
fn filter_map<B, F>(self, f: F) -> FilterMap<Self, F> where Self: Sized, F: FnMut(Self::Item) -> Option<B> { ... }
let a = ["1", "two", "NaN", "four", "5"];
let mut iter = a.iter().filter_map(|s| s.parse().ok());
assert_eq!(iter.next(), Some(1));
assert_eq!(iter.next(), Some(5));
assert_eq!(iter.next(), None);
// 等效为 filter().map()
let a = ["1", "two", "NaN", "four", "5"];
let mut iter = a.iter().map(|s| s.parse()).filter(|s| s.is_ok()).map(|s| s.unwrap());
assert_eq!(iter.next(), Some(1));
assert_eq!(iter.next(), Some(5));
assert_eq!(iter.next(), None);

// 返回 (i, value) 的迭代器，i 的类型为 usize
fn enumerate(self) -> Enumerate<Self> where Self: Sized { ... }
let a = ['a', 'b', 'c'];
let mut iter = a.iter().enumerate();
assert_eq!(iter.next(), Some((0, &'a')));
assert_eq!(iter.next(), Some((1, &'b')));
assert_eq!(iter.next(), Some((2, &'c')));
assert_eq!(iter.next(), None);

// 返回一个迭代器，他的 peek/peek_mut 返回下一个迭代元素，但是并不消费迭代元素
fn peekable(self) -> Peekable<Self> where Self: Sized { ... }
let xs = [1, 2, 3];
let mut iter = xs.iter().peekable();
// peek() lets us see into the future
assert_eq!(iter.peek(), Some(&&1));
assert_eq!(iter.next(), Some(&1));
assert_eq!(iter.next(), Some(&2));
// we can peek() multiple times, the iterator won't advance
assert_eq!(iter.peek(), Some(&&3));
assert_eq!(iter.peek(), Some(&&3));
assert_eq!(iter.next(), Some(&3));
// after the iterator is finished, so is peek()
assert_eq!(iter.peek(), None);
assert_eq!(iter.next(), None);

// 迭代时一直忽略元素，直到 predicte 返回 false（包含返回 false 的元素）
fn skip_while<P>(self, predicate: P) -> SkipWhile<Self, P> where Self: Sized, P: FnMut(&Self::Item) -> bool { ... }
let a = [-1i32, 0, 1];
let mut iter = a.iter().skip_while(|x| x.is_negative());
assert_eq!(iter.next(), Some(&0));
assert_eq!(iter.next(), Some(&1));
assert_eq!(iter.next(), None);
// 注意：一旦 predicate 返回 false，后续就不再对元素进行判断
let a = [-1, 0, 1, -2];
let mut iter = a.iter().skip_while(|x| **x < 0);
assert_eq!(iter.next(), Some(&0));
assert_eq!(iter.next(), Some(&1));
// while this would have been false, since we already got a false,
// skip_while() isn't used any more
assert_eq!(iter.next(), Some(&-2));
assert_eq!(iter.next(), None);

// 当 predicate 返回 true 时，返回元素。但是一旦返回 false，则忽略后续的元素。
fn take_while<P>(self, predicate: P) -> TakeWhile<Self, P> where Self: Sized, P: FnMut(&Self::Item) -> bool { ... }
let a = [-1i32, 0, 1];
let mut iter = a.iter().take_while(|x| x.is_negative());
assert_eq!(iter.next(), Some(&-1));
assert_eq!(iter.next(), None);
// 一旦 predicate 返回 false，就不再返回后续的元素。
let a = [-1, 0, 1, -2];
let mut iter = a.iter().take_while(|x| **x < 0);
assert_eq!(iter.next(), Some(&-1));
// We have more elements that are less than zero, but since we already
// got a false, take_while() isn't used any more
assert_eq!(iter.next(), None);

// 持续对每个元素应用 predicate，直到它返回 None（也就是 predicate 返回 some 时继续）
fn map_while<B, P>(self, predicate: P) -> MapWhile<Self, P> where Self: Sized, P: FnMut(Self::Item) -> Option<B> { ... }
let a = [-1i32, 4, 0, 1];
let mut iter = a.iter().map_while(|x| 16i32.checked_div(*x));
assert_eq!(iter.next(), Some(-16));
assert_eq!(iter.next(), Some(4));
assert_eq!(iter.next(), None);

// 忽略前 n 个元素
fn skip(self, n: usize) -> Skip<Self> where Self: Sized { ... }
let a = [1, 2, 3];
let mut iter = a.iter().skip(2);
assert_eq!(iter.next(), Some(&3));
assert_eq!(iter.next(), None);

// 只获取前 n 个元素
fn take(self, n: usize) -> Take<Self> where Self: Sized { ... }
let v = [1, 2];
let mut iter = v.into_iter().take(5);
assert_eq!(iter.next(), Some(1));
assert_eq!(iter.next(), Some(2));
assert_eq!(iter.next(), None);

// 和 fold 类似，但返回的是可迭代对象，每次迭代返回 f 闭包执行的结果 Some，当闭包 f 返回 None 时停止迭代
fn scan<St, B, F>(self, initial_state: St, f: F) -> Scan<Self, St, F> where Self: Sized, F: FnMut(&mut St, Self::Item) -> Option<B> { ... }
let a = [1, 2, 3, 4];
let mut iter = a.iter().scan(1, |state, &x| {
    // each iteration, we'll multiply the state by the element ...
    *state = *state * x;
    // ... and terminate if the state exceeds 6
    if *state > 6 {
        return None;
    }
    // ... else yield the negation of the state
    Some(-*state)
});
assert_eq!(iter.next(), Some(-1));
assert_eq!(iter.next(), Some(-2));
assert_eq!(iter.next(), Some(-6));
assert_eq!(iter.next(), None);

// 先对元素进行 map F 操作，F 返回一个迭代器，然后对各 map 结果迭代器进行 flat
fn flat_map<U, F>(self, f: F) -> FlatMap<Self, U, F> where Self: Sized, U: IntoIterator, F: FnMut(Self::Item) -> U { ... }
let words = ["alpha", "beta", "gamma"];
// chars() returns an iterator
let merged: String = words.iter().flat_map(|s| s.chars()).collect();
assert_eq!(merged, "alphabetagamma");

// 返回一个将可迭代元素打平的迭代器
fn flatten(self) -> Flatten<Self> where Self: Sized, Self::Item: IntoIterator { ... }
let data = vec![vec![1, 2, 3, 4], vec![5, 6]];
let flattened = data.into_iter().flatten().collect::<Vec<u8>>();
assert_eq!(flattened, &[1, 2, 3, 4, 5, 6]);
// Option/Result 也是可迭代的，所以可以使用 flatten() 处理
let options = vec![Some(123), Some(321), None, Some(231)];
let flattened_options: Vec<_> = options.into_iter().flatten().collect();
assert_eq!(flattened_options, vec![123, 321, 231]);
let results = vec![Ok(123), Ok(321), Err(456), Ok(231)];
let flattened_results: Vec<_> = results.into_iter().flatten().collect();
assert_eq!(flattened_results, vec![123, 321, 231]);
// flatten 只会打平一级
let d3 = [[[1, 2], [3, 4]], [[5, 6], [7, 8]]];
let d2 = d3.iter().flatten().collect::<Vec<_>>();
assert_eq!(d2, [&[1, 2], &[3, 4], &[5, 6], &[7, 8]]);
let d1 = d3.iter().flatten().flatten().collect::<Vec<_>>();
assert_eq!(d1, [&1, &2, &3, &4, &5, &6, &7, &8]);

// 先将元素按照 N 分组 window（window 间元素有重合），然后再对每个 window 的元素执行 f 闭包如果元素
// 少于 N，则返回空迭代器
fn map_windows<F, R, const N: usize>(self, f: F) -> MapWindows<Self, F, N> where Self: Sized, F: FnMut(&[Self::Item; N]) -> R { ... }
#![feature(iter_map_windows)]
let strings = "abcd".chars()
    .map_windows(|[x, y]| format!("{}+{}", x, y)) //  &['a', 'b'], &['b', 'c'] and &['c', 'd']
    .collect::<Vec<String>>();
assert_eq!(strings, vec!["a+b", "b+c", "c+d"]);
#![feature(iter_map_windows)]
let mut it = [0.5, 1.0, 3.5, 3.0, 8.5, 8.5, f32::NAN].iter()
    .map_windows(|[a, b]| a <= b);
assert_eq!(it.next(), Some(true));  // 0.5 <= 1.0
assert_eq!(it.next(), Some(true));  // 1.0 <= 3.5
assert_eq!(it.next(), Some(false)); // 3.5 <= 3.0
assert_eq!(it.next(), Some(true));  // 3.0 <= 8.5
assert_eq!(it.next(), Some(true));  // 8.5 <= 8.5
assert_eq!(it.next(), Some(false)); // 8.5 <= NAN
assert_eq!(it.next(), None);

// 返回一个新迭代器，终止于原迭代器返回的第一个 None，用于防止原迭代器不规范的实现
fn fuse(self) -> Fuse<Self> where Self: Sized { ... }

// 对迭代的每一个元素执行闭包操作
fn inspect<F>(self, f: F) -> Inspect<Self, F> where Self: Sized, F: FnMut(&Self::Item) { ... }
// let's add some inspect() calls to investigate what's happening
let sum = a.iter()
    .cloned()
    .inspect(|x| println!("about to filter: {x}"))
    .filter(|x| x % 2 == 0)
    .inspect(|x| println!("made it through filter: {x}"))
    .fold(0, |sum, i| sum + i);
println!("{sum}");

// borrow 但不消耗 self，这样原来的迭代器可以继续使用
fn by_ref(&mut self) -> &mut Self where Self: Sized { ... }
let mut words = ["hello", "world", "of", "Rust"].into_iter(); // words 是迭代器对象
// Take the first two words.
let hello_world: Vec<_> = words.by_ref().take(2).collect(); // 不消耗  words
assert_eq!(hello_world, vec!["hello", "world"]);
// Collect the rest of the words.
// We can only do this because we used `by_ref` earlier.
let of_rust: Vec<_> = words.collect(); // words 还可以继续使用
assert_eq!(of_rust, vec!["of", "Rust"]);

// 使用 FromIterator trait 从迭代器元素生成 B 类型对象
fn collect<B>(self) -> B where B: FromIterator<Self::Item>, Self: Sized { ... }
let doubled: Vec<i32> = a.iter() .map(|&x| x * 2) .collect();
assert_eq!(vec![2, 4, 6], doubled);
let a = [1, 2, 3];
let doubled = a.iter().map(|x| x * 2).collect::<Vec<i32>>();
assert_eq!(vec![2, 4, 6], doubled);
// 检查 Result 列表
let results = [Ok(1), Err("nope"), Ok(3), Err("bad")];
let result: Result<Vec<_>, &str> = results.iter().cloned().collect();
// gives us the first error
assert_eq!(Err("nope"), result);
let results = [Ok(1), Ok(3)];
let result: Result<Vec<_>, &str> = results.iter().cloned().collect();
// gives us the list of answers
assert_eq!(Ok(vec![1, 3]), result);

// 允许失败的 collect，主要用于将迭代元素是 Option<T> 转换为 Option<Collector<T>> 类型
fn try_collect<B>( &mut self ) -> <<Self::Item as Try>::Residual as Residual<B>>::TryType where Self: Sized, Self::Item: Try, <Self::Item as Try>::Residual: Residual<B>, B: FromIterator<<Self::Item as Try>::Output> { ... }
#![feature(iterator_try_collect)]
let u = vec![Some(1), Some(2), Some(3)];
let v = u.into_iter().try_collect::<Vec<i32>>();
assert_eq!(v, Some(vec![1, 2, 3]));

// 将 self 迭代的元素添加到传入的 collection 中
fn collect_into<E>(self, collection: &mut E) -> &mut E where E: Extend<Self::Item>, Self: Sized { ... }
#![feature(iter_collect_into)]
let a = [1, 2, 3];
let mut vec: Vec::<i32> = vec![0, 1];
a.iter().map(|&x| x * 2).collect_into(&mut vec);
a.iter().map(|&x| x * 10).collect_into(&mut vec);
assert_eq!(vec, vec![0, 1, 2, 4, 6, 10, 20, 30]);

// 使用 f 将迭代器元素分两组，分别为返回 true/false 的元素
fn partition<B, F>(self, f: F) -> (B, B) where Self: Sized, B: Default + Extend<Self::Item>, F: FnMut(&Self::Item) -> bool { ... }
let a = [1, 2, 3];
let (even, odd): (Vec<_>, Vec<_>) = a
    .into_iter()
    .partition(|n| n % 2 == 0);
assert_eq!(even, vec![2]);
assert_eq!(odd, vec![1, 3]);

// 原地修改 self，前一半部分为 true，后一半为 false，返回 true 元素数量
fn partition_in_place<'a, T, P>(self, predicate: P) -> usize where T: 'a, Self: Sized + DoubleEndedIterator<Item = &'a mut T>, P: FnMut(&T) -> bool { ... }
#![feature(iter_partition_in_place)]
let mut a = [1, 2, 3, 4, 5, 6, 7];
// Partition in-place between evens and odds
let i = a.iter_mut().partition_in_place(|&n| n % 2 == 0);
assert_eq!(i, 3);
assert!(a[..i].iter().all(|&n| n % 2 == 0)); // evens
assert!(a[i..].iter().all(|&n| n % 2 == 1)); // odds

// 返回 self 是否按照 predicate 排序
fn is_partitioned<P>(self, predicate: P) -> bool where Self: Sized, P: FnMut(Self::Item) -> bool { ... }
#![feature(iter_is_partitioned)]
assert!("Iterator".chars().is_partitioned(char::is_uppercase));
assert!(!"IntoIterator".chars().is_partitioned(char::is_uppercase));

fn try_fold<B, F, R>(&mut self, init: B, f: F) -> R where Self: Sized, F: FnMut(B, Self::Item) -> R, R: Try<Output = B> { ... }
fn try_for_each<F, R>(&mut self, f: F) -> R where Self: Sized, F: FnMut(Self::Item) -> R, R: Try<Output = ()> { ... }

// 将迭代器值按照 F 进行聚合，返回最后的结果
fn fold<B, F>(self, init: B, f: F) -> B where Self: Sized, F: FnMut(B, Self::Item) -> B { ... }
let a = [1, 2, 3];
// the sum of all of the elements of the array
let sum = a.iter().fold(0, |acc, x| acc + x);
assert_eq!(sum, 6);

// 和 fold 类似，但是使用第一个值作为初始值
fn reduce<F>(self, f: F) -> Option<Self::Item> where Self: Sized, F: FnMut(Self::Item, Self::Item) -> Self::Item { ... }
let reduced: i32 = (1..10).reduce(|acc, e| acc + e).unwrap();
assert_eq!(reduced, 45);
// Which is equivalent to doing it with `fold`:
let folded: i32 = (1..10).fold(0, |acc, e| acc + e);
assert_eq!(reduced, folded);

fn try_reduce<F, R>( &mut self, f: F ) -> <<R as Try>::Residual as Residual<Option<<R as Try>::Output>>>::TryType where Self: Sized, F: FnMut(Self::Item, Self::Item) -> R, R: Try<Output = Self::Item>, <R as Try>::Residual: Residual<Option<Self::Item>> { ... }

// 迭代的所有元素满足 f
fn all<F>(&mut self, f: F) -> bool where Self: Sized, F: FnMut(Self::Item) -> bool { ... }

// 迭代的任一元素满足 f
fn any<F>(&mut self, f: F) -> bool where Self: Sized, F: FnMut(Self::Item) -> bool { ... }

// 返回 predicate 返回 true 的元素；对比：position() 返回元素的 index
fn find<P>(&mut self, predicate: P) -> Option<Self::Item> where Self: Sized, P: FnMut(&Self::Item) -> bool { ... }
let a = [1, 2, 3];
let mut iter = a.iter();
assert_eq!(iter.find(|&&x| x == 2), Some(&2));
// we can still use `iter`, as there are more elements.
assert_eq!(iter.next(), Some(&3));

// 对迭代器元素执行 f，返回第一个非 None 的结果 Option，等效于 iter.filter_map(f).next().
fn find_map<B, F>(&mut self, f: F) -> Option<B> where Self: Sized, F: FnMut(Self::Item) -> Option<B> { ... }
let a = ["lol", "NaN", "2", "5"];
let first_number = a.iter().find_map(|s| s.parse().ok());
assert_eq!(first_number, Some(2));

fn try_find<F, R>( &mut self, f: F ) -> <<R as Try>::Residual as Residual<Option<Self::Item>>>::TryType where Self: Sized, F: FnMut(&Self::Item) -> R, R: Try<Output = bool>, <R as Try>::Residual: Residual<Option<Self::Item>> { ... }

// 查找满足 predicate 的元素，返回他的 index。对比： find() 返回元素本身。
fn position<P>(&mut self, predicate: P) -> Option<usize> where Self: Sized, P: FnMut(Self::Item) -> bool { ... }
let a = [1, 2, 3];
assert_eq!(a.iter().position(|&x| x == 2), Some(1));
assert_eq!(a.iter().position(|&x| x == 5), None);

fn rposition<P>(&mut self, predicate: P) -> Option<usize> where P: FnMut(Self::Item) -> bool, Self: Sized + ExactSizeIterator + DoubleEndedIterator { ... }

fn max(self) -> Option<Self::Item> where Self: Sized, Self::Item: Ord { ... }
fn min(self) -> Option<Self::Item> where Self: Sized, Self::Item: Ord { ... }

// 根据 f 闭包返回的结果来找最大值
fn max_by_key<B, F>(self, f: F) -> Option<Self::Item> where B: Ord, Self: Sized, F: FnMut(&Self::Item) -> B { ... }
let a = [-3_i32, 0, 1, 5, -10];
assert_eq!(*a.iter().max_by_key(|x| x.abs()).unwrap(), -10);

// 根据  compare 函数的返回值来找最大值
fn max_by<F>(self, compare: F) -> Option<Self::Item> where Self: Sized, F: FnMut(&Self::Item, &Self::Item) -> Ordering { ... }
let a = [-3_i32, 0, 1, 5, -10];
assert_eq!(*a.iter().max_by(|x, y| x.cmp(y)).unwrap(), 5);

fn min_by_key<B, F>(self, f: F) -> Option<Self::Item> where B: Ord, Self: Sized, F: FnMut(&Self::Item) -> B { ... }
fn min_by<F>(self, compare: F) -> Option<Self::Item> where Self: Sized, F: FnMut(&Self::Item, &Self::Item) -> Ordering { ... }

// 返回迭代器的反向迭代器
fn rev(self) -> Rev<Self> where Self: Sized + DoubleEndedIterator { ... }
let a = [1, 2, 3];
let mut iter = a.iter().rev();
assert_eq!(iter.next(), Some(&3));
assert_eq!(iter.next(), Some(&2));
assert_eq!(iter.next(), Some(&1));
assert_eq!(iter.next(), None);

// 迭代器本身迭代返回 (A,B), 然后返回两个分别是 A 、B 聚合后的对象
fn unzip<A, B, FromA, FromB>(self) -> (FromA, FromB) where FromA: Default + Extend<A>, FromB: Default + Extend<B>, Self: Sized + Iterator<Item = (A, B)> { ... }
let a = [(1, 2), (3, 4), (5, 6)];
let (left, right): (Vec<_>, Vec<_>) = a.iter().cloned().unzip();
assert_eq!(left, [1, 3, 5]);
assert_eq!(right, [2, 4, 6]);
// you can also unzip multiple nested tuples at once
let a = [(1, (2, 3)), (4, (5, 6))];
let (x, (y, z)): (Vec<_>, (Vec<_>, Vec<_>)) = a.iter().cloned().unzip();
assert_eq!(x, [1, 4]);
assert_eq!(y, [2, 5]);
assert_eq!(z, [3, 6]);

// 使用元素的 copy 对象来返回一个新的迭代器，特别适合从 &T 返回 T
fn copied<'a, T>(self) -> Copied<Self> where T: 'a + Copy, Self: Sized + Iterator<Item = &'a T> { ... }
let a = [1, 2, 3];
let v_copied: Vec<_> = a.iter().copied().collect();
// copied is the same as .map(|&x| x)
let v_map: Vec<_> = a.iter().map(|&x| x).collect();
assert_eq!(v_copied, vec![1, 2, 3]);
assert_eq!(v_map, vec![1, 2, 3]);

// 使用元素的 clone 对象来返回一个新的迭代器，特别适合 从 &T 返回 T
fn cloned<'a, T>(self) -> Cloned<Self> where T: 'a + Clone, Self: Sized + Iterator<Item = &'a T> { ... }
let a = [1, 2, 3];
let v_cloned: Vec<_> = a.iter().cloned().collect();
// cloned is the same as .map(|&x| x), for integers
let v_map: Vec<_> = a.iter().map(|&x| x).collect();
assert_eq!(v_cloned, vec![1, 2, 3]);
assert_eq!(v_map, vec![1, 2, 3]);

// 循环返回迭代器的元素
fn cycle(self) -> Cycle<Self> where Self: Sized + Clone { ... }
let a = [1, 2, 3];
let mut it = a.iter().cycle();
assert_eq!(it.next(), Some(&1));
assert_eq!(it.next(), Some(&2));
assert_eq!(it.next(), Some(&3));
assert_eq!(it.next(), Some(&1));
assert_eq!(it.next(), Some(&2));
assert_eq!(it.next(), Some(&3));
assert_eq!(it.next(), Some(&1));

fn array_chunks<const N: usize>(self) -> ArrayChunks<Self, N> where Self: Sized { ... }

// 返回元素的 sum，可能会 panic。Option/Result 也实现了 Sum
fn sum<S>(self) -> S where Self: Sized, S: Sum<Self::Item> { ... }
let a = [1, 2, 3];
let sum: i32 = a.iter().sum();
assert_eq!(sum, 6);

// 返回元素的乘积
fn product<P>(self) -> P where Self: Sized, P: Product<Self::Item> { ... }
fn factorial(n: u32) -> u32 {
    (1..=n).product()
}
assert_eq!(factorial(0), 1);
assert_eq!(factorial(1), 1);
assert_eq!(factorial(5), 120);

// 比较两个迭代的各元素， 元素必须实现 Ord trait（所以不能比较 float值）
fn cmp<I>(self, other: I) -> Ordering where I: IntoIterator<Item = Self::Item>, Self::Item: Ord, Self: Sized { ... }
use std::cmp::Ordering;
assert_eq!([1].iter().cmp([1].iter()), Ordering::Equal);
assert_eq!([1].iter().cmp([1, 2].iter()), Ordering::Less);
assert_eq!([1, 2].iter().cmp([1].iter()), Ordering::Greater);

// 使用指定的 cmp 闭包函数来比较两个迭代器的元素
fn cmp_by<I, F>(self, other: I, cmp: F) -> Ordering where Self: Sized, I: IntoIterator, F: FnMut(Self::Item, <I as IntoIterator>::Item) -> Ordering { ... }
#![feature(iter_order_by)]
use std::cmp::Ordering;
let xs = [1, 2, 3, 4];
let ys = [1, 4, 9, 16];
assert_eq!(xs.iter().cmp_by(&ys, |&x, &y| x.cmp(&y)), Ordering::Less);
assert_eq!(xs.iter().cmp_by(&ys, |&x, &y| (x * x).cmp(&y)), Ordering::Equal);
assert_eq!(xs.iter().cmp_by(&ys, |&x, &y| (2 * x).cmp(&y)), Ordering::Greater);

// 与 cmp 相比，partial_cmp可以比较实现 PartialOrd trait 的值，如 float64
fn partial_cmp<I>(self, other: I) -> Option<Ordering> where I: IntoIterator, Self::Item: PartialOrd<<I as IntoIterator>::Item>, Self: Sized { ... }
fn partial_cmp_by<I, F>(self, other: I, partial_cmp: F) -> Option<Ordering> where Self: Sized, I: IntoIterator, F: FnMut(Self::Item, <I as IntoIterator>::Item) -> Option<Ordering> { ... }


// 比较两个迭代器的元素，返回 ture/false
fn eq<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialEq<<I as IntoIterator>::Item>, Self: Sized { ... }
fn eq_by<I, F>(self, other: I, eq: F) -> bool where Self: Sized, I: IntoIterator, F: FnMut(Self::Item, <I as IntoIterator>::Item) -> bool { ... }
fn ne<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialEq<<I as IntoIterator>::Item>, Self: Sized { ... }
fn lt<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<<I as IntoIterator>::Item>, Self: Sized { ... }
fn le<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<<I as IntoIterator>::Item>, Self: Sized { ... }
fn gt<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<<I as IntoIterator>::Item>, Self: Sized { ... }
fn ge<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<<I as IntoIterator>::Item>, Self: Sized { ... }

// 判断迭代器的元素是否已排序
fn is_sorted(self) -> bool where Self: Sized, Self::Item: PartialOrd { ... }
fn is_sorted_by<F>(self, compare: F) -> bool where Self: Sized, F: FnMut(&Self::Item, &Self::Item) -> bool { ... }
fn is_sorted_by_key<F, K>(self, f: F) -> bool where Self: Sized, F: FnMut(Self::Item) -> K, K: PartialOrd { ... }
}
```


### <span class="section-num">19.2</span> std::iter::IntoIterator {#std-iter-intoiterator}

实现该 trait 的对象，它的 into_iter() 方法返回一个迭代器 Iterator：

-   into_iter(self) 方法将对象所有权转义到了返回 Iterator。

<!--listend-->

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;

    // Required method
    fn into_iter(self) -> Self::IntoIter;
}
```


### <span class="section-num">19.3</span> for-in 迭代 {#for-in-迭代}

标准库的如下类型支持 for-in 迭代：

1.  array: [T;N];
2.  动态数组：Vec&lt;T&gt;;
3.  Hash 表：HashMap;
4.  切片引用：&amp;[T]; ([T] 不支持迭代)

当迭代它们的引用类型时，返回的元素是引用类型：

```rust
fn extend(vec: &mut Vec<f64>, slice: &[f64]) {
  for elt in slice { // slice 是切片引用
     vec.push(*elt); // 迭代产生的元素 elt 是 &f64 类型引用，需要 * 解引用
  }
}

// v should have at least one element.
fn smallest(v: &[i32]) -> &i32 {
  let mut s = &v[0];  // v 虽然是引用类型，v[0] 返回值类型是 i32
  for r in &v[1..] { // &v[1..] 返回切片引用，对引用进行迭代，结果 r 还是引用，所以需要 *r 来获得 r 的值。
    if *r < *s { s = r; }
  }
  s
}

fn show(table: &Table) {
  for (artist, works) in table { // 迭代引用时，结果元素 artist 和 works 都是引用类型
    println!("works by {}:", artist); // 宏函数自动解引用
    for work in works { // 迭代引用
            println!("  {}", work); // work 还是引用，宏函数自动解引用
    }
  }
}
```

这是因为 HashMap 等 `为 &HashMap/&mut HashMap/HashMap 类型分别实现` 了对应的 IntoIterator：

```rust
// https://doc.rust-lang.org/src/std/collections/hash/map.rs.html#2173-2182
#[stable(feature = "rust1", since = "1.0.0")]
impl<'a, K, V, S> IntoIterator for &'a HashMap<K, V, S> {
    type Item = (&'a K, &'a V);
    type IntoIter = Iter<'a, K, V>;   // Iter 是 HashMap 定义的实现 Iterator trait 的类型

    #[inline]
    #[rustc_lint_query_instability]
    fn into_iter(self) -> Iter<'a, K, V> {
        self.iter()
    }
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<'a, K, V, S> IntoIterator for &'a mut HashMap<K, V, S> {
    type Item = (&'a K, &'a mut V);
    type IntoIter = IterMut<'a, K, V>;  // IterMut 是 HashMap 定义的实现 Iterator trait 的类型

    #[inline]
    #[rustc_lint_query_instability]
    fn into_iter(self) -> IterMut<'a, K, V> {
        self.iter_mut()
    }
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<K, V, S> IntoIterator for HashMap<K, V, S> {
    type Item = (K, V);
    type IntoIter = IntoIter<K, V>; // IntoIter 是 HashMap 定义的实现 Iterator trait 的类型

    /// Creates a consuming iterator, that is, one that moves each key-value
    /// pair out of the map in arbitrary order. The map cannot be used after
    /// calling this.
    ///
    /// # Examples
    ///
    /// ```
    /// use std::collections::HashMap;
    ///
    /// let map = HashMap::from([
    ///     ("a", 1),
    ///     ("b", 2),
    ///     ("c", 3),
    /// ]);
    ///
    /// // Not possible with .iter()
    /// let vec: Vec<(&str, i32)> = map.into_iter().collect();
    /// ```
    #[inline]
    #[rustc_lint_query_instability]
    fn into_iter(self) -> IntoIter<K, V> {
        IntoIter { base: self.base.into_iter() }
    }
}

// file:///Users/zhangjun/.rustup/toolchains/nightly-x86_64-apple-darwin/share/doc/rust/html/src/std/collections/hash/map.rs.html#2223
// HashMap 的 Iter 实现了 Iterator
#[stable(feature = "rust1", since = "1.0.0")]
impl<'a, K, V> Iterator for Iter<'a, K, V> {
    type Item = (&'a K, &'a V);

    #[inline]
    fn next(&mut self) -> Option<(&'a K, &'a V)> {
        self.base.next()
    }
    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        self.base.size_hint()
    }
}
```

rust 为实现了 Iterator 的类型自动实现了 IntoIterator：impl&lt;I&gt; IntoIterator for I where I: Iterator，所以可以使用:

```text
for i in vec.iter() {println!("{i}")};
```

注意：

1.  String 和 &amp;String 都是不支持迭代的，但是它的部分方法的返回类型支持迭代：
    -   pub fn as_bytes(&amp;self) -&gt; &amp;[u8]
    -   pub unsafe fn as_mut_vec(&amp;mut self) -&gt; &amp;mut Vec&lt;u8, Global&gt;
    -   pub fn as_str(&amp;self) -&gt; &amp;str
    -   pub fn into_bytes(self) -&gt; Vec&lt;u8, Global&gt;
2.  &amp;str 也不支持迭代，它的大量方法返回的类型支持迭代：
    -   pub const fn as_bytes(&amp;self) -&gt; &amp;[u8]
    -   pub unsafe fn as_bytes_mut(&amp;mut self) -&gt; &amp;mut [u8]
    -   pub fn bytes(&amp;self) -&gt; Bytes&lt;'_&gt;
    -   pub fn split_whitespace(&amp;self) -&gt; SplitWhitespace&lt;'_&gt;
    -   pub fn char_indices(&amp;self) -&gt; CharIndices&lt;'_&gt;
    -   pub fn chars(&amp;self) -&gt; Chars&lt;'_&gt;
    -   pub fn lines(&amp;self) -&gt; Lines&lt;'_&gt;
    -   pub fn matches&lt;'a, P&gt;(&amp;'a self, pat: P) -&gt; Matches&lt;'a, P&gt;


### <span class="section-num">19.4</span> std::iter::FromIterator {#std-iter-fromiterator}

从输入的迭代器 iter 创建一个 Self 对象（取决于实现该 Fromiterator 的对象类型）

-   输入的 iter 是 IntoIterator 类型，所以会转义 iter 对象的所有权。

迭代器的泛型方法 collect&lt;T&gt;() 会自动调用 T::from_iter(self) 方法来生成 T 对象。

```rust
// file:///Users/zhangjun/.rustup/toolchains/nightly-x86_64-apple-darwin/share/doc/rust/html/std/iter/trait.FromIterator.html
pub trait FromIterator<A>: Sized {
    // Required method
    fn from_iter<T>(iter: T) -> Self
       where T: IntoIterator<Item = A>;
}

// 示例
let five_fives = std::iter::repeat(5).take(5);
let v = Vec::from_iter(five_fives);
assert_eq!(v, vec![5, 5, 5, 5, 5]);

// collect() 函数默认自动使用 FromIterator<A> trait
let five_fives = std::iter::repeat(5).take(5);
let v: Vec<i32> = five_fives.collect(); // 等效于：Vec<i32>::from_iter(five_fives)
assert_eq!(v, vec![5, 5, 5, 5, 5]);
```

实现 FromIterator&lt;T&gt; 的类型：

-   impl&lt;K, V&gt; FromIterator&lt;(K, V)&gt; for BTreeMap&lt;K, V&gt;
-   impl&lt;T&gt; FromIterator&lt;T&gt; for BTreeSet&lt;T&gt; where T: Ord,
-   impl&lt;T&gt; FromIterator&lt;T&gt; for Vec&lt;T&gt;

<!--listend-->

```rust
// file:///Users/zhangjun/.rustup/toolchains/nightly-x86_64-apple-darwin/share/doc/rust/html/std/iter/trait.FromIterator.html#implementors
impl FromIterator<char> for String
impl FromIterator<()> for ()

use std::io::*;
let data = vec![1, 2, 3, 4, 5];
let res: Result<()> = data.iter()
    .map(|x| writeln!(stdout(), "{x}"))
    .collect();
assert!(res.is_ok());

impl FromIterator<Box<str>> for String
impl FromIterator<OsString> for OsString
impl FromIterator<String> for String
impl<'a> FromIterator<&'a char> for String
impl<'a> FromIterator<&'a str> for String
impl<'a> FromIterator<&'a OsStr> for OsString
impl<'a> FromIterator<Cow<'a, str>> for String
impl<'a> FromIterator<Cow<'a, OsStr>> for OsString
impl<'a> FromIterator<char> for Cow<'a, str>
impl<'a> FromIterator<String> for Cow<'a, str>
impl<'a, 'b> FromIterator<&'b str> for Cow<'a, str>
impl<'a, T> FromIterator<T> for Cow<'a, [T]>
where
    T: Clone,

impl<A, E, V> FromIterator<Result<A, E>> for Result<V, E>
where
    V: FromIterator<A>,

impl<A, V> FromIterator<Option<A>> for Option<V>
where
    V: FromIterator<A>,

impl<I> FromIterator<I> for Box<[I]>

impl<K, V> FromIterator<(K, V)> for BTreeMap<K, V>
where
    K: Ord,

impl<K, V, S> FromIterator<(K, V)> for HashMap<K, V, S>
where
    K: Eq + Hash,
    S: BuildHasher + Default,

impl<P: AsRef<Path>> FromIterator<P> for PathBuf

impl<T> FromIterator<T> for BTreeSet<T>
where
    T: Ord,

impl<T> FromIterator<T> for BinaryHeap<T>
where
    T: Ord,

impl<T> FromIterator<T> for LinkedList<T>

impl<T> FromIterator<T> for VecDeque<T>

impl<T> FromIterator<T> for Rc<[T]>

impl<T> FromIterator<T> for Arc<[T]>

impl<T> FromIterator<T> for Vec<T>

impl<T, S> FromIterator<T> for HashSet<T, S>
where
    T: Eq + Hash,
    S: BuildHasher + Default,
impl FromIterator<TokenStream> for TokenStream
impl FromIterator<TokenTree> for TokenStream
```


## <span class="section-num">20</span> crate/module/pakckage {#crate-module-pakckage}

crate 是 rust 的编译单元: rustc some_file.rs 中的 some_file.rs 是 crate file.

-   crate 可以被编译为 binary 或 library, 可以通过 --crate-type=lib/bin 来定义;

<!--listend-->

```shell
  # A binary
  cargo new foo

  # A library
  cargo new --lib bar

  .
  ├── bar
  │   ├── Cargo.toml
  │   └── src
  │       └── lib.rs
  └── foo
      ├── Cargo.toml
      └── src
          └── main.rs

  # bin 目录下的文件名分别为单独的 binary, 可以通过 cargo 的 --bin my_other_bin 来指定
  foo
  ├── Cargo.toml
  └── src
      ├── main.rs
      └── bin
          └── my_other_bin.rs

foo
├── Cargo.toml
├── src
│   └── main.rs
│   └── lib.rs
└── tests # 集成测试
    ├── my_test.rs
    └── my_other_test.rs

```

crate file 可以使用 mod 声明来引用其他 mod 中 item 对象, 该 mod 可以是单独的文件或当前文件中定义,
mod 可以嵌套.

父 module 中的元素, 不管是否 public, 都可以在自身和子 module 中使用. 但是不能在父 module(递归向上)
和其它非子 module 中使用.

在 Rust 2015 以后版本, 基本上不需要再使用 extern crate xx 了, 因为 rustc 会从 Cargo.toml 中获得外部依赖的 crate.

module 引入了一级 namespace, 其中的 item 需要使用 module::item 来访问.

module 中 item 的可见性

-   pub fn
-   pub (in path): pub(in crate::my_mod), 对指定的 crate 开放;
-   pub (self):: 只对当前 module 开放; 等效于不加 pub;
-   pub (super):: 只对父 module 开放;
-   pub(crate):: 只对当前 crate 开放;

<!--listend-->

```rust
  // A module named `my_mod`
  mod my_mod {
      // Items in modules default to private visibility.
      fn private_function() {
          println!("called `my_mod::private_function()`");
      }

      // Use the `pub` modifier to override default visibility.
      pub fn function() {
          println!("called `my_mod::function()`");
      }

      // Items can access other items in the same module, even when private.
      pub fn indirect_access() {
          print!("called `my_mod::indirect_access()`, that\n> ");
          private_function();
      }

      // Modules can also be nested
      pub mod nested {
          pub fn function() {
              println!("called `my_mod::nested::function()`");
          }

          #[allow(dead_code)]
          fn private_function() {
              println!("called `my_mod::nested::private_function()`");
          }

          // Functions declared using `pub(in path)` syntax are only visible within the given
          // path. `path` must be a parent or ancestor module
          pub(in crate::my_mod) fn public_function_in_my_mod() {
              print!("called `my_mod::nested::public_function_in_my_mod()`, that\n> ");
              public_function_in_nested();
          }

          // Functions declared using `pub(self)` syntax are only visible within the current module,
          // which is the same as leaving them private
          pub(self) fn public_function_in_nested() {
              println!("called `my_mod::nested::public_function_in_nested()`");
          }

          // Functions declared using `pub(super)` syntax are only visible within
          // the parent module
          pub(super) fn public_function_in_super_mod() {
              println!("called `my_mod::nested::public_function_in_super_mod()`");
          }
      }

      pub fn call_public_function_in_my_mod() {
          print!("called `my_mod::call_public_function_in_my_mod()`, that\n> ");
          nested::public_function_in_my_mod();
          print!("> ");
          nested::public_function_in_super_mod();
      }

      // pub(crate) makes functions visible only within the current crate
      pub(crate) fn public_function_in_crate() {
          println!("called `my_mod::public_function_in_crate()`");
      }

      // Nested modules follow the same rules for visibility
      mod private_nested {
          #[allow(dead_code)]
          pub fn function() {
              println!("called `my_mod::private_nested::function()`");
          }

          // Private parent items will still restrict the visibility of a child item,
          // even if it is declared as visible within a bigger scope.
          #[allow(dead_code)]
          pub(crate) fn restricted_function() {
              println!("called `my_mod::private_nested::restricted_function()`");
          }
      }
  }

  fn function() {
      println!("called `function()`");
  }

  fn main() {
      // Modules allow disambiguation between items that have the same name.
      function();
      my_mod::function();

      // Public items, including those inside nested modules, can be
      // accessed from outside the parent module.
      my_mod::indirect_access();
      my_mod::nested::function();
      my_mod::call_public_function_in_my_mod();

      // pub(crate) items can be called from anywhere in the same crate
      my_mod::public_function_in_crate();
  }
```

package 目录结构：

```rust
# Create a package which contains
# 1. three binary crates: `hello-package`, `main1` and `main2`
# 2. one library crate
# describe the directory tree below
.
├── Cargo.toml
├── Cargo.lock
├── src
│   ├── __
│   ├── __
│   └── __
│       └── __
│       └── __
├── tests # directory for integrated tests files
│   └── some_integration_tests.rs
├── benches # dir for benchmark files
│   └── simple_bench.rs
└── examples # dir for example files
    └── simple_example.rs
```

可以使用 self 或 super 来访问当前 module 或父 module 的 item:

```rust
fn function() {
    println!("called `function()`");
}

mod cool {
    pub fn function() {
        println!("called `cool::function()`");
    }
}

mod my {
    fn function() {
        println!("called `my::function()`");
    }

    mod cool {
        pub fn function() {
            println!("called `my::cool::function()`");
        }
    }

    pub fn indirect_call() {
        // Let's access all the functions named `function` from this scope!
        print!("called `my::indirect_call()`, that\n> ");

        // The `self` keyword refers to the current module scope - in this case `my`.
        // Calling `self::function()` and calling `function()` directly both give
        // the same result, because they refer to the same function.
        self::function();  // self 表示当前 module
        function();

        // We can also use `self` to access another module inside `my`:
        self::cool::function();  // 当前 module 的子 module cool

        // The `super` keyword refers to the parent scope (outside the `my` module).
        super::function();

        // This will bind to the `cool::function` in the *crate* scope.
        // In this case the crate scope is the outermost scope.
        {
            use crate::cool::function as root_function; // create 表示当前 module 所在的 create
            root_function();
        }
    }
}

fn main() {
    my::indirect_call();
}
```

use 语句: 将某个标识符和一个 full path 绑定, 后续可以直接使用标识符:

-   use xx::yy 中的 xx 是相对于当前 module 的, 可以是子 module 或 item, 如果都不存在则 xx 是crate;
-   use self::item 导入当前 module 的 item;
-   yse super::item 导入父 module 的 item;
-   use crate::module::item 中的 crate 表示当前 module 所在的 crate;
-   开头表示使用外部 crate image.

<!--listend-->

```rust
  use crate::deeply::nested::{
      my_first_function,
      my_second_function,
      AndATraitType
  };

  fn main() {
      my_first_function();
  }


  // 使用 use..as 对绑定重命名

  // Bind the `deeply::nested::function` path to `other_function`.
  use deeply::nested::function as other_function;
  fn function() {
      println!("called `function()`");
  }
  mod deeply {
      pub mod nested {
          pub fn function() {
              println!("called `deeply::nested::function()`");
          }
      }
  }
  fn main() {
      // Easier access to `deeply::nested::function`
      other_function();

      println!("Entering block");
      {
          // This is equivalent to `use deeply::nested::function as function`.
          // This `function()` will shadow the outer one.
          use crate::deeply::nested::function;

          // `use` bindings have a local scope. In this case, the
          // shadowing of `function()` is only in this block.
          function();

          println!("Leaving block");
      }

      function();
  }


  // 使用 pub use 将绑定在当前 create module 重新 export
  pub use deeply::nested::function as other_function;
```


## <span class="section-num">21</span> attribute {#attribute}

attribute 有两种形式:

1.  \#[outer_attribute], 直接对紧接者的 item 有效;
2.  \#\![inner_attribute], 对 enclosing item, 一般是 module 或 crate

<!--listend-->

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

#![allow(unused_variables)]
fn main() {
    let x = 3; // This would normally warn about an unused variable.
}
```

attribute 可以带参数:

-   \#[attribute = "value"]
-   \#[attribute(key = "value")]
-   \#[attribute(value)]
-   \#[attribute(value, value2)]
-   \#[attribute(value, value2, value3, value4, value5)]


### <span class="section-num">21.1</span> 常见 attribute {#常见-attribute}

1.  nightly-only experimental API： 需要使用 #\![feature(API_NAME)] 来启用 nightly 实验性 APIs：
    -   需要安装 nightly toolchain；

<!--listend-->

```rust
  #![feature(iter_next_chunk)]
  let mut iter = "lorem".chars();
  assert_eq!(iter.next_chunk().unwrap(), ['l', 'o']);              // N is inferred as 2
  assert_eq!(iter.next_chunk().unwrap(), ['r', 'e', 'm']);         // N is inferred as 3
  assert_eq!(iter.next_chunk::<4>().unwrap_err().as_slice(), &[]); // N is explicitly 4

  let quote = "not all those who wander are lost";
  let [first, second, third] = quote.split_whitespace().next_chunk().unwrap();
  assert_eq!(first, "not");
  assert_eq!(second, "all");
  assert_eq!(third, "those");
```

1.  \#[allow(dead_code)]: 运行未使用的代码(如变量声明, 函数定义).

<!--listend-->

```rust
  // Nested modules follow the same rules for visibility
  mod private_nested {
      #[allow(dead_code)]
      pub fn function() {
          println!("called `my_mod::private_nested::function()`");
      }
  }
```

1.  \#\![allow(unused_variables)]:

<!--listend-->

```rust
#![allow(unused_variables)]
fn main() {
    let x = 3; // This would normally warn about an unused variable.
}
```

1.  \#[cfg(...)] 条件编译:

<!--listend-->

```rust
  // This function only gets compiled if the target OS is linux
  #[cfg(target_os = "linux")]
  fn are_you_on_linux() {
      println!("You are running linux!");
  }

  // And this function only gets compiled if the target OS is *not* linux
  #[cfg(not(target_os = "linux"))]
  fn are_you_on_linux() {
      println!("You are *not* running linux!");
  }

  fn main() {
      are_you_on_linux();

      println!("Are you sure?");
      if cfg!(target_os = "linux") {
          println!("Yes. It's definitely linux!");
      } else {
          println!("Yes. It's definitely *not* linux!");
      }
  }

  // rustc --cfg some_condition custom.rs && ./custom
  #[cfg(some_condition)]  // 自定义 cfg
  fn conditional_function() {
      println!("condition met!");
  }

  fn main() {
      conditional_function();
  }
```


## <span class="section-num">22</span> panic! {#panic}

panic 是最简单的异常处理机制，它打印 error message，然后开始 unwinding stack，最后退出当前 thread：

1.  如果是 main thread panic，则程序退出；
2.  否则，如果是 spawned thread panic，则该 thread 会终止，程序不退出。

unwinding stack 过程中，rust 会回溯调用栈，drop 所有的对象和资源。也可以在 Cargo.toml 里设置 panic
时不 unwiding stack 而是直接 abort 退出：

```toml
[profile.release]
panic = 'abort'
```


## <span class="section-num">23</span> 常见方法和 trait {#常见方法和-trait}


### <span class="section-num">23.1</span> make_mut {#make-mut}

`pub fn make_mut(this: &mut Arc<T, A>) -> &mut T`

Makes a mutable reference into the given Arc.

If there are other Arc pointers to the same allocation, then make_mut will `clone the inner value` to
a new allocation to `ensure unique ownership`. This is also referred to as `clone-on-write`.

However, if there are no other Arc pointers to this allocation, but some Weak pointers, then the
Weak pointers will be dissociated and the inner value will not be cloned.

See also `get_mut`, which will fail rather than cloning the inner value or dissociating Weak pointers.

```rust
use std::sync::Arc;

let mut data = Arc::new(5);

*Arc::make_mut(&mut data) += 1;         // Won't clone anything
let mut other_data = Arc::clone(&data); // Won't clone inner data
*Arc::make_mut(&mut data) += 1;         // Clones inner data
*Arc::make_mut(&mut data) += 1;         // Won't clone anything
*Arc::make_mut(&mut other_data) *= 2;   // Won't clone anything

// Now `data` and `other_data` point to different allocations.
assert_eq!(*data, 8);
assert_eq!(*other_data, 12);
```


### <span class="section-num">23.2</span> get_mut {#get-mut}

`pub fn get_mut(this: &mut Arc<T, A>) -> Option<&mut T>`

get_mut 一般返回 Option.

Returns a mutable reference into the given Arc, if there are no other Arc or Weak pointers to the
same allocation.

Returns `None` otherwise, because it is not safe to mutate a shared value.

See also make_mut, which will clone the inner value when there are other Arc pointers.

Examples

```rust
use std::sync::Arc;

let mut x = Arc::new(3);
*Arc::get_mut(&mut x).unwrap() = 4;
assert_eq!(*x, 4);

let _y = Arc::clone(&x);
assert!(Arc::get_mut(&mut x).is_none());
```


### <span class="section-num">23.3</span> into_inner {#into-inner}

```text
pub fn into_inner(this: Arc<T, A>) -> Option<T>
```

Returns the inner value, if the Arc has exactly one strong reference.

Otherwise, `None is returned and the Arc is dropped`.

This will succeed even if there are outstanding weak references.

If Arc::into_inner is called on every clone of this Arc, it is guaranteed that `exactly one` of the
calls returns the inner value. This means in particular that the inner value is not dropped.

The similar expression Arc::try_unwrap(this).ok() does not offer such a guarantee. See the last
example below and the documentation of Arc::try_unwrap.

Examples

Minimal example demonstrating the guarantee that Arc::into_inner gives.

```rust
use std::sync::Arc;

let x = Arc::new(3);
let y = Arc::clone(&x);

// Two threads calling `Arc::into_inner` on both clones of an `Arc`:
let x_thread = std::thread::spawn(|| Arc::into_inner(x));
let y_thread = std::thread::spawn(|| Arc::into_inner(y));

let x_inner_value = x_thread.join().unwrap();
let y_inner_value = y_thread.join().unwrap();

// One of the threads is guaranteed to receive the inner value:
assert!(matches!(
    (x_inner_value, y_inner_value),
    (None, Some(3)) | (Some(3), None)
));
// The result could also be `(None, None)` if the threads called
// `Arc::try_unwrap(x).ok()` and `Arc::try_unwrap(y).ok()` instead.
```

实现 into_inner() 函数的类型还有:

-   std::boxed::Box  pub fn into_inner(boxed: Box&lt;T, A&gt;) -&gt; T
-


### <span class="section-num">23.4</span> downcase {#downcase}

一般是 `dyn Any` 的智能指针,如 Arc/Rc/Box 等实现了 downcase 方法, 将 trait object 转换为具体的类型:

trait object 的标准定义一般有 3 种形式, 主要是看是否有 Send/Sync 约束. 以 Box 为例:

-   pub fn downcast&lt;T&gt;(self) -&gt; Result&lt;Box&lt;T, A&gt;, Box&lt;dyn Any, A&gt;&gt;
-   pub fn downcast&lt;T&gt;(self) -&gt; Result&lt;Box&lt;T, A&gt;, Box&lt;dyn Any + Send, A&gt;&gt;
-   pub fn downcast&lt;T&gt;(self) -&gt; Result&lt;Box&lt;T, A&gt;, Box&lt;dyn Any + Sync + Send, A&gt;&gt;

<!--listend-->

```rust
impl<A> Box<dyn Any, A>
where
    A: Allocator,
pub fn downcast<T>(self) -> Result<Box<T, A>, Box<dyn Any, A>>
where
    T: Any,


impl<A> Box<dyn Any + Send, A>
where
    A: Allocator,
pub fn downcast<T>(self) -> Result<Box<T, A>, Box<dyn Any + Send, A>>
where
    T: Any,


impl<A> Box<dyn Any + Sync + Send, A>
where
    A: Allocator,
pub fn downcast<T>(self) -> Result<Box<T, A>, Box<dyn Any + Sync + Send, A>>
where
    T: Any,
```

Rc/Arc/Cell/RefCell 等智能指针类型, 均具有这三种类型的 downcase 方法:

```rust
impl<A> Arc<dyn Any + Sync + Send, A>
where
    A: Allocator + Clone,

pub fn downcast<T>(self) -> Result<Arc<T, A>, Arc<dyn Any + Sync + Send, A>>
where
    T: Any + Send + Sync,
```

Attempt to downcast the Arc&lt;dyn Any + Send + Sync&gt; to a concrete type.

Examples

```rust
use std::any::Any;
use std::sync::Arc;

// downcast 方法有自己的类型参数,所以需要使用 比目鱼语法 来指定调用.
fn print_if_string(value: Arc<dyn Any + Send + Sync>) {
    if let Ok(string) = value.downcast::<String>() { // string 类型是 Arc<String>
        println!("String ({}): {}", string.len(), string);
    }
}

let my_string = "Hello World".to_string();
print_if_string(Arc::new(my_string));
print_if_string(Arc::new(0i8));
```


## <span class="section-num">24</span> macro {#macro}

宏可以用来简化重复的代码编写任务、实现特定的 DSL（领域特定语言）、或者进行编译时代码生成。 创建 Vec
的 vec!宏、为数据结构添加各种 trait 支持的 #[derive(Debug, Default, ...)]、条件编译时使用的
\#[cfg(test)] 宏等。

rust macro 可以实现 metaprogramming 范式， 有用的特性：

-   Don't repeat yourself;
-   DSL;
-   Variadic interface： 实现一些普通函数不支持的特性，比如 println!() 的可变长参数列表；

Rust 宏的定义有两类：

1.  声明宏（macro_rules!）：对代码模版做简单替换，比如 vec!、println!等
2.  过程宏（Procedural Macros）：定制并在编译期生成代码。Rust允许用户通过改写代码语法书改写原有逻辑，类似 Java 字节码增强。

声明宏(macro_rules): 宏可用模式匹配接收不同的输入参数，并生成相应的代码。

-   macro_rules! 声明内部是一系列 `匹配模式` 和对应的生成代码;

<!--listend-->

```rust
  macro_rules! say_what {
      ($expression:expr) => {
          println!("You said: {}", $expression);
      };
  }

  fn main() {
      say_what!("I'm learning macros");
  }

  macro_rules! create_function {
      // This macro takes an argument of designator `ident` and
      // creates a function named `$func_name`.
      // The `ident` designator is used for variable/function names.
      ($func_name:ident) => {
          fn $func_name() {
              // The `stringify!` macro converts an `ident` into a string.
              println!("You called {:?}()",
                  stringify!($func_name));
          }
      };
  }

  // Create functions named `foo` and `bar` with the above macro.
  create_function!(foo);
  create_function!(bar);

  macro_rules! print_result {
      // This macro takes an expression of type `expr` and prints
      // it as a string along with its result.
      // The `expr` designator is used for expressions.
      ($expression:expr) => {
          // `stringify!` will convert the expression *as it is* into a string.
          println!("{:?} = {:?}",
              stringify!($expression),
              $expression);
      };
  }

  fn main() {
      foo();
      bar();

      print_result!(1u32 + 1);

      // Recall that blocks are expressions too!
      print_result!({
          let x = 1u32;

          x * x + 2 * x - 1
      });
  }
```

这里的 $expression:expr 是一个捕获表达式，它匹配任何 Rust 表达式，并将其作为参数传递给宏。expr 的类型如下：

-   block
-   expr is used for expressions
-   ident is used for variable/function names
-   item
-   literal is used for literal constants
-   pat (pattern)
-   path
-   stmt (statement)
-   tt (token tree)
-   ty (type)
-   vis (visibility qualifier)

可以定义多个 pattern 来实现 overload：

```rust
  // `test!` will compare `$left` and `$right`
  // in different ways depending on how you invoke it:
  macro_rules! test {
      // Arguments don't need to be separated by a comma.
      // Any template can be used!
      ($left:expr; and $right:expr) => {
          println!("{:?} and {:?} is {:?}",
              stringify!($left),
              stringify!($right),
              $left && $right)
      };

      // ^ each arm must end with a semicolon.
      ($left:expr; or $right:expr) => {
          println!("{:?} or {:?} is {:?}",
              stringify!($left),
              stringify!($right),
              $left || $right)
      };
  }

  fn main() {
      test!(1i32 + 1 == 2i32; and 2i32 * 2 == 4i32);
      test!(true; or false);
  }
```

在参数列表中 macro 使用 $(...),+ 语法来表示重复，在 body 中使用 $($xx),+ 来引用匹配的重复。

```rust
// `find_min!` will calculate the minimum of any number of arguments.
macro_rules! find_min {
    // Base case:
    ($x:expr) => ($x);
    // `$x` followed by at least one `$y,`
    ($x:expr, $($y:expr),+) => (
        // Call `find_min!` on the tail `$y`
        std::cmp::min($x, find_min!($($y),+))
    )
}

fn main() {
    println!("{}", find_min!(1));
    println!("{}", find_min!(1 + 2, 2));
    println!("{}", find_min!(5, 2 * 3, 4));
}
```

使用 macro 来减少重复的例子（DRY）：

```rust
  use std::ops::{Add, Mul, Sub};

  macro_rules! assert_equal_len {
      // The `tt` (token tree) designator is used for
      // operators and tokens.
      ($a:expr, $b:expr, $func:ident, $op:tt) => {
          assert!($a.len() == $b.len(),
              "{:?}: dimension mismatch: {:?} {:?} {:?}",
              stringify!($func),
              ($a.len(),),
              stringify!($op),
              ($b.len(),));
      };
  }

  macro_rules! op {
      ($func:ident, $bound:ident, $op:tt, $method:ident) => {
          fn $func<T: $bound<T, Output=T> + Copy>(xs: &mut Vec<T>, ys: &Vec<T>) {
              assert_equal_len!(xs, ys, $func, $op);

              for (x, y) in xs.iter_mut().zip(ys.iter()) {
                  *x = $bound::$method(*x, *y);
                  // *x = x.$method(*y);
              }
          }
      };
  }

  // Implement `add_assign`, `mul_assign`, and `sub_assign` functions.
  op!(add_assign, Add, +=, add);
  op!(mul_assign, Mul, *=, mul);
  op!(sub_assign, Sub, -=, sub);

  mod test {
      use std::iter;
      macro_rules! test {
          ($func:ident, $x:expr, $y:expr, $z:expr) => {
              #[test]
              fn $func() {
                  for size in 0usize..10 {
                      let mut x: Vec<_> = iter::repeat($x).take(size).collect();
                      let y: Vec<_> = iter::repeat($y).take(size).collect();
                      let z: Vec<_> = iter::repeat($z).take(size).collect();

                      super::$func(&mut x, &y);

                      assert_eq!(x, z);
                  }
              }
          };
      }

      // Test `add_assign`, `mul_assign`, and `sub_assign`.
      test!(add_assign, 1u32, 2u32, 3u32);
      test!(mul_assign, 2u32, 3u32, 6u32);
      test!(sub_assign, 3u32, 2u32, 1u32);
  }
```

使用 macro 实现 DSL：

```rust
macro_rules! calculate {
    (eval $e:expr) => {
        {
            let val: usize = $e; // Force types to be unsigned integers
            println!("{} = {}", stringify!{$e}, val);
        }
    };
}

fn main() {
    calculate! {
        eval 1 + 2 // hehehe `eval` is _not_ a Rust keyword!
    }

    calculate! {
        eval (1 + 2) * (3 / 4)
    }
}
```

使用 macro 实现可变参数：

```rust
macro_rules! calculate {
    // The pattern for a single `eval`
    (eval $e:expr) => {
        {
            let val: usize = $e; // Force types to be integers
            println!("{} = {}", stringify!{$e}, val);
        }
    };

    // Decompose multiple `eval`s recursively
    (eval $e:expr, $(eval $es:expr),+) => {{
        calculate! { eval $e }
        calculate! { $(eval $es),+ }
    }};
}

fn main() {
    calculate! { // Look ma! Variadic `calculate!`!
        eval 1 + 2,
        eval 3 + 4,
        eval (2 * 3) + 1
    }
}
```

过程宏（Procedural Macros）: 过程宏是更高级的宏，它允许你操作Rust代码的抽象语法树（AST）。过程宏有三种类型：

1.  派生宏（derive）：为 derive 属性添加新的功能。这是我们平时使用最多的宏，比如 #[derive(Debug)] 为我们的数据结构提供 Debug trait 的实现
2.  属性宏（attribute-like）：可以在其他代码块上添加属性，为代码块提供更多功能。比如 rocket 的
    get/put 等路由属性。
3.  函数宏（function-like）：类函数宏可以让我们定义像函数那样调用的宏。

derive 宏: derive宏允许你为任何struct/enum/union `自动实现特定的trait` 。

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
}

fn main() {
    let user = User {
        name: String::from("Alice"),
        age: 30,
    };
    println!("{:?}", user);
}

// 他的实现方式类似下面
#[proc_macro_derive(Debug)]
pub fn debug_derive (input: TokenStream) -> TokenStream {
    // 基于 Input 构建 AST 语法树
}
```

属性宏: 属性宏类似于derive宏，但它们不仅限于实现trait。derive宏只能用于结构体或枚举类型，属性宏可以用在其他类型上，比如函数。假设我们在开发一个 web 框架，当用户通过 HTTP GET 请求访问 / 根路径时，使用
index 函数为其提供服务:

```rust
#[route(GET, "/")]
fn index() {

// 如上所示，代码功能非常清晰、简洁，这里的 #[route] 属性就是一个过程宏，它的定义函数大概如下：
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
}
```

与 derive 宏不同，类属性宏的定义函数有两个参数：

1.  第一个参数时用于说明属性包含的内容：Get, "/" 部分
2.  第二个是属性所标注的类型项，在这里是 fn index() {...}，注意，函数体也被包含其中

函数宏: 函数宏看起来和普通的Rust函数类似，但它们可以接受和返回Rust的token。

```rust
#[proc_macro]
pub fn define_function_macro(input: TokenStream) -> TokenStream {}

// 使用形式类似函数调用
let result = define_function_macro!("this is a paramater");
```

目前过程宏只能定义在单独的 package 中，宏的包名也要以 derive 结尾，运行以下命令创建单独的 lib 包：

-   crate type 是特殊的 proc_macro (另外的类型是 bin/lib), 通过抵债 Cargo.toml 中定义
    lib.proc-macro=true 来标识该 crate 是 proc-macro create type.
-   编译器提供的 crate type 类型: <https://doc.rust-lang.org/reference/linkage.html>
-   RFC 参考: <https://rust-lang.github.io/rfcs/1566-proc-macros.html>

<!--listend-->

```shell
.
├── Cargo.lock
├── Cargo.toml
├── hello_macro_derive
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
└── src
    ├── lib.rs
    └── main.rs
```

为何要在单独的 crate package 中[定义过程宏的解答](https://users.rust-lang.org/t/why-do-we-need-procedural-macro-separation-on-a-crate-level/107295/4): A proc macro has to `be compiled for the host` such
that rustc can dlopen them, but non-proc macro crates have to `be compiled for the target` such that
they can be linked into the target executable or dynamic library. When host can be different from
the target, this implies that the proc macro and non-proc macro definitions have to be in separate
crates.

修改 hello_macro/Cargo.toml 文件，方便在 src/main.rs 引用 hello_macro_derive 包的内容：

```toml
[dependencies]
hello_macro_derive = { path = "../hello_macro/hello_macro_derive" }
# 也可以使用下面的相对路径
# hello_macro_derive = { path = "./hello_macro_derive" }
```

定义过程宏: 在 hello_macro_derive/Cargo.toml 文件中添加以下内容：

```toml
[lib]
proc-macro = true # 链接 rustc 工具链提供的 proc-macro 库 libproc_macro

[dependencies] # 定义过程宏依赖的包
syn = "1.0"
quote = "1.0"
```

libproc_macro 是随 rustc 编译器一起提供和安装的 proc_macro create 提供的库:

-   同时 {toolchain}/libexec/rust-analyzer-proc-macro-srv 用于 rust-analyzer 来对 proc macro 进行
    expand 分析(也即获得 proc macro 展开后的代码).
-   正常情况下 rustc 提供了 proc-macro server 的功能, 但是 rust-analyzer 也提供了该功能:
    <https://fasterthanli.me/articles/proc-macro-support-in-rust-analyzer-for-nightly-rustc-versions>

<!--listend-->

```shell
zj@a:~/docs$ ls -l ~/.rustup/toolchains/nightly-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libproc_macro-ce17747687ef7ea0.rlib
-rw-r--r-- 1 zhangjun 4.5M  3  4 19:09 /Users/zhangjun/.rustup/toolchains/nightly-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libproc_macro-ce17747687ef7ea0.rlib

zj@a:~/docs$ ls -l ~/.rustup/toolchains/nightly-x86_64-apple-darwin/libexec/
total 1.4M
-rwxr-xr-x 1 zhangjun 1.4M  3  4 19:09 rust-analyzer-proc-macro-srv*

# The main rust-analyzer process and the proc-macro server communicate over a JSON interface
# https://fasterthanli.me/articles/proc-macro-support-in-rust-analyzer-for-nightly-rustc-versions
# libpm-e886d9f9eaf24619.so 是编译后的 proc-macro crate lib
$ echo '{"ListMacros":{"dylib_path":"target/debug/deps/libpm-e886d9f9eaf24619.so"}}' | ~/.cargo/bin/rust-analyzer proc-macro
{"ListMacros":{"Ok":[["do_thrice","FuncLike"]]}}
```

由于 rustc 内置的 proc_macro create 提供的 API 只能在 procedure macro 类型 crate 中使用, 所以不能在
build.rs 和 main.rs 等场景中使用它们, 同时也不能用来进行测试。所以社区引入了 [proc-macro2 create](https://github.com/dtolnay/proc-macro2%20%20%20)，它是 rustc 内置 proc_macro create 的封装(wrapper), 主要提供如下两个特性:

1.  Bring proc-macro-like functionality to other contexts like build.rs and main.rs.
2.  Make procedural macros unit testable.

proc-macro2 在 serde/tokio-marcros 等 project 中的带广泛使用。

其次，在 hello_macro_derive/src/lib.rs 中添加如下代码：

```rust
use proc_macro::TokenStream;
use syn;
use syn::DeriveInput;
use quote::quote;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // 基于 Input 构建 AST 语法树
    let ast: DeriveInput = syn::parse(input).unwrap();
    impl_hello_macro(&ast)
}

fn impl_hello_macro(ast: &syn:: DeriveInput) -> TokenStream {
    // 获取结构体、枚举标识
    let name = &ast.ident;
    let gen = quote! {
        // 为目标结构体或枚举自动实现特征
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    gen.into()
}
```


## <span class="section-num">25</span> error handling/Option/Result {#error-handling-option-result}

panic!()/unimplemented()!/todo!()

panic!() 时返回错误信息，unwinding stack 和释放资源（drop 对象）：

```rust
  fn drink(beverage: &str) {
      // You shouldn't drink too much sugary beverages.
      if beverage == "lemonade" { panic!("AAAaaaaa!!!!"); }

      println!("Some refreshing {} is all I need.", beverage);
  }

  fn main() {
      drink("water");
      drink("lemonade");
      drink("still water");
  }
```

panic!() 发生时默认是 unwind，也可以在 .cargo/config.toml 的 profile 中配置为 abort：

-   也可以通过 cargo build 或 rustc 的 -C panic=abort/unwind 参数来配置： rustc  lemonade.rs -C panic=abort

<!--listend-->

```toml
  [profile.dev]
  opt-level = 0
  debug = true
  split-debuginfo = '...'  # Platform-specific.
  strip = "none"
  debug-assertions = true
  overflow-checks = true
  lto = false
  panic = 'unwind'
  incremental = true
  codegen-units = 256
  rpath = false
```

代码可以使用 #[cfg(panic = "xx")] 来进行条件编译， 使用 cfg!() 来进行条件判断；

```rust
  // 根据 panic 设置进行条件编译
  #[cfg(panic = "unwind")]
  fn ah() {
      println!("Spit it out!!!!");
  }

  #[cfg(not(panic = "unwind"))]
  fn ah() {
      println!("This is not your party. Run!!!!");
  }

  fn drink(beverage: &str) {
      if beverage == "lemonade" {
          ah();
      } else {
          println!("Some refreshing {} is all I need.", beverage);
      }
  }

  fn main() {
      drink("water");
      drink("lemonade");
  }


  fn drink(beverage: &str) {
      // You shouldn't drink too much sugary beverages.
      if beverage == "lemonade" {
          if cfg!(panic = "abort") {
              println!("This is not your party. Run!!!!");
          } else {
              println!("Spit it out!!!!");
          }
      } else {
          println!("Some refreshing {} is all I need.", beverage);
      }
  }

  fn main() {
      drink("water");
      drink("lemonade");
  }
```

对于 Option/Result 可以使用 ? 来进行 unpacking，? 可以用于方法调用等表达式中间来使用。

```rust
  fn next_birthday(current_age: Option<u8>) -> Option<String> {
      // If `current_age` is `None`, this returns `None`.  If `current_age` is `Some`, the inner `u8`
      // value + 1 gets assigned to `next_age`
      let next_age: u8 = current_age? + 1; // unpacing Some 值或提前返回 None
      Some(format!("Next year I will be {}", next_age))
  }

  struct Person {
      job: Option<Job>,
  }

  #[derive(Clone, Copy)]
  struct Job {
      phone_number: Option<PhoneNumber>,
  }

  #[derive(Clone, Copy)]
  struct PhoneNumber {
      area_code: Option<u8>,
      number: u32,
  }

  impl Person {

      // Gets the area code of the phone number of the person's job, if it exists.
      fn work_phone_area_code(&self) -> Option<u8> {
          // This would need many nested `match` statements without the `?` operator.
          // It would take a lot more code - try writing it yourself and see which
          // is easier.
          self.job?.phone_number?.area_code
      }
  }

  fn main() {
      let p = Person {
          job: Some(Job {
              phone_number: Some(PhoneNumber {
                  area_code: Some(61),
                  number: 439222222,
              }),
          }),
      };

      assert_eq!(p.work_phone_area_code(), Some(61));
  }
```


### <span class="section-num">25.1</span> Option {#option}

```rust
  impl<T> Option<T>

  pub const fn is_some(&self) -> bool

                                      // 结果是 Some 切满足 predit
  pub fn is_some_and(self, f: impl FnOnce(T) -> bool) -> bool
  let x: Option<u32> = Some(2);
  assert_eq!(x.is_some_and(|x| x > 1), true);
  let x: Option<u32> = Some(0);
  assert_eq!(x.is_some_and(|x| x > 1), false);

  pub const fn is_none(&self) -> bool

                                    // 从 &Option<T> 转换为 Option<&T>
  pub const fn as_ref(&self) -> Option<&T>
  let text: Option<String> = Some("Hello, world!".to_string());
  // First, cast `Option<String>` to `Option<&String>` with `as_ref`,
  // then consume *that* with `map`, leaving `text` on the stack.
  let text_length: Option<usize> = text.as_ref().map(|s| s.len());
  println!("still can print text: {text:?}");

  pub fn as_mut(&mut self) -> Option<&mut T>
  let mut x = Some(2);
  match x.as_mut() {
      Some(v) => *v = 42,
      None => {},
  }
  assert_eq!(x, Some(42));

  pub fn as_pin_ref(self: Pin<&Option<T>>) -> Option<Pin<&T>>
  pub fn as_pin_mut(self: Pin<&mut Option<T>>) -> Option<Pin<&mut T>>

                                // 返回一个 slice，包含对应的 Some 元素，如果为 None 则返回空 slice
  pub fn as_slice(&self) -> &[T]
                                assert_eq!(
                                    [Some(1234).as_slice(), None.as_slice()],
                                    [&[1234][..], &[][..]],
                                );

  pub fn as_mut_slice(&mut self) -> &mut [T]

                              // 返回 Some 值，如果是 None 则 panic 并打印 msg
                              pub fn expect(self, msg: &str) -> T
                              let x = Some("value");
  assert_eq!(x.expect("fruits are healthy"), "value");

  // 返回 Some 值，如果是 None 则 panic
  pub fn unwrap(self) -> T

                              // 返回 Some 值，如果是 None，则使用缺省值、或函数返回值
  pub fn unwrap_or(self, default: T) -> T
  pub fn unwrap_or_else<F>(self, f: F) -> T where F: FnOnce() -> T
  pub fn unwrap_or_default(self) -> T where T: Default
  pub unsafe fn unwrap_unchecked(self) -> T

                              // 将 Option<T> 转换为 Option<U>, 如果为 None 则返回 None
  pub fn map<U, F>(self, f: F) -> Option<U> where F: FnOnce(T) -> U
  let maybe_some_string = Some(String::from("Hello, World!"));
  // `Option::map` takes self *by value*, consuming `maybe_some_string`
  let maybe_some_len = maybe_some_string.map(|s| s.len());
  assert_eq!(maybe_some_len, Some(13));
  let x: Option<&str> = None;
  assert_eq!(x.map(|s| s.len()), None);

  pub fn inspect<F>(self, f: F) -> Option<T> where F: FnOnce(&T)
  let v = vec![1, 2, 3, 4, 5];
  // prints "got: 4"
  let x: Option<&usize> = v.get(3).inspect(|x| println!("got: {x}"));
  // prints nothing
  let x: Option<&usize> = v.get(5).inspect(|x| println!("got: {x}"));

  // 如果为 None 则返回 default 值, 否则对 Some 值执行 f 函数
  pub fn map_or<U, F>(self, default: U, f: F) -> U where F: FnOnce(T) -> U
  let x = Some("foo");
  assert_eq!(x.map_or(42, |v| v.len()), 3);
  let x: Option<&str> = None;
  assert_eq!(x.map_or(42, |v| v.len()), 42);

  pub fn map_or_else<U, D, F>(self, default: D, f: F) -> U where D: FnOnce() -> U, F: FnOnce(T) -> U

                        // 将 Option 转换为 Result: 将 Some(v) -> Ok(v), None -> Err(err)
  pub fn ok_or<E>(self, err: E) -> Result<T, E>
  let x = Some("foo");
  assert_eq!(x.ok_or(0), Ok("foo"));
  let x: Option<&str> = None;
  assert_eq!(x.ok_or(0), Err(0));

  pub fn ok_or_else<E, F>(self, err: F) -> Result<T, E> where F: FnOnce() -> E

  pub fn as_deref(&self) -> Option<&<T as Deref>::Target> where T: Deref
  pub fn as_deref_mut(&mut self) -> Option<&mut <T as Deref>::Target> where T: DerefMut

  pub fn iter(&self) -> Iter<'_, T>
  let x = Some(4);
  assert_eq!(x.iter().next(), Some(&4));
  let x: Option<u32> = None;
  assert_eq!(x.iter().next(), None);

  pub fn iter_mut(&mut self) -> IterMut<'_, T>

                  // 如果 self 是 None 则返回  None,否则返回 optb
  pub fn and<U>(self, optb: Option<U>) -> Option<U>
  let x = Some(2);
  let y: Option<&str> = None;
  assert_eq!(x.and(y), None);
  let x: Option<u32> = None;
  let y = Some("foo");
  assert_eq!(x.and(y), None);
  let x = Some(2);
  let y = Some("foo");
  assert_eq!(x.and(y), Some("foo"));
  let x: Option<u32> = None;
  let y: Option<&str> = None;
  assert_eq!(x.and(y), None);


  // 如果 self 是 None 则返回 None,否则返回 f 函数的结果
  pub fn and_then<U, F>(self, f: F) -> Option<U> where F: FnOnce(T) -> Option<U>
  fn sq_then_to_string(x: u32) -> Option<String> {
      x.checked_mul(x).map(|sq| sq.to_string())
  }
  assert_eq!(Some(2).and_then(sq_then_to_string), Some(4.to_string()));
  assert_eq!(Some(1_000_000).and_then(sq_then_to_string), None); // overflowed!
  assert_eq!(None.and_then(sq_then_to_string), None);

  // 如果 self 是 None 则返回 None, 否则如果 predicate 返回 true 则返回 Some 值;
  pub fn filter<P>(self, predicate: P) -> Option<T> where P: FnOnce(&T) -> bool
  fn is_even(n: &i32) -> bool {
      n % 2 == 0
  }
  assert_eq!(None.filter(is_even), None);
  assert_eq!(Some(3).filter(is_even), None);
  assert_eq!(Some(4).filter(is_even), Some(4));

  pub fn or(self, optb: Option<T>) -> Option<T>
  let x = Some(2);
  let y = None;
  let x = None;
  let y = Some(100);
  assert_eq!(x.or(y), Some(100));
  let x = Some(2);
  let y = Some(100);
  assert_eq!(x.or(y), Some(2));
  let x: Option<u32> = None;
  let y = None;
  assert_eq!(x.or(y), None);

  pub fn or_else<F>(self, f: F) -> Option<T> where F: FnOnce() -> Option<T>
  pub fn xor(self, optb: Option<T>) -> Option<T>

          // 将 value 插入 Option 返回他的 &mut, Option 原来值被 dropped
  pub fn insert(&mut self, value: T) -> &mut T
  let mut opt = None;
  let val = opt.insert(1);
  assert_eq!(*val, 1);
  assert_eq!(opt.unwrap(), 1);
  let val = opt.insert(2);
  assert_eq!(*val, 2);
  *val = 3;
  assert_eq!(opt.unwrap(), 3);

  // 返回 Some 值的 &mut, 否则插入 value 值并返回他的 &mut
  pub fn get_or_insert(&mut self, value: T) -> &mut T
  let mut x = None;
  {
      let y: &mut u32 = x.get_or_insert(5);
      assert_eq!(y, &5);
      *y = 7;
  }
  assert_eq!(x, Some(7));

  pub fn get_or_insert_default(&mut self) -> &mut T where T: Default
  pub fn get_or_insert_with<F>(&mut self, f: F) -> &mut T where F: FnOnce() -> T

      // 从 self 中获取 Some 值, 将 self 设为 None
  pub fn take(&mut self) -> Option<T>
  let mut x = Some(2);
  let y = x.take();
  assert_eq!(x, None);
  assert_eq!(y, Some(2));
  let mut x: Option<u32> = None;
  let y = x.take();
  assert_eq!(x, None);
  assert_eq!(y, None);

  // 当 predicate 返回 ture 时 take Some 的值,将 self 设置为 None
  pub fn take_if<P>(&mut self, predicate: P) -> Option<T> where P: FnOnce(&mut T) -> bool

    // 用 value 替换 self 值, 返回 self 以前的值
  pub fn replace(&mut self, value: T) -> Option<T>
  let mut x = Some(2);
  let old = x.replace(5);
  assert_eq!(x, Some(5));
  assert_eq!(old, Some(2));
  let mut x = None;
  let old = x.replace(3);
  assert_eq!(x, Some(3));
  assert_eq!(old, None);

  // 如果 self 是 Some(s) 且 other 也是 Some(o),则返回 Some((s, o)), 否则返回 None
  pub fn zip<U>(self, other: Option<U>) -> Option<(T, U)>
  let x = Some(1);
  let y = Some("hi");
  let z = None::<u8>;
  assert_eq!(x.zip(y), Some((1, "hi")));
  assert_eq!(x.zip(z), None);

  pub fn zip_with<U, F, R>(self, other: Option<U>, f: F) -> Option<R> where F: FnOnce(T, U) -> R

// 从 Option<&T> 生成 Option<T>
impl<T> Option<&T>
pub fn copied(self) -> Option<T> where T: Copy
pub fn cloned(self) -> Option<T> where T: Clone
```


### <span class="section-num">25.2</span> Result {#result}

也支持 map/and_then 等方法:

```rust
  use std::num::ParseIntError;

  // As with `Option`, we can use combinators such as `map()`.
  // This function is otherwise identical to the one above and reads:
  // Multiply if both values can be parsed from str, otherwise pass on the error.
  fn multiply(first_number_str: &str, second_number_str: &str) -> Result<i32, ParseIntError> {
      first_number_str.parse::<i32>().and_then(|first_number| {
          second_number_str.parse::<i32>().map(|second_number| first_number * second_number)
      })
  }

  fn print(result: Result<i32, ParseIntError>) {
      match result {
          Ok(n)  => println!("n is {}", n),
          Err(e) => println!("Error: {}", e),
      }
  }

  fn main() {
      // This still presents a reasonable answer.
      let twenty = multiply("10", "2");
      print(twenty);

      // The following now provides a much more helpful error message.
      let tt = multiply("t", "2");
      print(tt);
  }
```

在 match 表达式中可以提前返回 Err(e):

```rust
  use std::num::ParseIntError;

  fn multiply(first_number_str: &str, second_number_str: &str) -> Result<i32, ParseIntError> {
      let first_number = match first_number_str.parse::<i32>() {
          Ok(first_number)  => first_number,
          Err(e) => return Err(e),
      };

      let second_number = match second_number_str.parse::<i32>() {
          Ok(second_number)  => second_number,
          Err(e) => return Err(e),
      };

      Ok(first_number * second_number)
  }

  fn print(result: Result<i32, ParseIntError>) {
      match result {
          Ok(n)  => println!("n is {}", n),
          Err(e) => println!("Error: {}", e),
      }
  }

  fn main() {
      print(multiply("10", "2"));
      print(multiply("t", "2"));
  }
```

Result 别名: 简化 Error 类型:

```rust
use std::fmt;

type Result<T> = std::result::Result<T, DoubleError>;

// Define our error types. These may be customized for our error handling cases.
// Now we will be able to write our own errors, defer to an underlying error
// implementation, or do something in between.
#[derive(Debug, Clone)]
struct DoubleError;

// Generation of an error is completely separate from how it is displayed.
// There's no need to be concerned about cluttering complex logic with the display style.
//
// Note that we don't store any extra info about the errors. This means we can't state
// which string failed to parse without modifying our types to carry that information.
impl fmt::Display for DoubleError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "invalid first item to double")
    }
}

fn double_first(vec: Vec<&str>) -> Result<i32> {
    vec.first()
        // Change the error to our new type.
        .ok_or(DoubleError)
        .and_then(|s| {
            s.parse::<i32>()
                // Update to the new error type here also.
                .map_err(|_| DoubleError)
                .map(|i| 2 * i)
        })
}

fn print(result: Result<i32>) {
    match result {
        Ok(n) => println!("The first doubled is {}", n),
        Err(e) => println!("Error: {}", e),
    }
}

fn main() {
    let numbers = vec!["42", "93", "18"];
    let empty = vec![];
    let strings = vec!["tofu", "93", "18"];

    print(double_first(numbers));
    print(double_first(empty));
    print(double_first(strings));
}
```

Box&lt;dyn std::error::Error&gt; 可以实现打包不同类型的 error:

-   Box 和 ? 都可以使用 From/Into trait 来实现自定义类型到返回值 error 类型转换.

<!--listend-->

```rust
  use std::error;
  use std::fmt;

  // Change the alias to use `Box<dyn error::Error>`.
  type Result<T> = std::result::Result<T, Box<dyn error::Error>>;

  #[derive(Debug, Clone)]
  struct EmptyVec;  // 自定义 error 类型
  impl fmt::Display for EmptyVec {
      fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
          write!(f, "invalid first item to double")
      }
  }
  impl error::Error for EmptyVec {} // std::error::Error 是 marker trait

  fn double_first(vec: Vec<&str>) -> Result<i32> {
      vec.first()
          .ok_or_else(|| EmptyVec.into()) // Converts to Box
          .and_then(|s| {
              s.parse::<i32>()
                  .map_err(|e| e.into()) // Converts to Box
                  .map(|i| 2 * i)
          })
  }

  // // The same structure as before but rather than chain all `Results`
  // // and `Options` along, we `?` to get the inner value out immediately.
  // fn double_first(vec: Vec<&str>) -> Result<i32> {
  //     let first = vec.first().ok_or(EmptyVec)?;
  //     let parsed = first.parse::<i32>()?;
  //     Ok(2 * parsed)
  // }


  fn print(result: Result<i32>) {
      match result {
          Ok(n) => println!("The first doubled is {}", n),
          Err(e) => println!("Error: {}", e),
      }
  }

  fn main() {
      let numbers = vec!["42", "93", "18"];
      let empty = vec![];
      let strings = vec!["tofu", "93", "18"];

      print(double_first(numbers));
      print(double_first(empty));
      print(double_first(strings));
  }
```

自定义 Error 类型, 提供更丰富/个性化的上下文和出错信息:

-   实现 fmt::Display, std::error::Error;
-   实现  From&lt;XX&gt;  trait, 将其他类型错误 XX 转换为自定义类型错误;

<!--listend-->

```rust
use std::error;
use std::error::Error;
use std::num::ParseIntError;
use std::fmt;

type Result<T> = std::result::Result<T, DoubleError>;

#[derive(Debug)]
enum DoubleError {
    EmptyVec,
    // We will defer to the parse error implementation for their error.
    // Supplying extra info requires adding more data to the type.
    Parse(ParseIntError),
}

impl fmt::Display for DoubleError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            DoubleError::EmptyVec =>
                write!(f, "please use a vector with at least one element"),
            // The wrapped error contains additional information and is available
            // via the source() method.
            DoubleError::Parse(..) =>
                write!(f, "the provided string could not be parsed as int"),
        }
    }
}

impl error::Error for DoubleError {
    fn source(&self) -> Option<&(dyn error::Error + 'static)> {
        match *self {
            DoubleError::EmptyVec => None,
            // The cause is the underlying implementation error type. Is implicitly
            // cast to the trait object `&error::Error`. This works because the
            // underlying type already implements the `Error` trait.
            DoubleError::Parse(ref e) => Some(e),
        }
    }
}

// Implement the conversion from `ParseIntError` to `DoubleError`.
// This will be automatically called by `?` if a `ParseIntError`
// needs to be converted into a `DoubleError`.
impl From<ParseIntError> for DoubleError {
    fn from(err: ParseIntError) -> DoubleError {
        DoubleError::Parse(err)
    }
}

fn double_first(vec: Vec<&str>) -> Result<i32> {
    let first = vec.first().ok_or(DoubleError::EmptyVec)?;
    // Here we implicitly use the `ParseIntError` implementation of `From` (which
    // we defined above) in order to create a `DoubleError`.
    let parsed = first.parse::<i32>()?;

    Ok(2 * parsed)
}

fn print(result: Result<i32>) {
    match result {
        Ok(n)  => println!("The first doubled is {}", n),
        Err(e) => {
            println!("Error: {}", e);
            if let Some(source) = e.source() {
                println!("  Caused by: {}", source);
            }
        },
    }
}

fn main() {
    let numbers = vec!["42", "93", "18"];
    let empty = vec![];
    let strings = vec!["tofu", "93", "18"];

    print(double_first(numbers));
    print(double_first(empty));
    print(double_first(strings));
}
```


## <span class="section-num">26</span> FFI {#ffi}

Rust 提供 Foreign Function Interface (FFI) 来调用 C 库.

Foreign function 必须在 extern {} block 中声明, 而且使用 #[link] attr 来指定要链接的外部 C 库名称:

```rust
  use std::fmt;

  // this extern block links to the libm library
  #[cfg(target_family = "windows")]
  #[link(name = "msvcrt")]
  extern {
      // this is a foreign function that computes the square root of a single precision complex number
      fn csqrtf(z: Complex) -> Complex;

      fn ccosf(z: Complex) -> Complex;
  }

  #[cfg(target_family = "unix")]
  #[link(name = "m")]
  extern {
      // this is a foreign function that computes the square root of a single precision complex number
      fn csqrtf(z: Complex) -> Complex;

      fn ccosf(z: Complex) -> Complex;
  }

  // Since calling foreign functions is considered unsafe, it's common to write safe wrappers around
  // them.
  fn cos(z: Complex) -> Complex {
      unsafe { ccosf(z) }
  }

  fn main() {
      // z = -1 + 0i
      let z = Complex { re: -1., im: 0. };

      // calling a foreign function is an unsafe operation
      let z_sqrt = unsafe { csqrtf(z) };

      println!("the square root of {:?} is {:?}", z, z_sqrt);

      // calling safe API wrapped around unsafe operation
      println!("cos({:?}) = {:?}", z, cos(z));
  }

  // Minimal implementation of single precision complex numbers
  #[repr(C)]
  #[derive(Clone, Copy)]
  struct Complex {
      re: f32,
      im: f32,
  }

  impl fmt::Debug for Complex {
      fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
          if self.im < 0. {
              write!(f, "{}-{}i", self.re, -self.im)
          } else {
              write!(f, "{}+{}i", self.re, self.im)
          }
      }
  }
```


## <span class="section-num">27</span> testing {#testing}

Rust 提供了如下测试类型:

1.  Unit testing;
2.  Doc testing;
3.  Integration testing;

Rust 的 Cargo.toml 中也为 testing 提供了单独的依赖配置 dev-dependencies:

```toml
# standard crate data is left out
[dev-dependencies]
pretty_assertions = "1"
```

src/lib.rs:

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq; // crate for test-only use. Cannot be used in non-test code.

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }
}
```

Unit testing:

1.  使用 #[cfg(test)] 来注解 test module;
2.  使用 #[test] 来注解 test 函数;
3.  test 函数内部使用 assert!()/assert_eq!()/assert_ne!() 等来报告错误;
4.  单元测试 module/func 和源码在同一个文件, 所以可以测试 private code(但是集成测试在不同 crate,只能测试 public 接口);

<!--listend-->

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// This is a really bad adding function, its purpose is to fail in this
// example.
#[allow(dead_code)]
fn bad_add(a: i32, b: i32) -> i32 {
    a - b
}

#[cfg(test)]
mod tests {
    // Note this useful idiom: importing names from outer (for mod tests) scope.
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(1, 2), 3);
    }

    #[test]
    fn test_bad_add() {
        // This assert would fire and test will fail.
        // Please note, that private functions can be tested too!
        assert_eq!(bad_add(1, 2), 3);
    }
}
```

使用 cargo test 来运行测试:

```shell
$ cargo test

running 2 tests
test tests::test_bad_add ... FAILED
test tests::test_add ... ok

failures:

---- tests::test_bad_add stdout ----
        thread 'tests::test_bad_add' panicked at 'assertion failed: `(left == right)`
  left: `-1`,
 right: `3`', src/lib.rs:21:8
note: Run with `RUST_BACKTRACE=1` for a backtrace.


failures:
    tests::test_bad_add

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out

```

Rust 2018 版本的单元测试函数支持返回 Result 来报错, 这样可以使用 ?

```rust
fn sqrt(number: f64) -> Result<f64, String> {
    if number >= 0.0 {
        Ok(number.powf(0.5))
    } else {
        Err("negative floats don't have square roots".to_owned())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_sqrt() -> Result<(), String> {
        let x = 4.0;
        assert_eq!(sqrt(x)?.powf(2.0), x);
        Ok(())
    }
}
```

单元测试也支持 panic!() 测试:

```rust
pub fn divide_non_zero_result(a: u32, b: u32) -> u32 {
    if b == 0 {
        panic!("Divide-by-zero error");
    } else if a < b {
        panic!("Divide result is zero");
    }
    a / b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_divide() {
        assert_eq!(divide_non_zero_result(10, 2), 5);
    }

    #[test]
    #[should_panic]
    fn test_any_panic() {
        divide_non_zero_result(1, 0);
    }

    #[test]
    #[should_panic(expected = "Divide result is zero")]
    fn test_specific_panic() {
        divide_non_zero_result(1, 10);
    }
}
```

也可以为 test 函数添加 #[ignore] 来忽略测试:

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 2), 4);
    }

    #[test]
    fn test_add_hundred() {
        assert_eq!(add(100, 2), 102);
        assert_eq!(add(2, 100), 102);
    }

    #[test]
    #[ignore]
    fn ignored_test() {
        assert_eq!(add(0, 0), 0);
    }
}
```

Rsut 源码中的 document comment 使用 Markdown 语法, 支持嵌入代码块, 这些代码别编译和文档测试:

```rust
/// First line is a short summary describing function.
///
/// The next lines present detailed documentation. Code blocks start with
/// triple backquotes and have implicit `fn main()` inside
/// and `extern crate <cratename>`. Assume we're testing `doccomments` crate:
///
/// ```
/// let result = doccomments::add(2, 3);
/// assert_eq!(result, 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

/// Usually doc comments may include sections "Examples", "Panics" and "Failures".
///
/// The next function divides two numbers.
///
/// # Examples
///
/// ```
/// let result = doccomments::div(10, 2);
/// assert_eq!(result, 5);
/// ```
///
/// # Panics
///
/// The function panics if the second argument is zero.
///
/// ```rust,should_panic
/// // panics on division by zero
/// doccomments::div(10, 0);
/// ```
pub fn div(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("Divide-by-zero error");
    }

    a / b
}
```

测试:

```shell
$ cargo test
running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests doccomments

running 3 tests
test src/lib.rs - add (line 7) ... ok
test src/lib.rs - div (line 21) ... ok
test src/lib.rs - div (line 31) ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

```

单元测试 module/func 和源码在同一个文件, 所以可以测试 private code. 但是集成测试在不同 crate,只能测试 public 接口, 集成测试的代码位于 tests 目录下:

```rust
  // src/lib.rs
  // Define this in a crate called `adder`.
  pub fn add(a: i32, b: i32) -> i32 {
      a + b
  }

  // tests/integration_test.rs
  #[test]
  fn test_add() {
      assert_eq!(adder::add(3, 2), 5);
  }
```

测试:

```shell
$ cargo test
running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

     Running target/debug/deps/integration_test-bcd60824f5fbfe19

running 1 test
test test_add ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

集成测试也可以包含 module:

```rust
// tests/common/mod.rs:
pub fn setup() {
    // some setup code, like creating required files/directories, starting
    // servers, etc.
}

// tests/integration_test.rs
// importing common module.
mod common;

#[test]
fn test_add() {
    // using common code.
    common::setup();
    assert_eq!(adder::add(3, 2), 5);
}
```

参考:

1.  [Everything you need to know about testing in Rust](https://www.shuttle.rs/blog/2024/03/21/testing-in-rust)


## <span class="section-num">28</span> unsafe {#unsafe}

unsafe {} 块注解用于指示 rustc 忽略一些严格的安全检查, 主要使用场景:

1.  解构 raw pointers;
2.  调用 FFI 函数;
3.  调用标记为 unsafe 的函数;
4.  存取 static mut 全局变量;
5.  实现 unsafe trait;

raw pointer:

```rust
fn main() {
    let raw_p: *const u32 = &10;
    unsafe {
        assert!(*raw_p == 10);
    }
}
```

调用 unsafe 函数:

```rust
use std::slice;

fn main() {
    let some_vector = vec![1, 2, 3, 4];

    let pointer = some_vector.as_ptr();
    let length = some_vector.len();

    unsafe {
        let my_slice: &[u32] = slice::from_raw_parts(pointer, length);

        assert_eq!(some_vector.as_slice(), my_slice);
    }
}
```


## <span class="section-num">29</span> 工程化 {#工程化}

Cargo.toml： 文件中指定的外部crate版本，最好是确定版本号，即 0.8.3这样的，以避免意外的版本不同而导致结果不同（这也意味着需要手动维护升级依赖库版本）

rust-toolchain：指定编译的toolchain版本，以避免使用不同的toolchain版本导致结果不同

rustfmt.toml：工程上，需要大家都统一代码格式 （我刚遇到混乱的代码格式，如果想自动修复，就得用cargo
fix）

代码提交前都需要的基本检查：比如没有warning，cargo clippy没有报错

跨crate的依赖，禁止使用路径方式：遇到类似，crate A编译依赖crate B， C， D，但目录放置需要按照：
crateA + crateB + crateB/crateC + crataB/crateC/crateD的样式来放置才可以编译通过。

尽可能松散的包的依赖关系：遇到类似，一个crate编译需要依赖其他N个crate，N个包之间的关系错综复杂的依赖。

如果这些包确实是紧耦合，为什么不放在一个workspace里面？

任何大crate都需要在顶层放置Readme，讲述怎么编译，其中的模块大致介绍和彼此之间的关系


## <span class="section-num">30</span> 参考 {#参考}

1.  [Rust By Example](https://doc.rust-lang.org/rust-by-example/index.html)
