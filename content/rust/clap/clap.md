---
title: "clap"
author: ["zhangjun"]
lastmod: 2024-06-08T21:57:43+08:00
tags: ["rust", "clap"]
draft: false
series: ["rust crate"]
series_order: 12
---

可以为 command 和 args 自定义 template：
<https://docs.rs/clap/latest/clap/_derive/_cookbook/repl/index.html>

两种使用方式:

1.  derive macro: <https://docs.rs/clap/latest/clap/_derive/index.html>
    -   需要启用 clap crate 的 derive/cargo feature，features 列表:
        <https://docs.rs/clap/latest/clap/_features/index.html>
2.  builder: <https://docs.rs/clap/latest/clap/builder/index.html>

<!--listend-->

```shell
cd codes/rust/
mkdir clap
cd clap/
cargo new clap-demo
cd clap-demo/
cargo add clap --features derive,cargo
cargo build

zj@a:~/codes/rust/clap/clap-demo$ ./target/debug/clap-demo -h
Usage: clap-demo [OPTIONS] [name] [COMMAND]

Commands:
test  does testing things
help  Print this message or the help of the given subcommand(s)

Arguments:
[name]  Optional name to operate on

Options:
-c, --config <FILE>  Sets a custom config file
-d, --debug...       Turn debugging information on
-h, --help           Print help
-V, --version        Print version
zj@a:~/codes/rust/clap/clap-demo$
```

在 builder 模式中使用 macro 来快速定义 Comand/Arg：

command
: Allows you to build the Command instance from your Cargo.toml at compile time.

    -   command!() 宏可以从 Cargo.toml 中提取 Command 的 name、about、author、version 等信息，不需要单独设

    定；<https://docs.rs/clap/latest/clap/macro.command.html>

arg
: Create an Arg from a usage string， arg!() 宏可以更方便的创建 [struct Arg 对象]()；

crate_authors
: Allows you to pull the authors for the command from your Cargo.toml at compile
    time in the form:"author1 lastname &lt;author1@example.com&gt;:author2 lastname &lt;author2@example.com&gt;"

crate_description
: Allows you to pull the description from your Cargo.toml at compile time.

crate_name
: Allows you to pull the name from your Cargo.toml at compile time.

crate_version
: Allows you to pull the version from your Cargo.toml at compile time as
    MAJOR.MINOR.PATCH_PKGVERSION_PRE
    -   crate_XX!() 宏从 Cargo.toml 中提取 Command 的 name、about、author、version、description 等信息；

value_parser
: Select a `ValueParser` implementation from the intended type，默认是
    ValueParser::string；
    -   value_parser!() 的参数是类型名称, 如 std::path::PathBuf 或其他自定义类型等;

<!--listend-->

```rust
use std::path::PathBuf;

use clap::{arg, command, value_parser, ArgAction, Command};

fn main() {

    let m = Command::new("cmd")
        .author(crate_authors!("\n"))
        .version(crate_version!())
        .about(crate_description!())
        .get_matches();

    let m = Command::new(crate_name!()).get_matches();

    // command!() : requires `cargo` feature，从 Cargo.toml 中读取 name/version/authors/description
    // 信息
    let matches = command!()
        .arg(arg!( [name] "Optional name to operate on") )
        .arg(arg!( -c --config <FILE> "Sets a custom config file" )
            // We don't have syntax yet for optional options, so manually calling `required`
            .required(false)
            .value_parser(value_parser!(PathBuf)),
        )
        .arg(arg!( -d --debug ... "Turn debugging information on" ))
        .subcommand(
            Command::new("test")
                .about("does testing things")
                .arg(arg!(-l --list "lists test values").action(ArgAction::SetTrue)),
        )
        .get_matches();

    // You can check the value provided by positional arguments, or option arguments
    if let Some(name) = matches.get_one::<String>("name") {
        println!("Value for name: {name}");
    }

    if let Some(config_path) = matches.get_one::<PathBuf>("config") {
        println!("Value for config: {}", config_path.display());
    }

    // You can see how many times a particular flag or argument occurred
    // Note, only flags can have multiple occurrences
    match matches.get_one::<u8>("debug").expect("Count's are defaulted")
    {
        0 => println!("Debug mode is off"),
        1 => println!("Debug mode is kind of on"),
        2 => println!("Debug mode is on"),
        _ => println!("Don't be crazy"),
    }

    // You can check for the existence of subcommands, and if found use their matches just as you
    // would the top level cmd
    if let Some(matches) = matches.subcommand_matches("test") {
        // "$ myapp test" was run
        if matches.get_flag("list") {
            // "$ myapp test -l" was run
            println!("Printing testing lists...");
        } else {
            println!("Not printing testing lists...");
        }
    }

    // Continued program logic goes here...
}
```

Traits: 通过 derive macro 来实现，使用 #[derive] 定义命令和参数。

-   Parser    Parse command-line arguments into Self.
-   Args     Parse a set of arguments into a user-defined container.
-   CommandFactory    Create a Command relevant for a user-defined container.
-   FromArgMatches    Converts an instance of ArgMatches to a user-defined container.
-   Subcommand    Parse a sub-command into a user-defined enum.
-   ValueEnum    Parse arguments into enums.


## <span class="section-num">1</span> ValueParser {#valueparser}

ValueParser 是 Arg::value_parser() 的参数类型，用于解析和校验参数:

```rust
pub struct ValueParser(/* private fields */);
```

有两种创建方式：

1.  `value_parser!` for automatically selecting an implementation for a given type
    -   value_parser!() 的参数是 `类型名称` , 如 std::path::PathBuf 或其他自定义类型等;
    -   clap::builder::ValueParserFactory 提供了 value_parser!() 可以使用的类型，包括 Rust 常见内置原始类型；
2.  `ValueParser::new()` for additional TypedValueParser that can be used
    -   new() 函数的签名: pub fn new&lt;P&gt;(other: P) -&gt; ValueParser where  P: TypedValueParser,
    -   所以, 任何实现 TypedValueParser trait 的类型都可以用来创建 ValueParser;
    -   clap::builder 提供了一些实现 TypedValueParser trait 的类型, 如 BoolValueParser 等;

value_parser!() 是通过 clap::builder::ValueParserFactory 来注册支持的类型：

1.  `ValueParserFactory` types, including
    -   Native types: bool, String, OsString, PathBuf
    -   Ranged numeric types: u8, i8, u16, i16, u32, i32, u64, i64
2.  ValueEnum types
3.  From&lt;OsString&gt; types and From&lt;&amp;OsStr&gt; types
4.  From&lt;String&gt; types and From&lt;&amp;str&gt; types
5.  FromStr types, including `usize, isize`

<!--listend-->

```rust
// Register a type with value_parser!
pub trait ValueParserFactory {
    type Parser;

    // Required method
    fn value_parser() -> Self::Parser;
}

// clap 内置注册的类型
impl ValueParserFactory for bool
impl ValueParserFactory for i8
impl ValueParserFactory for i16
impl ValueParserFactory for i32
impl ValueParserFactory for i64
impl ValueParserFactory for u8
impl ValueParserFactory for u16
impl ValueParserFactory for u32
impl ValueParserFactory for u64
impl ValueParserFactory for Box<str>
impl ValueParserFactory for Box<OsStr>
impl ValueParserFactory for Box<Path>
impl ValueParserFactory for String
impl ValueParserFactory for OsString
impl ValueParserFactory for PathBuf

impl<T> ValueParserFactory for Box<T>
where
    T: ValueParserFactory + Send + Sync + Clone,
    <T as ValueParserFactory>::Parser: TypedValueParser<Value = T>,

impl<T> ValueParserFactory for Arc<T>
where
    T: ValueParserFactory + Send + Sync + Clone,
    <T as ValueParserFactory>::Parser: TypedValueParser<Value = T>,

impl<T> ValueParserFactory for Wrapping<T>
where
    T: ValueParserFactory + Send + Sync + Clone,
    <T as ValueParserFactory>::Parser: TypedValueParser<Value = T>,
```

通过实现 clap::builder::ValueParserFactory 和 clap::builder::TypedValueParser 来为注册自定义类型的解析：

```rust
#[derive(Copy, Clone, Debug)]
pub struct Custom(u32);

impl clap::builder::ValueParserFactory for Custom {
    type Parser = CustomValueParser;
    fn value_parser() -> Self::Parser {
        CustomValueParser
    }
}
#[derive(Clone, Debug)]
pub struct CustomValueParser;

impl clap::builder::TypedValueParser for CustomValueParser {
    type Value = Custom;

    fn parse_ref(
        &self,
        cmd: &clap::Command,
        arg: Option<&clap::Arg>,
        value: &std::ffi::OsStr,
    ) -> Result<Self::Value, clap::Error> {
        let inner = clap::value_parser!(u32);
        let val = inner.parse_ref(cmd, arg, value)?;
        Ok(Custom(val))
    }
}
let parser: CustomValueParser = clap::value_parser!(Custom);
```

clap::builder 提供其他实现了 TypedValuePrarser trait 的类型，他们也可以作为 ValueParser::new() 的参数, 来创建 ValueParser 对象:

-   BoolValueParser    Implementation for ValueParser::bool
-   BoolishValueParser    Parse bool-like string values, everything else is true
-   EnumValueParser    Parse an ValueEnum value.
-   FalseyValueParser    Parse false-like string values, everything else is true
-   MapValueParser    Adapt a TypedValueParser from one value to another
-   NonEmptyStringValueParser    Parse non-empty string values
-   OsStringValueParser    Implementation for ValueParser::os_string
-   PathBufValueParser    Implementation for ValueParser::path_buf
-   PossibleValue    A possible value of an argument.
-   PossibleValuesParser    Verify the value is from an enumerated set of PossibleValue.
-   RangedI64ValueParser    Parse number that fall within a range of values
-   RangedU64ValueParser    Parse number that fall within a range of values
-   StringValueParser    Implementation for ValueParser::string
-   TryMapValueParser    Adapt a TypedValueParser from one value to another
-   UnknownArgumentValueParser    When encountered, report ErrorKind::UnknownArgument

RangedI64ValueParser 和 RangedU64ValueParser 用于定义一个 range 范围：

```rust
let mut cmd = clap::Command::new("raw")
    .arg(
        clap::Arg::new("port")
            .long("port")
            .value_parser(clap::value_parser!(u16).range(3000..))
            .action(clap::ArgAction::Set)
            .required(true)
    );

let m = cmd.try_get_matches_from_mut(["cmd", "--port", "3001"]).unwrap();
let port: u16 = *m.get_one("port")
    .expect("required");
assert_eq!(port, 3001);


let mut cmd = clap::Command::new("raw")
    .arg(
        clap::Arg::new("append")
            .value_parser(clap::builder::NonEmptyStringValueParser::new())
            .required(true)
    );

let m = cmd.try_get_matches_from_mut(["cmd", "true"]).unwrap();
let port: &String = m.get_one("append")
    .expect("required");
assert_eq!(port, "true");

pub enum ColorChoice {
    Auto,
    Always,
    Never,
}
```

value_parser!() 为 Rust 基本类型和复杂类型创建 ValueParser，而 clap::builder 则为一些特殊类型预定义了实现 TypeValueParser trait 的类型 XXValueParser（如 NonEmptyStringValueParser) , 所以两者一般是结合使用的。

Arg::value_parser() 的方法签名：pub fn value_parser(self, parser: impl IntoResettable&lt;ValueParser&gt;)
-&gt; Arg, 也就是任何实现 IntoResettable&lt;ValueParser&gt; 的类型，都可以作为 parser 参数。

-   value_parser!(T) for auto-selecting a value parser for a given type，
    -   Or range expressions like 0..=1 as a shorthand for RangedI64ValueParser
-   Fn(&amp;str) -&gt; Result&lt;T, E&gt;
    -   该类型闭包实现了 TypedValueParser trait.
-   `[&str]` and PossibleValuesParser for static enumerated values
-   BoolishValueParser, and FalseyValueParser for alternative bool implementations
-   NonEmptyStringValueParser for basic validation for strings
-   or any other `TypedValueParser` implementation

总结: value_parser!(T) 返回的 ValueParser, 闭包 Fn(&amp;str) -&gt; Result&lt;T, E&gt;, [&amp;str], 各种实现
TypedValueParser 的类型(如 clap::builder 提供的 BoolValueParser) 都可以作为 Arg::value_parser() 的参数.

```rust
pub fn value_parser(self, parser: impl IntoResettable<ValueParser>) -> Arg

// clap 为所有 Into<ValueParser> 实现了 IntoResettable<ValueParser>
impl<I> IntoResettable<ValueParser> for I where I: Into<ValueParser>,
impl IntoResettable<ValueParser> for Option<ValueParser>
impl<I> IntoResettable<ValueRange> for I where I: Into<ValueRange>,

// 例子: 使用 Fn 闭包(因为他实现了 TypeValueParser) 来作为 value_parser.
// https://github.com/tokio-rs/mini-redis/blob/master/src/bin/cli.rs#L32
fn bytes_from_str(src: &str) -> Result<Bytes, Infallible> {
    Ok(Bytes::from(src.to_string()))
}

#[derive(Subcommand, Debug)]
enum Command {
    Ping {
        /// Message to ping
        #[clap(value_parser = bytes_from_str)] // 新 clap 版本使用  #[arg(value_parser = bytes_from_str)]
        msg: Option<Bytes>,
    },
    /// Get the value of key.
    Get {
        /// Name of key to get
        key: String,
    },
    /// Set key to hold the string value.
    Set {
        /// Name of key to set
        key: String,

        /// Value to set.
        #[clap(value_parser = bytes_from_str)]
        value: Bytes,

        /// Expire the value after specified amount of time
        #[clap(value_parser = duration_from_ms_str)]
        expires: Option<Duration>,
    },
    ///  Publisher to send a message to a specific channel.
    Publish {
        /// Name of channel
        channel: String,

        #[clap(value_parser = bytes_from_str)]
        /// Message to publish
        message: Bytes,
    },
    /// Subscribe a client to a specific channel or channels.
    Subscribe {
        /// Specific channel or channels
        channels: Vec<String>,
    },
}
```

clap 为 Into&lt;ValueParser&gt;/Option&lt;ValueParser&gt; 实现了 IntoResettable&lt;ValueParser&gt;，所以任何实现了
From&lt;T&gt; 来转成 ValueParser 的类型都可以作为 Arg::value_parser() 的参数，如 [P; C], Vec&lt;P&gt; 和
TypedValueParser。

```rust
// 为数组 [P; C] 和 Vec<P> 转换为 ValueParser
impl<P, const C: usize> From<[P; C]> for ValueParser where P: Into<PossibleValue>,
impl<P> From<Vec<P>> for ValueParser where P: Into<PossibleValue>,

// 为任何实现 TypedValueParser 的自定义类型转换为 ValueParser
impl<P> From<P> for ValueParser where P: TypedValueParser + Send + Sync + 'static,

// 为 RangeXX 转换为 ValueParser
impl From<Range<i64>> for ValueParser
impl From<RangeFrom<i64>> for ValueParser
impl From<RangeFull> for ValueParser
impl From<RangeInclusive<i64>> for ValueParser
impl From<RangeTo<i64>> for ValueParser
impl From<RangeToInclusive<i64>> for ValueParser

// 从预定义的关联函数创建 ValueParser
pub const fn bool() -> ValueParser
pub const fn string() -> ValueParser
pub const fn os_string() -> ValueParser
pub const fn path_buf() -> ValueParser

// 更灵活的是为实现 TypedValueParser 的自定义类型创建 ValueParser
pub fn new<P>(other: P) -> ValueParser where P: TypedValueParser,
```

PossibleValue 和 PossibleValuesParser：

1.  任何可以 Into&lt;Str&gt; 的类型都可以转成 PossibleValue；
2.  任何可以迭代生成 PossibleValue 的类型都可以转成 PossibleValuesParser；

<!--listend-->

```rust
pub struct PossibleValue { /* private fields */ }
impl PossibleValue
pub fn new(name: impl Into<Str>) -> PossibleValue
impl<S> From<S> for PossibleValue where S: Into<Str>, // 任何能 Into<Str> 的类型都可以转成 PossibleValue

impl PossibleValuesParser
pub fn new(values: impl Into<PossibleValuesParser>) -> PossibleValuesParser
impl<I, T> From<I> for PossibleValuesParser where
    I: IntoIterator<Item = T>,
    T: Into<PossibleValue>,
impl TypedValueParser for PossibleValuesParser

// 示例
let mut cmd = clap::Command::new("raw")
    .arg(
        clap::Arg::new("color")
            // 必须都是字符串列表
            .value_parser(clap::builder::PossibleValuesParser::new(["always", "auto", "never"]))
            .required(true)
        );

let m = cmd.try_get_matches_from_mut(["cmd", "always"]).unwrap();
let port: &String = m.get_one("color")
    .expect("required");
assert_eq!(port, "always");
```

TypedValueParser 是一个 trait，可以为自定义类型实现该 trait 来实现自定义解析，他们可以
Into&lt;ValueParser&gt;，所以都可以作为 Arg::value_parser() 的参数：

-   TypedValueParser 的 parse_ref() 从传入的 arg/value 来解析出 type Value 对应的类型值。
-   闭包 Fn(&amp;str) -&gt; Result&lt;T,E&gt; 也实现了 TypedValueParser，可以灵活的对参数进行解析；

<!--listend-->

```rust
pub trait TypedValueParser: Clone + Send + Sync + 'static {
    type Value: Send + Sync + Clone;

    // Required method
    fn parse_ref(
        &self,
        cmd: &Command,
        arg: Option<&Arg>,
        value: &OsStr
    ) -> Result<Self::Value, Error>;
    //...
}

impl TypedValueParser for BoolValueParser // 返回 bool
impl TypedValueParser for BoolishValueParser
impl TypedValueParser for FalseyValueParser
impl TypedValueParser for NonEmptyStringValueParser // 返回非空 String
impl TypedValueParser for OsStringValueParser // 返回 OsString
impl TypedValueParser for PathBufValueParser // 返回 PathBuf
impl TypedValueParser for PossibleValuesParser
impl TypedValueParser for StringValueParser // 返回 String
impl TypedValueParser for UnknownArgumentValueParser
impl<E> TypedValueParser for EnumValueParser<E> where E: ValueEnum + Clone + Send + Sync + 'static,

//  Fn(&str) -> Result<T,E> 闭包也实现了 TypedValueParser, 也可以作为 Arg.value_parser() 的输入参数.
impl<F, T, E> TypedValueParser for F
where
    F: Fn(&str) -> Result<T, E> + Clone + Send + Sync + 'static,
    E: Into<Box<dyn Error + Sync + Send>>,
    T: Send + Sync + Clone,

impl<P, F, T> TypedValueParser for MapValueParser<P, F>
where
    P: TypedValueParser,
    <P as TypedValueParser>::Value: Send + Sync + Clone,
    F: Fn(<P as TypedValueParser>::Value) -> T + Clone + Send + Sync + 'static,
    T: Send + Sync + Clone,

impl<P, F, T, E> TypedValueParser for TryMapValueParser<P, F>
where
    P: TypedValueParser,
    <P as TypedValueParser>::Value: Send + Sync + Clone,
    F: Fn(<P as TypedValueParser>::Value) -> Result<T, E> + Clone + Send + Sync + 'static,
    T: Send + Sync + Clone,
    E: Into<Box<dyn Error + Sync + Send>>,

impl<T> TypedValueParser for RangedI64ValueParser<T>
where
    T: TryFrom<i64> + Clone + Send + Sync + 'static,
    <T as TryFrom<i64>>::Error: Send + Sync + 'static + Error + ToString,

impl<T> TypedValueParser for RangedU64ValueParser<T>
where
    T: TryFrom<u64> + Clone + Send + Sync + 'static,
    <T as TryFrom<u64>>::Error: Send + Sync + 'static + Error + ToString,
```

示例：

-   为调用 value_parser() 指定 ValueParser 时默认为 StringValueParser，所以默认解析为 String；
-   为调用 action() 时默认为 ArgAction::Set，对于 Vec 需要指定为 action(ArgAction::Append))；

<!--listend-->

```rust
let matches = command!() // requires `cargo` feature
    .arg(
        arg!([PORT])
            .value_parser(value_parser!(u16)) // 原始类型
            .default_value("2020"),
    )
    .get_matches();


let cfg = Arg::new("config")
    .action(ArgAction::Set)
    .value_name("FILE")
    // [PossibleValue; 3] 实现了 ValueParser
    .value_parser([
        PossibleValue::new("fast"),
        PossibleValue::new("slow").help("slower than fast"),
        PossibleValue::new("secret speed").hide(true)
    ]);

let cfg = Arg::new("config")
    .action(ArgAction::Set)
    .value_name("FILE")
    // RangeFull 实现了 ValueParser
    .value_parser(2..5);

use clap::{arg, command, ArgAction};

fn main() {
    let matches = command!() // requires `cargo` feature
        .next_line_help(true)
        // 未指定 ValueParser 时默认为 StringValueParser
        .arg(arg!(--two <VALUE>).required(true).action(ArgAction::Set))
        .arg(arg!(--one <VALUE>).required(true).action(ArgAction::Set))
        .get_matches();
    println!(
        "two: {:?}",
        matches.get_one::<String>("two").expect("required") // 默认为 String
    );
    println!(
        "one: {:?}",
        matches.get_one::<String>("one").expect("required")
    );
}

let mut cmd = clap::Command::new("raw")
    .arg(
        clap::Arg::new("color")
            .long("color")
            // [&str; 3] 实现了 ValueParser
            .value_parser(["always", "auto", "never"])
            .default_value("auto")
    );

let mut cmd = clap::Command::new("raw")
    .arg(
        clap::Arg::new("output")
            .value_parser(clap::value_parser!(PathBuf))
            .required(true)
    );

let m = cmd.try_get_matches_from_mut(["cmd", "file.txt"]).unwrap();
let port: &PathBuf = m.get_one("output")
    .expect("required");
assert_eq!(port, Path::new("file.txt"));

// Built-in types
let parser = clap::value_parser!(String);
assert_eq!(format!("{parser:?}"), "ValueParser::string");
let parser = clap::value_parser!(std::ffi::OsString);
assert_eq!(format!("{parser:?}"), "ValueParser::os_string");
let parser = clap::value_parser!(std::path::PathBuf);
assert_eq!(format!("{parser:?}"), "ValueParser::path_buf");
clap::value_parser!(u16).range(3000..);
clap::value_parser!(u64).range(3000..);

// FromStr types
let parser = clap::value_parser!(usize);
assert_eq!(format!("{parser:?}"), "_AnonymousValueParser(ValueParser::other(usize))");

// ValueEnum types
clap::value_parser!(ColorChoice);
```

Vec 类型 Arg 的实现:

-   builder 风格: 使用 .action(clap::ArgAction::Append) 来指定 flag Arg 出现多次时是 Append 形成一个 Vec;
    -   但 .num_args(2)  是为 flag Arg 指定有多个参数值;
-   derive 风格: Vec&lt;T&gt;:
    -   clap 隐式调用 .action(ArgAction::Append).required(false);
    -   默认为 T 自动添加  `.value_parser(value_parser!(T))`, 如果不符合预期, 如 T 不是 Rust 原始类型,
        则需要为自定义类型实现 ValueParserFactory;

<!--listend-->

```rust
// builder 风格
let cmd = Command::new("mycmd")
    .arg(
        Arg::new("flag")
            .long("flag")
            .action(clap::ArgAction::Append)
    );

let matches = cmd.try_get_matches_from(["mycmd", "--flag", "value", "--flag" "value2"]).unwrap();
assert!(matches.contains_id("flag"));
assert_eq!(
    matches.get_many::<String>("flag").unwrap_or_default().map(|v| v.as_str()).collect::<Vec<_>>(),
    vec!["value", "value2"]
);

// derive 风格
#[derive(Parser, Debug)]
struct AddArgs {
    name: Vec<String>,
}
#[derive(Parser, Debug)]
struct RemoveArgs {
    #[arg(short, long)]
    force: bool,
    name: Vec<String>,
}


// num_args
let cmd = Command::new("prog")
    .arg(Arg::new("file")
        .action(ArgAction::Set)
        .num_args(2)
        .short('F'));
let m = cmd.clone()
    .get_matches_from(vec![
        "prog", "-F", "in-file", "out-file"
    ]);
assert_eq!(
    m.get_many::<String>("file").unwrap_or_default().map(|v| v.as_str()).collect::<Vec<_>>(),
    vec!["in-file", "out-file"]
);
let res = cmd.clone()
    .try_get_matches_from(vec![
        "prog", "-F", "file1"
    ]);
assert_eq!(res.unwrap_err().kind(), ErrorKind::WrongNumberOfValues);
```

Option 类型 Arg 的实现:

-   builder 风格: 为 Arg 设置 .required(false);
-   derive 风格: Option&lt;Vec&lt;T&gt;&gt;, clap 自动调用 .action(ArgAction::Append).required(false)
    -   默认为 T 自动添加  `.value_parser(value_parser!(T))`, 如果不符合预期, 如 T 不是 Rust 原始类型,
        则需要为自定义类型实现 ValueParserFactory;

<!--listend-->

```rust
let mut cmd = clap::Command::new("raw")
    .arg(
        clap::Arg::new("append")
            .value_parser(clap::builder::FalseyValueParser::new())
            .required(false)
    );
```


## <span class="section-num">2</span> builder {#builder}

核心 Struct：

1.  Command：
2.  Arg：
3.  ArgGroup：
4.  ValueParser：作为 Arg.value_parser( valueParser) 的输入;

command!(), arg!(), value_parser!() 宏可以快速创建这些对象。


## <span class="section-num">3</span> derive {#derive}

通过 derive macro 和 attr macro 来声明式定义命令和参数，[文档](https://docs.rs/clap/latest/clap/_derive/index.html)：

derive macro：

1.  Parser
2.  Args/Subcommand
3.  ValueEnum

attr macro：

1.  command
2.  arg

示例: <https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html#documentation-derive-tutorial>

Parser trait:

-   Parser 是 FromArgMatches + CommandFactory 子类型, FromArgsMathes trait 实现从 struct ArgMatches 来生成自身对象;
-   使用 #[derive(Parser)] 来定义命令行解构的入口类型;
-   parse() 函数默认从 std::env::args_os 获取命令行参数, 用户也可以使用 parse_from() 函数来传入其他命令行字符串来源;

<!--listend-->

```rust
pub trait Parser: FromArgMatches + CommandFactory + Sized {
    // Provided methods
    fn parse() -> Self { ... }

    fn try_parse() -> Result<Self, Error> { ... }
    fn parse_from<I, T>(itr: I) -> Self
       where I: IntoIterator<Item = T>,
             T: Into<OsString> + Clone { ... }
    fn try_parse_from<I, T>(itr: I) -> Result<Self, Error>
       where I: IntoIterator<Item = T>,
             T: Into<OsString> + Clone { ... }
    fn update_from<I, T>(&mut self, itr: I)
       where I: IntoIterator<Item = T>,
             T: Into<OsString> + Clone { ... }
    fn try_update_from<I, T>(&mut self, itr: I) -> Result<(), Error>
       where I: IntoIterator<Item = T>,
             T: Into<OsString> + Clone { ... }
}
```

示例:

```rust
use clap::Parser;

/// Simple program to greet a person
#[derive(Parser, Debug)]
#[command(author, version, about, long_about = None)] // 1. 从 Cargo.toml 文件中获取缺省值
#[command(name = "MyApp")] // 2. 或者指定缺省值
struct Args {
    /// Name of the person to greet
    #[arg(short, long)]
    name: String,

    /// Number of times to greet
    #[arg(short, long, default_value_t = 1)]
    count: u8,
}

fn main() {
    let args = Args::parse(); // std::env::args_os
    for _ in 0..args.count {
        println!("Hello {}!", args.name)
    }
}

// 例子 2
use clap::Parser;

#[derive(Parser)]
#[command(name = "MyApp")]
#[command(author = "Kevin K. <kbknapp@gmail.com>")]
#[command(version = "1.0")]
#[command(about = "Does awesome things", long_about = None)]
struct Cli {
    #[arg(long)]
    two: String,
    #[arg(long)]
    one: String,
}

fn main() {
    let cli = Cli::parse();

    println!("two: {:?}", cli.two);
    println!("one: {:?}", cli.one);
}

// $ ./02_apps_derive --help
// Does awesome things // about

// Usage: 02_apps_derive[EXE] --two <TWO> --one <ONE>

// Options:
//       --two <TWO>
//       --one <ONE>
//   -h, --help       Print help
//   -V, --version    Print version

// $ 02_apps_derive --version  // name + version
// MyApp 1.0
```

Parser 下：

1.  \#[command] 可以使用任何 [Command builder](https://docs.rs/clap/latest/clap/struct.Command.html) 方法, 如 Command::next_line_help:
2.  \#[arg] 可以使用任何 Args builder 方法，如 long

<!--listend-->

```rust
let m = Command::new("My Program")
    .author("Me, me@mail.com")
    .version("1.0.2")
    .about("Explains in brief what the program does")
    .arg(
        Arg::new("in_file")
    )
    .after_help("Longer explanation to appear after the options when \
                 displaying the help information from --help or -h")
    .get_matches();

// Your program logic starts here...

use clap::Parser;
#[derive(Parser)]
#[command(author, version, about, long_about = None)] // auth/version/about 等均为 Command builder 方法
#[command(next_line_help = true)] // Command builder 方法
struct Cli {
    #[arg(long)] // Arg builder 方法
    two: String,
    #[arg(long)]
    one: String,
}

fn main() {
    let cli = Cli::parse();
    println!("two: {:?}", cli.two);
    println!("one: {:?}", cli.one);
}
```

位置参数：没有指定任何 clip 相关的 attr 的 filed 为位置参数；

```rust
use clap::Parser;

#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Cli {
    name: Option<String>, // 没有任何 attribute, 所以作为命令行位置参数
    name2: Vec<String>,  // 同样是命令行位置参数，但是可以指定多个（空格分隔）
}

fn main() {
    let cli = Cli::parse();
    println!("name: {:?}", cli.name.as_deref());
}
```

Arg：需要通过 #[arg] 来修饰 field:

-   Arg.action() 的参数 ArgAction 默认为 `Set/SetTrue`, 对于 Vec 等类型需要明确设置为 `Append`:
    <https://docs.rs/clap/latest/clap/enum.ArgAction.html>
-   Arg field 的类型要求:
    -   Vec&lt;XX&gt; : 可以指定多次 flag, 各参数值被 Append 到 Vec 中;
    -   Option&lt;XX&gt;: 表示该 field 是可选的(默认是必选);
    -   clap 自动为各 field 添加 XX 的 value_parser!(XX) 配置, 所以 XX 必须是实现 ValueParser 的类型.

<!--listend-->

```rust
use clap::Parser;

#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Cli {
    #[arg(short = 'n')]
    #[arg(long = "name")]  // 可以使用 clap::builder::Arg 的各种方法来设置 arg 的参数
    #[arg(short, long)] // 根据 filed name 自动推断
    name: Option<String>, // filed 类型可以是任何 clip 支持的类型

    #[arg(short, long)]
    verbose: bool,
}

fn main() {
    let cli = Cli::parse();
    println!("name: {:?}", cli.name.as_deref());
}
```

使用 Fn 闭包来实现 ValueParser 的例子:

```rust
// https://github.com/tokio-rs/mini-redis/blob/master/src/bin/cli.rs#L32

fn bytes_from_str(src: &str) -> Result<Bytes, Infallible> {
    Ok(Bytes::from(src.to_string()))
}

#[derive(Subcommand, Debug)]
enum Command {
    Ping {
        /// Message to ping
        #[clap(value_parser = bytes_from_str)]
        msg: Option<Bytes>,
    },
    /// Get the value of key.
    Get {
        /// Name of key to get
        key: String,
    },
    /// Set key to hold the string value.
    Set {
        /// Name of key to set
        key: String,

        /// Value to set.
        #[clap(value_parser = bytes_from_str)]
        value: Bytes,

        /// Expire the value after specified amount of time
        #[clap(value_parser = duration_from_ms_str)]
        expires: Option<Duration>,
    },
    ///  Publisher to send a message to a specific channel.
    Publish {
        /// Name of channel
        channel: String,

        #[clap(value_parser = bytes_from_str)]
        /// Message to publish
        message: Bytes,
    },
    /// Subscribe a client to a specific channel or channels.
    Subscribe {
        /// Specific channel or channels
        channels: Vec<String>,
    },
}
```

对于 derive 风格的 subcommand，需要使用 enum 类型，每一个 struct variant 都是一个 subcommand，struct
的 field 为 subcommand 的 args：

-   \#[derive(Subcommand)] 只支持 enum 类型, 所以只能通过 enum 来定义 subcommnad；
-   \#[derive(Args)] 只支持 non-tuple struct 类型，所以只能通过 struct field 来定义 Args；
-   \#[command(flatten)] 有两种使用场景：
    1.  在 Parser/Args 中： <https://rustdoc.swc.rs/clap/trait.Args.html%EF%BC%8C%E8%BF%99%E6%97%B6> filed type 必须实现 Args；
    2.  在 Subcommand 中：<https://rustdoc.swc.rs/clap/trait.Subcommand.html%EF%BC%8C%E8%BF%99%E6%97%B6%E5%BF%85%E9%A1%BB%E5%9C%A8> enum 中使用，而且 field type 必须实现 Subcommand；

<!--listend-->

```rust
fn main() {
    use clap::{Args, Parser, Subcommand};
    use std::fmt::Debug;

    /// Doc comment
    #[derive(Parser, Debug)]
    struct Cli {
        #[command(flatten)] // Args 场景下，flatten 的 field type 必须是 struct 类型
        delegate: Struct,  // Struct 必须实现 Args，将它的 field 作为 Args

        #[command(subcommand)] // 必须修饰 enum 类型
        command: Command,      // Command 必须是 enum struct 类型
    }

    /// Doc comment
    #[derive(Args, Debug)] // Struct必须实现 Args
    struct Struct {
        /// Doc comment
        field: u8,
    }

    /// Doc comment
    #[derive(Subcommand, Debug)] // Subcommand 必须是 enum 类型
    enum Command {
        /// Doc comment
        Variant1(Struct), // 每一个 Variant 都是一个 subcommand

        /// Doc comment
        Variant2 {
            // Variant struct 的 field 对应 subcommand 的 args
            /// Doc comment
            field: u8,
        },

        // https://rustdoc.swc.rs/clap/trait.Subcommand.html
        #[command(flatten)] // SubCommand 场景下也可以使用 flatten，但必须是实现 Subcommand 的 enum 类型.
        Variant3(SubCmd), // SubCmd 必须实现 Subcommand，而 Subcommand 必须是 enum 类型
    }

    /// Doc comment
    #[derive(Subcommand, Debug)]
    enum SubCmd {
        /// Doc comment
        Sub1 { field: u8 },
    }

    let cli = Cli::parse();
    println!("{:?}", cli);
}

// zj@a:~/codes/rust/clap/clap-demo$ ./target/debug/clap-demo -h
// Doc comment

// Usage: clap-demo <FIELD> <COMMAND>

// Commands:
//   variant1  Doc comment
//   variant2  Doc comment
//   sub1      Doc comment
//   help      Print this message or the help of the given subcommand(s)

// Arguments:
//   <FIELD>  Doc comment

// Options:
//   -h, --help  Print help
// zj@a:~/codes/rust/clap/clap-demo$
```

在使用 drive macro 来定义 Arg 时，使用 #[arg(value_enum)] 来定义枚举 field，clap 自动调用
clap::EnumValueParser

-   可以使用 #[arg(default_value_t)] 来自动实现 Display；

<!--listend-->

```rust
pub enum ColorChoice {
    Auto,
    Always,
    Never,
}

// Usage
let mut cmd = clap::Command::new("raw")
    .arg(
        clap::Arg::new("color")
            .value_parser(clap::builder::EnumValueParser::<ColorChoice>::new())
            .required(true)
    );
let m = cmd.try_get_matches_from_mut(["cmd", "always"]).unwrap();
let port: ColorChoice = *m.get_one("color")
    .expect("required");
assert_eq!(port, ColorChoice::Always);

// 例子2
use clap::{Parser, ValueEnum};

#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Cli {
    /// What mode to run the program in
    #[arg(value_enum)]
    mode: Mode,
}

#[derive(Copy, Clone, PartialEq, Eq, PartialOrd, Ord, ValueEnum)]
enum Mode {
    /// Run swiftly
    Fast,
    /// Crawl slowly but steadily
    ///
    /// This paragraph is ignored because there is no long help text for possible values.
    Slow,
}

fn main() {
    let cli = Cli::parse();

    match cli.mode {
        Mode::Fast => {
            println!("Hare");
        }
        Mode::Slow => {
            println!("Tortoise");
        }
    }
}


use clap::Parser;
#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Cli {
    #[arg(default_value_t = 2020)]
    port: u16,
}

fn main() {
    let cli = Cli::parse();
    println!("port: {:?}", cli.port);
}
```

参考：

1.  <https://github.com/mrjackwills/havn/blob/main/src/parse_arg.rs>
2.  <https://github.com/franticxx/dn/blob/main/src/cli/cli.rs>
