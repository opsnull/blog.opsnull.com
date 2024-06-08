---
title: "regex"
author: ["zhangjun"]
lastmod: 2024-06-08T21:57:43+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 13
---

[regex crate](https://docs.rs/regex/latest/regex/) 主要类型：Regex、Match、Capture。

Regex 是编译后的正则表达式对象，提供如下方法：

1.  Regex::new : 使用缺省配置对正则表达式字符串进行编译，返回一个 Regex 类型对象。使用 `RegexBuilder`
    可以创建一个自定义配置的 Regex 对象；
2.  Regex::is_match() : 判断正则表达式是否匹配传入的字符串；
3.  Regex::find(): 返回匹配的 byte offset，而 Regex::find_iter() 返回一个匹配所有内容的迭代器；
4.  Regex::captures() : 返回一个 Captures，同时返回一个匹配整个正则表达式的 byte offsets，以及各
    capture group 的 byte offsets。Regex::captures_iter() 返回这些匹配的迭代器。

Regex::is_match(): 返回传入的字符串是否匹配正则表达式；

-   默认是任意位置匹配；

<!--listend-->

```rust
use regex::Regex;

let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();
assert!(re.is_match("2010-03-14"));
```

Regex::find(): 返回第一个 Match 对象

-   Match 对象可以提供匹配的开始 m.start()、结束 m.end() 的 byte offset，以及匹配的字符串本身 m.as_str();

<!--listend-->

```rust
let re = Regex::new(r"\p{Greek}+").unwrap();
let hay = "Greek: αβγδ";
let m = re.find(hay).unwrap();
assert_eq!(7, m.start());
assert_eq!(15, m.end());
assert!(!m.is_empty());
assert_eq!(8, m.len());
assert_eq!(7..15, m.range());
assert_eq!("αβγδ", m.as_str());
```

Regex::find_iter(): 返回一个迭代器，每次迭代返回 Match 类型对象，分别匹配不重叠的子串；

```rust
use regex::Regex;

let re = Regex::new(r"[0-9]{4}-[0-9]{2}-[0-9]{2}").unwrap();
let hay = "What do 1865-04-14, 1881-07-02, 1901-09-06 and 1963-11-22 have in common?";
// 'm' is a 'Match', and 'as_str()' returns the matching part of the haystack.
let dates: Vec<&str> = re.find_iter(hay).map(|m| m.as_str()).collect();
assert_eq!(dates, vec![
    "1865-04-14",
    "1881-07-02",
    "1901-09-06",
    "1963-11-22",
]);
```

Regex::captures(): 返回一个 Captures 类型对象，它是 Match 对象的封装，包含第一个匹配的匹配组信息。

-   Captures 类型支持使用 index 来获得各匹配组，index 的 key 可以是匹配组的编号（0 表示整个正则表达式的匹配）或匹配组的名称；
-   Captures 类型的 c.extract() 方法返回本次匹配的完整子字符串和从 1 开始的所有匹配组数组；
-   Captures 类型的 c.get(index) 方法指定匹配组索引的 Match 对象；
-   Captures 类型的 c.name(name) 方法指定匹配组名称的 Match 对象；（建议）
    -   index["name"] 和 c.name("name") 的区别是返回的 Match 对象的 as_str() 的 lifetime 不一致，前者和
        Captures 的 lifetime 一致，而后者和传入的字符串一致（建议使用 c.name("name))；
-   Captures 类型的 c.expand() 方法使用匹配组来替换传入的字符串模板，结果字符串存入指定的 String

<!--listend-->

```rust
use regex::Regex;

// index 操作同时支持命名组的 index 和 name
let re = Regex::new(r"(?<first>\w)(\w)(?:\w)\w(?<last>\w)").unwrap();
let caps = re.captures("toady").unwrap();
assert_eq!("toady", &caps[0]);
assert_eq!("t", &caps["first"]);
assert_eq!("o", &caps[2]);
assert_eq!("y", &caps["last"]);

// Captures 的 extract() 方法，本次匹配的子字符串和所有匹配组数组
let re = Regex::new(r"([0-9]{4})-([0-9]{2})-([0-9]{2})").unwrap();
let hay = "On 2010-03-14, I became a Tenneessee lamb.";
let Some((full, [year, month, day])) = re.captures(hay).map(|caps| caps.extract()) else { return };
assert_eq!("2010-03-14", full);
assert_eq!("2010", year);
assert_eq!("03", month);
assert_eq!("14", day);

// get() 指定 index
let re = Regex::new(r"[a-z]+(?:([0-9]+)|([A-Z]+))").unwrap();
let caps = re.captures("abc123").unwrap();
let substr1 = caps.get(1).map_or("", |m| m.as_str());
let substr2 = caps.get(2).map_or("", |m| m.as_str());
assert_eq!(substr1, "123");
assert_eq!(substr2, "");

// name() 指定 name
let re = Regex::new( r"[a-z]+(?:(?<numbers>[0-9]+)|(?<letters>[A-Z]+))", ).unwrap();
let caps = re.captures("abc123").unwrap();
let numbers = caps.name("numbers").map_or("", |m| m.as_str());
let letters = caps.name("letters").map_or("", |m| m.as_str());
assert_eq!(numbers, "123");
assert_eq!(letters, "");

// expand() 对字符串进行替换
let re = Regex::new( r"(?<day>[0-9]{2})-(?<month>[0-9]{2})-(?<year>[0-9]{4})",).unwrap();
let hay = "On 14-03-2010, I became a Tenneessee lamb.";
let caps = re.captures(hay).unwrap();
let mut dst = String::new();
caps.expand("year=$year, month=$month, day=$day", &mut dst);
assert_eq!(dst, "year=2010, month=03, day=14");
```

Regex::captures_iter()： 返回一个迭代器，每次迭代返回 Cpatures 类型对象：

```rust
use regex::Regex;

let re = Regex::new(r"'([^']+)'\s+\(([0-9]{4})\)").unwrap();
let hay = "'Citizen Kane' (1941), 'The Wizard of Oz' (1939), 'M' (1931).";
let mut movies = vec![];

for (_, [title, year]) in re.captures_iter(hay).map(|c| c.extract()) {
    movies.push((title, year.parse::<i64>()?));
}
assert_eq!(movies, vec![
    ("Citizen Kane", 1941),
    ("The Wizard of Oz", 1939),
    ("M", 1931),
]);

// 命名组
let re = Regex::new(r"'(?<title>[^']+)'\s+\((?<year>[0-9]{4})\)").unwrap();
let hay = "'Citizen Kane' (1941), 'The Wizard of Oz' (1939), 'M' (1931).";
let mut it = re.captures_iter(hay);

let caps = it.next().unwrap();
assert_eq!(&caps["title"], "Citizen Kane");
assert_eq!(&caps["year"], "1941");

let caps = it.next().unwrap();
assert_eq!(&caps["title"], "The Wizard of Oz");
assert_eq!(&caps["year"], "1939");

let caps = it.next().unwrap();
assert_eq!(&caps["title"], "M");
assert_eq!(&caps["year"], "1931");
```

Regex::split(): 使用正则表达式对字符串进行拆分，返回拆分后的 Vec 数组；

```rust
let re = Regex::new(r"[ \t]+").unwrap();
let hay = "a b \t  c\td    e";
let fields: Vec<&str> = re.split(hay).collect();
assert_eq!(fields, vec!["a", "b", "c", "d", "e"]);

// 连续的匹配会返回一个空字符串
let re = Regex::new(r"X").unwrap();
let hay = "lionXXtigerXleopard";
let got: Vec<&str> = re.split(hay).collect();
assert_eq!(got, vec!["lion", "", "tiger", "leopard"]);

let re = Regex::new(r"0").unwrap();
let hay = "010";
let got: Vec<&str> = re.split(hay).collect();
assert_eq!(got, vec!["", "1", ""]);
```

Regex::replace()/replace_all(): 匹配的模式替换字符串内容；

```rust
pub fn replace<'h, R: Replacer>(
    &self,
    haystack: &'h str,
    rep: R
) -> Cow<'h, str>

// 以下类型实现了 Replacer trait
impl Replacer for String
impl<'a> Replacer for &'a Cow<'a, str>
impl<'a> Replacer for &'a str
impl<'a> Replacer for Cow<'a, str>

use regex::Regex;
let re = Regex::new(r"[^01]+").unwrap();
assert_eq!(re.replace("1078910", ""), "1010");


let re = Regex::new(r"([^,\s]+),\s+(\S+)").unwrap();
let result = re.replace("Springsteen, Bruce", |caps: &Captures| { format!("{} {}", &caps[2], &caps[1]) });
assert_eq!(result, "Bruce Springsteen");

let re = Regex::new(r"(?<last>[^,\s]+),\s+(?<first>\S+)").unwrap();
let result = re.replace("Springsteen, Bruce", "$first $last");
// let result = re.replace("deep fried", "${first}_$second");
assert_eq!(result, "Bruce Springsteen");

let re = Regex::new(r"(?<last>[^,\s]+),\s+(\S+)").unwrap();
let result = re.replace("Springsteen, Bruce", NoExpand("$2 $last"));
assert_eq!(result, "$2 $last");

// replace_all()
use regex::Regex;

let re = Regex::new(r"(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})").unwrap();
let before = "1973-01-05, 1975-08-25 and 1980-10-18";
let after = re.replace_all(before, "$m/$d/$y");
assert_eq!(after, "01/05/1973, 08/25/1975 and 10/18/1980");
```

正则表达式 verbose 模式：可以在正则表达式中添加注释等内容，编译时忽略：

```rust
use regex::Regex;

let re = Regex::new(r"(?x)
  (?P<y>\d{4}) # the year, including all Unicode digits
  -
  (?P<m>\d{2}) # the month, including all Unicode digits
  -
  (?P<d>\d{2}) # the day, including all Unicode digits
").unwrap();

let before = "1973-01-05, 1975-08-25 and 1980-10-18";
let after = re.replace_all(before, "$m/$d/$y");
assert_eq!(after, "01/05/1973, 08/25/1975 and 10/18/1980");
```

使用 once_cell 或 lazy_static 来避免重复编译正则表达式：

```rust
use {
    once_cell::sync::Lazy,
    regex::Regex,
};

fn some_helper_function(haystack: &str) -> bool {
    static RE: Lazy<Regex> = Lazy::new(|| Regex::new(r"...").unwrap());
    RE.is_match(haystack)
}

fn main() {
    assert!(some_helper_function("abc"));
    assert!(!some_helper_function("ac"));
}
```

编译生成的 Regexp 对象可以在多线程环境中共享。

regex crate 提供了 bytes module，用于对 &amp;[u8] 而非 &amp;str 字符串进行模式匹配。

正则语法：

1.  匹配单字符：
    ```rust
       .             any character except new line (includes new line with s flag)
       [0-9]         any ASCII digit
       \d            digit (\p{Nd})
       \D            not digit
       \pX           Unicode character class identified by a one-letter name
       \p{Greek}     Unicode character class (general category or script)
       \PX           Negated Unicode character class identified by a one-letter name
       \P{Greek}     negated Unicode character class (general category or script)
    ```
2.  字符类：
    ```rust
       [xyz]         A character class matching either x, y or z (union).
       [^xyz]        A character class matching any character except x, y and z.
       [a-z]         A character class matching any character in range a-z.
       [[:alpha:]]   ASCII character class ([A-Za-z])
       [[:^alpha:]]  Negated ASCII character class ([^A-Za-z])
       [x[^xyz]]     Nested/grouping character class (matching any character except y and z)

       // 取交集
       [a-y&&xyz]    Intersection (matching x or y)
       [0-9&&[^4]]   Subtraction using intersection and negation (matching 0-9 except 4)
       // 取差集
       [0-9--4]      Direct subtraction (matching 0-9 except 4)
       // 取对称差
       [a-g~~b-h]    Symmetric difference (matching `a` and `h` only)
       [\[\]]        Escaping in character classes (matching [ or ])
       [a&&b]        An empty character class matching nothing
    ```
3.  组合：| 的优先级最低
    ```rust
       xy    concatenation (x followed by y)
       x|y   alternation (x or y, prefer x)
    ```

4.  重复：
    ```rust
       // 贪婪重复
       x*        zero or more of x (greedy)
       x+        one or more of x (greedy)
       x?        zero or one of x (greedy)

       // 非贪婪重复
       x*?       zero or more of x (ungreedy/lazy)
       x+?       one or more of x (ungreedy/lazy)
       x??       zero or one of x (ungreedy/lazy)


       x{n,m}    at least n x and at most m x (greedy)
       x{n,}     at least n x (greedy)
       x{n}      exactly n x
       x{n,m}?   at least n x and at most m x (ungreedy/lazy)
       x{n,}?    at least n x (ungreedy/lazy)
       x{n}?     exactly n x
    ```
5.  匹配空白
    ```rust
       ^               the beginning of a haystack (or start-of-line with multi-line mode)
       $               the end of a haystack (or end-of-line with multi-line mode)

       \A              only the beginning of a haystack (even with multi-line mode enabled)
       \z              only the end of a haystack (even with multi-line mode enabled)

       \b              a Unicode word boundary (\w on one side and \W, \A, or \z on other)
       \B              not a Unicode word boundary

       \b{start}, \<   a Unicode start-of-word boundary (\W|\A on the left, \w on the right)
       \b{end}, \>     a Unicode end-of-word boundary (\w on the left, \W|\z on the right))
       \b{start-half}  half of a Unicode start-of-word boundary (\W|\A on the left)
       \b{end-half}    half of a Unicode end-of-word boundary (\W|\z on the right)
    ```

6.  分组
    ```rust
       // 编号分组
       (exp)          numbered capture group (indexed by opening parenthesis)

       // 命名分组
       (?P<name>exp)  named (also numbered) capture group (names must be alpha-numeric)
       (?<name>exp)   named (also numbered) capture group (names must be alpha-numeric)

       // 非分组
       (?:exp)        non-capturing group

       // 设置当前分组的 flags
       (?flags)       set flags within current group

       // 设置当前整个正则表达式 exp 的 flags
       (?flags:exp)   set flags for exp (non-capturing)
    ```
    可以设置的 flags 如下：
    ```rust
       i     case-insensitive: letters match both upper and lower case
       m     multi-line mode: ^ and $ match begin/end of line
       s     allow . to match \n
       R     enables CRLF mode: when multi-line mode is enabled, \r\n is used
       U     swap the meaning of x* and x*?
       u     Unicode support (enabled by default)
       x     verbose mode, ignores whitespace and allow line comments (starting with `#`)
    ```
    示例：

    -   ?xy 设置开启 flags xy；?-xy 设置关闭 flags xy；

    <!--listend-->

    ```rust
         let re = Regex::new(r"(?i)a+(?-i)b+").unwrap();
         let m = re.find("AaAaAbbBBBb").unwrap();
         assert_eq!(m.as_str(), "AaAaAbb");

         let re = Regex::new(r"(?m)^line \d+").unwrap();
         let m = re.find("line one\nline 2\n").unwrap();
         assert_eq!(m.as_str(), "line 2");
    ```

7.  转义序列
    ```rust
       \*              literal *, applies to all ASCII except [0-9A-Za-z<>]
       \a              bell (\x07)
       \f              form feed (\x0C)
       \t              horizontal tab
       \n              new line
       \r              carriage return
       \v              vertical tab (\x0B)

       \A              matches at the beginning of a haystack
       \z              matches at the end of a haystack

       \b              word boundary assertion
       \B              negated word boundary assertion

       \b{start}, \<   start-of-word boundary assertion
       \b{end}, \>     end-of-word boundary assertion

       \b{start-half}  half of a start-of-word boundary assertion
       \b{end-half}    half of a end-of-word boundary assertion

       \123            octal character code, up to three digits (when enabled)
       \x7F            hex character code (exactly two digits)

       \x{10FFFF}      any hex character code corresponding to a Unicode code point
       \u007F          hex character code (exactly four digits)
       \u{7F}          any hex character code corresponding to a Unicode code point
       \U0000007F      hex character code (exactly eight digits)
       \U{7F}          any hex character code corresponding to a Unicode code point
       \p{Letter}      Unicode character class
       \P{Letter}      negated Unicode character class
       \d, \s, \w      Perl character class
       \D, \S, \W      negated Perl character class
    ```

8.  Perl 字符类

<!--listend-->

```rust
\d     digit (\p{Nd})
\D     not digit
\s     whitespace (\p{White_Space})
\ S     not whitespace
\w     word character (\p{Alphabetic} + \p{M} + \d + \p{Pc} + \p{Join_Control})
\W     not word character
```

1.  ASCII 字符类
    ```rust
       [[:alnum:]]    alphanumeric ([0-9A-Za-z])
       [[:alpha:]]    alphabetic ([A-Za-z])
       [[:ascii:]]    ASCII ([\x00-\x7F])
       [[:blank:]]    blank ([\t ])
       [[:cntrl:]]    control ([\x00-\x1F\x7F])
       [[:digit:]]    digits ([0-9])
       [[:graph:]]    graphical ([!-~])
       [[:lower:]]    lower case ([a-z])
       [[:print:]]    printable ([ -~])
       [[:punct:]]    punctuation ([!-/:-@\[-`{-~])
       [[:space:]]    whitespace ([\t\n\v\f\r ])
       [[:upper:]]    upper case ([A-Z])
       [[:word:]]     word characters ([0-9A-Za-z_])
       [[:xdigit:]]   hex digit ([0-9A-Fa-f])
    ```
