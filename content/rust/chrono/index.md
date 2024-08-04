---
title: "chrono"
author: ["zhangjun"]
lastmod: 2024-08-04T17:09:16+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 3
---

[chrono crate](https://docs.rs/chrono/latest/chrono/) 提供了 Date/Time 相关类型和操作。

chrono crate 默认没有自带 Timezone data，需要使用 chrono-tz crate 或 tzfile crate 来提供完整功能。它们都提供了 chrono 的 TimeZone trait 的实现。chrono-tz 自带 timezone 数据库，而 tzfile 使用的是系统的
tz 数据库（/usr/share/zoneinfo）。

```rust
use chrono::{TimeZone, Utc};
use chrono_tz::US::Pacific;

let pacific_time = Pacific.ymd(1990, 5, 6).and_hms(12, 30, 45);
let utc_time = pacific_time.with_timezone(&Utc);
assert_eq!(utc_time, Utc.ymd(1990, 5, 6).and_hms(19, 30, 45));

use chrono::{TimeZone, NaiveDate};
use chrono_tz::Africa::Johannesburg;

let naive_dt = NaiveDate::from_ymd(2038, 1, 19).and_hms(3, 14, 08);
let tz_aware = Johannesburg.from_local_datetime(&naive_dt).unwrap();
assert_eq!(tz_aware.to_string(), "2038-01-19 03:14:08 SAST");
```

创建 Date 和 Time 对象：

1.  `NaiveDate/NaiveTime/NaiveDateTime`: 不带时区信息的 Date/Time/DateTime
2.  `DateTime<Utc>`: 带有时区信息的日期和时间：
    -   DateTime/Date 类型包含两个转换为 naive 版本的方法：naive_local() 和 naive_utc()
3.  方法中的 yo 和 ordinal 表示一年中的天数。

DateTime使用关联的, 实现 TimeZone trait 的对象来感知时区信息，在和 UTC 转换时需要使用该时区兴起。
chrono 提供了三种常见的 TimeZone 实现：

1.  Utc：UTC time zone；
2.  Local：系统本地的 time zone；
3.  FixedOffset：通过指定任意时区偏移量来自定义时区；

DateTime::with_timezone() 方法返回指定的 TimeZone 的 DateTime 对象。

-   可以传入 FixedOffset 对象，来返回任意 TimeZone 的 DateTime 对象。

<!--listend-->

```rust
use chrono::prelude::*;

fn main() {
    let date = NaiveDate::from_ymd(2023, 4, 16);
    let time = NaiveTime::from_hms(12, 34, 56);
    let datetime = NaiveDateTime::new(date, time);

    let utc_datetime = Utc::now(); // 返回一个 DateTime<Utc> 类型对象
    let local_datetime = Local::now(); // 返回一个 DateTime<Local> 类型对象

    let utc: DateTime<Utc> = Utc::now(); // e.g. `2014-11-28T12:45:59.324310806Z`
    let local: DateTime<Local> = Local::now(); // e.g. `2014-11-28T21:45:59.324310806+09:00`
    let dt = Utc.with_ymd_and_hms(2014, 7, 8, 9, 10, 11).unwrap(); // `2014-07-08T09:10:11Z`

    let offset = FixedOffset::east_opt(5 * 60 * 60).unwrap();
    let now_with_offset = Utc::now().with_timezone(&offset);

    NaiveDate::from_ymd_opt(2014, 7, 8)? .and_hms_opt(9, 10, 11)? .and_utc()

    println!("Date: {}", date);
    println!("Time: {}", time);
    println!("DateTime: {}", datetime);
    println!("UTC DateTime: {}", utc_datetime);
}
```

使用 DateTime 对象：

-   各种 from_XX() 方法可以创建 DateTime 对象；
-   各种 with_XX() 方法可以修改 DateTime 对象；
-   DateTime 可以和 TimeDelta 进行算术运算；

<!--listend-->

```rust
use chrono::prelude::*;
use chrono::TimeDelta;

// assume this returned `2014-11-28T21:45:59.324310806+09:00`:
let dt = FixedOffset::east_opt(9 * 3600)
    .unwrap()
    .from_local_datetime(
        &NaiveDate::from_ymd_opt(2014, 11, 28)
            .unwrap()
            .and_hms_nano_opt(21, 45, 59, 324310806)
            .unwrap(),
    )
    .unwrap();

// property accessors
assert_eq!((dt.year(), dt.month(), dt.day()), (2014, 11, 28));
assert_eq!((dt.month0(), dt.day0()), (10, 27)); // for unfortunate souls
assert_eq!((dt.hour(), dt.minute(), dt.second()), (21, 45, 59));
assert_eq!(dt.weekday(), Weekday::Fri);
assert_eq!(dt.weekday().number_from_monday(), 5); // Mon=1, ..., Sun=7
assert_eq!(dt.ordinal(), 332); // the day of year
assert_eq!(dt.num_days_from_ce(), 735565); // the number of days from and including Jan 1, 1

// time zone accessor and manipulation
assert_eq!(dt.offset().fix().local_minus_utc(), 9 * 3600);
assert_eq!(dt.timezone(), FixedOffset::east_opt(9 * 3600).unwrap());
assert_eq!(
    dt.with_timezone(&Utc), // 转换为 Utc 时区
    NaiveDate::from_ymd_opt(2014, 11, 28)
    .unwrap()
    .and_hms_nano_opt(12, 45, 59, 324310806)
    .unwrap()
    .and_utc()
);


// 各种 with_XX() 方法可以修改 DateTime
// a sample of property manipulations (validates dynamically)
assert_eq!(dt.with_day(29).unwrap().weekday(), Weekday::Sat); // 2014-11-29 is Saturday
assert_eq!(dt.with_day(32), None);
assert_eq!(dt.with_year(-300).unwrap().num_days_from_ce(), -109606); // November 29, 301 BCE

// arithmetic operations
let dt1 = Utc.with_ymd_and_hms(2014, 11, 14, 8, 9, 10).unwrap();
let dt2 = Utc.with_ymd_and_hms(2014, 11, 14, 10, 9, 8).unwrap();
```

解析和格式化 Date 和 Time 对象：

-   parse() : 可以用于从字符串创建 DateTime&lt;FixedOffset&gt;, DateTime&lt;Utc&gt; 和 DateTime&lt;Local&gt; 值；
-   parse_from_str() : 来创建 NaiveDate/DateTime&lt;FixedOffset&gt; 对象，字符串必须包含 offset，否则不能
    guess；
-   DateTime::parse_from_rfc2822 和 DateTime::parse_from_rfc3339：用于解析特定的格式字符串；
-   format() 将 NaiveDate/DateTime 对象格式化为字符串；
-   parse 和 format 的格式字符串语法参考 [chrono::format::strftime module](https://docs.rs/chrono/latest/chrono/format/strftime/index.html#specifiers).

<!--listend-->

```rust
use chrono::{NaiveDate, NaiveDateTime, DateTime, Utc, LocalResult};

fn main() {
    // Parsing a date from a string
    let date_str = "2023-04-16";
    let parsed_date = NaiveDate::parse_from_str(date_str, "%Y-%m-%d").unwrap();
    println!("Parsed Date: {}", parsed_date);

    // Parsing a datetime from a string
    let datetime_str = "2023-04-16T12:34:56Z";
    let parsed_datetime = DateTime::parse_from_rfc3339(datetime_str).unwrap();
    println!("Parsed DateTime: {}", parsed_datetime);

    // Formatting a date as a string
    let formatted_date = parsed_date.format("%A, %B %e, %Y");
    println!("Formatted Date: {}", formatted_date);

    // Formatting a datetime as a string
    let formatted_datetime = parsed_datetime.format("%Y-%m-%dT%H:%M:%S%z");
    println!("Formatted DateTime: {}", formatted_datetime);
}

// 使用 parse() 和 parse_from_str() 来从字符串创建 DateTime
use chrono::prelude::*;

let dt = Utc.with_ymd_and_hms(2014, 11, 28, 12, 0, 9).unwrap();
let fixed_dt = dt.with_timezone(&FixedOffset::east_opt(9 * 3600).unwrap());

// method 1
assert_eq!("2014-11-28T12:00:09Z".parse::<DateTime<Utc>>(), Ok(dt.clone()));
assert_eq!("2014-11-28T21:00:09+09:00".parse::<DateTime<Utc>>(), Ok(dt.clone()));
assert_eq!("2014-11-28T21:00:09+09:00".parse::<DateTime<FixedOffset>>(), Ok(fixed_dt.clone()));

// method 2
assert_eq!(
    DateTime::parse_from_str("2014-11-28 21:00:09 +09:00", "%Y-%m-%d %H:%M:%S %z"),
    Ok(fixed_dt.clone())
);
assert_eq!(
    DateTime::parse_from_rfc2822("Fri, 28 Nov 2014 21:00:09 +0900"),
    Ok(fixed_dt.clone())
);
assert_eq!(DateTime::parse_from_rfc3339("2014-11-28T21:00:09+09:00"), Ok(fixed_dt.clone()));

// oops, the year is missing!
assert!(DateTime::parse_from_str("Fri Nov 28 12:00:09", "%a %b %e %T %Y").is_err());
// oops, the format string does not include the year at all!
assert!(DateTime::parse_from_str("Fri Nov 28 12:00:09", "%a %b %e %T").is_err());
// oops, the weekday is incorrect!
assert!(DateTime::parse_from_str("Sat Nov 28 12:00:09 2014", "%a %b %e %T %Y").is_err());



// 另一个例子
use chrono::prelude::*;

let dt = Utc.with_ymd_and_hms(2014, 11, 28, 12, 0, 9).unwrap();
assert_eq!(dt.format("%Y-%m-%d %H:%M:%S").to_string(), "2014-11-28 12:00:09");
assert_eq!(dt.format("%a %b %e %T %Y").to_string(), "Fri Nov 28 12:00:09 2014");
assert_eq!(
    dt.format_localized("%A %e %B %Y, %T", Locale::fr_BE).to_string(),
    "vendredi 28 novembre 2014, 12:00:09"
);

assert_eq!(dt.format("%a %b %e %T %Y").to_string(), dt.format("%c").to_string());

// 缺省的 to_string() 实现
assert_eq!(dt.to_string(), "2014-11-28 12:00:09 UTC");
assert_eq!(dt.to_rfc2822(), "Fri, 28 Nov 2014 12:00:09 +0000");
assert_eq!(dt.to_rfc3339(), "2014-11-28T12:00:09+00:00");

// Debug 实现
assert_eq!(format!("{:?}", dt), "2014-11-28T12:00:09Z");

// Note that milli/nanoseconds are only printed if they are non-zero
let dt_nano = NaiveDate::from_ymd_opt(2014, 11, 28)
    .unwrap()
    .and_hms_nano_opt(12, 0, 9, 1)
    .unwrap()
    .and_utc();
assert_eq!(format!("{:?}", dt_nano), "2014-11-28T12:00:09.000000001Z");
```

DateTime 和 EPOCH timestamp 间的相互转换：

-   使用 DateTime::from_timestamp(seconds，nanosecdonds) 来从 timestamp 创建 DateTime&lt;Utc&gt;;
-   使用 DateTime::timestamp() 来返回 Unix timestamp；

<!--listend-->

```rust
// We need the trait in scope to use Utc::timestamp().
use chrono::{DateTime, Utc};

// Construct a datetime from epoch:
let dt: DateTime<Utc> = DateTime::from_timestamp(1_500_000_000, 0).unwrap();
assert_eq!(dt.to_rfc2822(), "Fri, 14 Jul 2017 02:40:00 +0000");

// Get epoch value from a datetime:
let dt = DateTime::parse_from_rfc2822("Fri, 14 Jul 2017 02:40:00 +0000").unwrap();
assert_eq!(dt.timestamp(), 1_500_000_000);
```

日期、时间的算术运算和比较

```rust
use chrono::{NaiveDate, Duration};

fn main() {
    let date1 = NaiveDate::from_ymd(2023, 4, 16);
    let date2 = NaiveDate::from_ymd(2023, 5, 1);

    // Adding and subtracting durations
    let date_plus_one_week = date1 + Duration::days(7);
    let date_minus_one_month = date1 - Duration::days(30);
    println!("One week later: {}", date_plus_one_week);
    println!("One month earlier: {}", date_minus_one_month);

    // Comparing dates
    if date1 < date2 {
        println!("{} is earlier than {}", date1, date2);
    }

    // Calculating the difference between dates
    let duration = date2 - date1;
    println!("There are {} days between {} and {}", duration.num_days(), date1, date2);
}
```

chrono 提供了 TimeDelta 及其别名 Duration 类型，和标准库的 std::time::Durataion 的主要区别是：
TimeDelta 是有符号类型，而非无符号类型。

-   两者可以相互转换：TimeDelta::from_std() 和 TimeDelta::to_std();


## <span class="section-num">1</span> DateTime {#datetime}

DateTime 对象有一个泛型参数 Tz，类型是 TimeZone trait：

```rust
pub struct DateTime<Tz: TimeZone> { /* private fields */ }

impl<Tz: TimeZone> DateTime<Tz>

// 从 naive utc 和 offset 创建一个 DateTime
pub const fn from_naive_utc_and_offset(
    datetime: NaiveDateTime, // UTC Date and Time，可以从已有的 DateTime.naive_utc() 获得。
    offset: Tz::Offset // TimeZone offset, 可以从已有的 DateTime.offset() 获得。
) -> DateTime<Tz>

use chrono::{DateTime, Local};
let dt = Local::now();
// Get components
let naive_utc = dt.naive_utc();
let offset = dt.offset().clone();
// Serialize, pass through FFI... and recreate the `DateTime`:
let dt_new = DateTime::<Local>::from_naive_utc_and_offset(naive_utc, offset);
assert_eq!(dt, dt_new);

// 创建 DateTime<Utc> 类型的关联函数（构造函数）
impl DateTime<Utc>
pub const fn from_timestamp(secs: i64, nsecs: u32) -> Option<Self>
pub const fn from_timestamp_millis(millis: i64) -> Option<Self>
pub const fn from_timestamp_micros(micros: i64) -> Option<Self>
pub const fn from_timestamp_nanos(nanos: i64) -> Self
// 解析字符串，创建 DateTime<FixedOffset>类型的关联函数（构造函数）
impl DateTime<FixedOffset>
pub fn parse_from_rfc2822(s: &str) -> ParseResult<DateTime<FixedOffset>>
pub fn parse_from_rfc3339(s: &str) -> ParseResult<DateTime<FixedOffset>>
pub fn parse_from_str(s: &str, fmt: &str) -> ParseResult<DateTime<FixedOffset>>
pub fn parse_and_remainder<'a>( s: &'a str, fmt: &str ) -> ParseResult<(DateTime<FixedOffset>, &'a str)>

// 返回指定 Utc 或指定 TimeZone 的 DateTime
pub fn fixed_offset(&self) -> DateTime<FixedOffset>
pub const fn to_utc(&self) -> DateTime<Utc>
pub fn with_timezone<Tz2: TimeZone>(&self, tz: &Tz2) -> DateTime<Tz2>


pub fn date_naive(&self) -> NaiveDate
pub const fn naive_utc(&self) -> NaiveDateTime
pub fn naive_local(&self) -> NaiveDateTime
pub fn time(&self) -> NaiveTime

pub const fn timestamp(&self) -> i64
pub const fn timestamp_millis(&self) -> i64
pub const fn timestamp_micros(&self) -> i64
pub const fn timestamp_nanos_opt(&self) -> Option<i64>
pub const fn timestamp_subsec_millis(&self) -> u32
pub const fn timestamp_subsec_micros(&self) -> u32
pub const fn timestamp_subsec_nanos(&self) -> u32

// 返回关联的 TimeZone 或 Offset
pub const fn offset(&self) -> &Tz::Offset
pub fn timezone(&self) -> Tz

pub fn checked_add_signed(self, rhs: TimeDelta) -> Option<DateTime<Tz>>
pub fn checked_add_months(self, months: Months) -> Option<DateTime<Tz>>
pub fn checked_sub_signed(self, rhs: TimeDelta) -> Option<DateTime<Tz>>
pub fn checked_sub_months(self, months: Months) -> Option<DateTime<Tz>>
pub fn checked_add_days(self, days: Days) -> Option<Self>
pub fn checked_sub_days(self, days: Days) -> Option<Self>

// 返回两个 DateTime 的时间差
pub fn signed_duration_since<Tz2: TimeZone>( self, rhs: impl Borrow<DateTime<Tz2>> ) -> TimeDelta
pub fn years_since(&self, base: Self) -> Option<u32>

// 格式化显示
pub fn to_rfc2822(&self) -> String
pub fn to_rfc3339(&self) -> String
pub fn to_rfc3339_opts(&self, secform: SecondsFormat, use_z: bool) -> String

// 设置 time 部分，如果 time 不对则返回  LocalResult::None
pub fn with_time(&self, time: NaiveTime) -> LocalResult<Self>

impl<Tz: TimeZone> DateTime<Tz> where Tz::Offset: Display,
pub fn format_with_items<'a, I, B>(&self, items: I) -> DelayedFormat<I>
    where
    I: Iterator<Item = B> + Clone,
    B: Borrow<Item<'a>>,

pub fn format<'a>(&self, fmt: &'a str) -> DelayedFormat<StrftimeItems<'a>>

pub fn format_localized_with_items<'a, I, B>(
    &self,
    items: I,
    locale: Locale
) -> DelayedFormat<I>
    where
    I: Iterator<Item = B> + Clone,
    B: Borrow<Item<'a>>,

pub fn format_localized<'a>(
    &self,
    fmt: &'a str,
    locale: Locale
) -> DelayedFormat<StrftimeItems<'a>>


// DateTIme 实现了如下 Add/AddAssign/Sub/SubAssgn trait
impl<Tz: TimeZone> Add<Days> for DateTime<Tz>
impl<Tz: TimeZone> Add<Duration> for DateTime<Tz>
impl<Tz: TimeZone> Add<FixedOffset> for DateTime<Tz>
impl<Tz: TimeZone> Add<Months> for DateTime<Tz>
impl<Tz: TimeZone> Add<TimeDelta> for DateTime<Tz>
impl<Tz: TimeZone> AddAssign<Duration> for DateTime<Tz> // 标准度 std::time::Duration
impl<Tz: TimeZone> AddAssign<TimeDelta> for DateTime<Tz>
```


## <span class="section-num">2</span> TimeZone {#timezone}

TimeZone 是 DateTime 的主构造器类型：

-   TimeZone::from_timestampXX() 返回 TimeZone&lt;Utc&gt;
-   TimeZone::parse_fromXX() 返回 TimeZone&lt;FixedOffset&gt;

TimeZone trait 定义：

```rust
pub trait TimeZone: Sized + Clone {

    type Offset: Offset;

    // Required methods
    fn from_offset(offset: &Self::Offset) -> Self;
    fn offset_from_local_date( &self, local: &NaiveDate ) -> MappedLocalTime<Self::Offset>;
    fn offset_from_local_datetime( &self, local: &NaiveDateTime ) -> MappedLocalTime<Self::Offset>;
    fn offset_from_utc_date(&self, utc: &NaiveDate) -> Self::Offset;
    fn offset_from_utc_datetime(&self, utc: &NaiveDateTime) -> Self::Offset;

    // Provided methods
    fn with_ymd_and_hms(&self, year: i32, month: u32, day: u32, hour: u32, min: u32, sec: u32 ) ->MappedLocalTime<DateTime<Self>> { ... }

    // 创建 Date 对象
    fn ymd(&self, year: i32, month: u32, day: u32) -> Date<Self> { ... }
    fn ymd_opt( &self, year: i32, month: u32, day: u32 ) -> MappedLocalTime<Date<Self>> { ... }
    fn yo(&self, year: i32, ordinal: u32) -> Date<Self> { ... }
    fn yo_opt(&self, year: i32, ordinal: u32) -> MappedLocalTime<Date<Self>> { ... }
    fn isoywd(&self, year: i32, week: u32, weekday: Weekday) -> Date<Self> { ... }
    fn isoywd_opt( &self, year: i32, week: u32, weekday: Weekday ) -> MappedLocalTime<Date<Self>> { ... }
    fn from_local_date(&self, local: &NaiveDate) -> MappedLocalTime<Date<Self>> { ... }
    fn from_utc_date(&self, utc: &NaiveDate) -> Date<Self> { ... }

    // 创建 DateTime 对象
    fn timestamp(&self, secs: i64, nsecs: u32) -> DateTime<Self> { ... }
    fn timestamp_opt( &self, secs: i64, nsecs: u32 ) -> MappedLocalTime<DateTime<Self>> { ... }
    fn timestamp_millis(&self, millis: i64) -> DateTime<Self> { ... }
    fn timestamp_millis_opt( &self, millis: i64 ) -> MappedLocalTime<DateTime<Self>> { ... }
    fn timestamp_nanos(&self, nanos: i64) -> DateTime<Self> { ... }
    fn timestamp_micros(&self, micros: i64) -> MappedLocalTime<DateTime<Self>> { ... }

    // 从字符串解析创建 DateTIme 对象
    fn datetime_from_str( &self, s: &str, fmt: &str ) -> ParseResult<DateTime<Self>> { ... }

    fn from_local_datetime( &self, local: &NaiveDateTime ) -> MappedLocalTime<DateTime<Self>> { ... }
    fn from_utc_datetime(&self, utc: &NaiveDateTime) -> DateTime<Self> { ... }
}
```

如下类型实现了 TimeZone 的类型：

1.  FixedOffset
2.  Local
3.  Utc

其中 Local 和 Ttc 类型都是 `无成员的 unit struct 类型` ：struct Local 和 struct Utc，所以唯一的实例是它们自身，可以直接调用它们的 self 方法：

-   一般情况下，struct 都是有成员的，在调用它们的方法前，需要先实例化一个对象。但是 unit struct 无任何成员，唯一的实例就是类型名。
-   FixedOffset ubshi unit struct 类型，它是有成员的，需要先创建一个实例（使用 east_opt/west_opt() 方法）后再调用相关方法。

FixedOffset：

-   实现了 TimeZone trait，可以调用相关 with_XX 方法来创建 DateTime&lt;FixedOffset&gt;；
-   FixedOffset 对象可以作为 DateTime.with_timezone() 的参数，实现将 DateTime 转换为任意 timezone 的
    DateTime 对象。

<!--listend-->

```rust
pub struct FixedOffset { /* private fields */ }

impl FixedOffset
pub const fn east_opt(secs: i32) -> Option<FixedOffset>
pub const fn west_opt(secs: i32) -> Option<FixedOffset>
pub const fn local_minus_utc(&self) -> i32
pub const fn utc_minus_local(&self) -> i32

// 典型使用方法：
// 1. 使用 east_opt()/west_opt() 创建 FixedOffset 实例，它实现了 TimeZone trait；
// 2. 该实例实现了 TimeZone trait，调他的 with_ymd_and_hms()/ymd()/timestampXX() 等方法来创建 DateTime<FixedOffset>
use chrono::{FixedOffset, TimeZone};
let hour = 3600;
let datetime = FixedOffset::east_opt(5 * hour).unwrap().with_ymd_and_hms(2016, 11, 08, 0, 0, 0).unwrap();
assert_eq!(&datetime.to_rfc3339(), "2016-11-08T00:00:00+05:00")

// Current time in some timezone (let's use +05:00)
let offset = FixedOffset::east_opt(5 * 60 * 60).unwrap();
let now_with_offset = Utc::now().with_timezone(&offset);
```

Utc：

-   Utc 是 unit struct Utc 类型，Utc 是唯一实例，故可以直接调用 TimeZone trait 的方法

<!--listend-->

```rust
// 无成员 unit struct，唯一的实例是 Utc 自身
pub struct Utc;

// 示例
use chrono::{DateTime, TimeZone, Utc};
let dt = DateTime::from_timestamp(61, 0).unwrap();
// Utc 是唯一实例，故可以直接调用 TimeZone trait 的方法
assert_eq!(Utc.timestamp_opt(61, 0).unwrap(), dt);
assert_eq!(Utc.with_ymd_and_hms(1970, 1, 1, 0, 1, 1).unwrap(), dt);


// Current time in UTC
let now_utc = Utc::now();
// Current date in UTC
let today_utc = now_utc.date_naive();
// Current time in some timezone (let's use +05:00)
let offset = FixedOffset::east_opt(5 * 60 * 60).unwrap();
let now_with_offset = Utc::now().with_timezone(&offset);
```

Local：

-   Local 和 Utc 类似，也是 unit struct 类型；

<!--listend-->

```rust
pub struct Local;

use chrono::{DateTime, Local, TimeZone};
let dt1: DateTime<Local> = Local::now();
let dt2: DateTime<Local> = Local.timestamp_opt(0, 0).unwrap();
assert!(dt1 >= dt2);
```


## <span class="section-num">3</span> TimeDelta {#timedelta}

TimeDelta 是纳秒精度的时间间隔，内部记录 seconds 和 nanoseconds，别名类型为 Duration；

-   和 std::time::Duration 相比，TimeDelta 是有符号的，可以是负值。

<!--listend-->

```rust
pub struct TimeDelta { /* private fields */ }

// TimeDelta 方法：
pub const fn new(secs: i64, nanos: u32) -> Option<TimeDelta>

pub const fn weeks(weeks: i64) -> TimeDelta
pub const fn try_weeks(weeks: i64) -> Option<TimeDelta>

pub const fn days(days: i64) -> TimeDelta
pub const fn try_days(days: i64) -> Option<TimeDelta>

pub const fn hours(hours: i64) -> TimeDelta
pub const fn try_hours(hours: i64) -> Option<TimeDelta>

pub const fn minutes(minutes: i64) -> TimeDelta
pub const fn try_minutes(minutes: i64) -> Option<TimeDelta>

pub const fn seconds(seconds: i64) -> TimeDelta
pub const fn try_seconds(seconds: i64) -> Option<TimeDelta>

pub const fn milliseconds(milliseconds: i64) -> TimeDelta
pub const fn try_milliseconds(milliseconds: i64) -> Option<TimeDelta>
pub const fn microseconds(microseconds: i64) -> TimeDelta
pub const fn nanoseconds(nanos: i64) -> TimeDelta


pub const fn num_weeks(&self) -> i64
pub const fn num_days(&self) -> i64
pub const fn num_hours(&self) -> i64
pub const fn num_minutes(&self) -> i64
pub const fn num_seconds(&self) -> i64
pub const fn subsec_nanos(&self) -> i32
pub const fn num_milliseconds(&self) -> i64
pub const fn num_microseconds(&self) -> Option<i64>
pub const fn num_nanoseconds(&self) -> Option<i64>

pub const fn checked_add(&self, rhs: &TimeDelta) -> Option<TimeDelta>
pub const fn checked_sub(&self, rhs: &TimeDelta) -> Option<TimeDelta>
pub const fn checked_mul(&self, rhs: i32) -> Option<TimeDelta>
pub const fn checked_div(&self, rhs: i32) -> Option<TimeDelta>
pub const fn abs(&self) -> TimeDelta
pub const fn min_value() -> TimeDelta
pub const fn max_value() -> TimeDelta
pub const fn zero() -> TimeDelta
pub const fn is_zero(&self) -> bool


pub const fn from_std(duration: Duration) -> Result<TimeDelta, OutOfRangeError>
pub const fn to_std(&self) -> Result<Duration, OutOfRangeError>
```

大多数 chrono 类型实现了 std::ops::Add trait, 可以和 TimeDelta 相加：

```rust
impl<Tz: TimeZone> Add<TimeDelta> for Date<Tz>
impl<Tz: TimeZone> Add<TimeDelta> for DateTime<Tz>
impl Add<TimeDelta> for NaiveDate
impl Add<TimeDelta> for NaiveDateTime
impl Add<TimeDelta> for NaiveTime
impl Add for TimeDelta

impl<Tz: TimeZone> AddAssign<TimeDelta> for Date<Tz>
impl<Tz: TimeZone> AddAssign<TimeDelta> for DateTime<Tz>
impl AddAssign<TimeDelta> for NaiveDate
impl AddAssign<TimeDelta> for NaiveDateTime
impl AddAssign<TimeDelta> for NaiveTime
impl AddAssign for TimeDelta
```


## <span class="section-num">4</span> format module {#format-module}

-   DateTime::format(): 使用传入的格式化字符串来格式化输出字符串；
-   DateTime::parse_from_str(): 使用格式化字符串来解析字符串，生成 DateTime；

<!--listend-->

```rust
use chrono::{NaiveDateTime, TimeZone, Utc};

let date_time = Utc.with_ymd_and_hms(2020, 11, 10, 0, 1, 32).unwrap();

let formatted = format!("{}", date_time.format("%Y-%m-%d %H:%M:%S"));
assert_eq!(formatted, "2020-11-10 00:01:32");

let parsed = NaiveDateTime::parse_from_str(&formatted, "%Y-%m-%d %H:%M:%S")?.and_utc();
assert_eq!(parsed, date_time);
```


## <span class="section-num">5</span> chrono 集成 {#chrono-集成}

diesel 等 ORM 引擎提供了 chrono 时间集成的能力：
<https://docs.rs/diesel/latest/diesel/sql_types/struct.Datetime.html>

chrono::serde module 提供了 serde crate 集成的能力（基于 serde with attribute）：

-   \#[serde(with = "module")] ： Combination of serialize_with and deserialize_with. Serde will use
    `$module::serialize` as the serialize_with function and `$module::deserialize` as the deserialize_with
    function.

<!--listend-->

```rust
use chrono::serde::ts_microseconds;
#[derive(Deserialize, Serialize)]
struct S {
    #[serde(with = "ts_microseconds")]
    time: DateTime<Utc>
}

let time = NaiveDate::from_ymd_opt(2018, 5, 17).unwrap().and_hms_micro_opt(02, 04, 59, 918355).unwrap().and_local_timezone(Utc).unwrap();
let my_s = S {
    time: time.clone(),
};

let as_string = serde_json::to_string(&my_s)?;
assert_eq!(as_string, r#"{"time":1526522699918355}"#);
let my_s: S = serde_json::from_str(&as_string)?;
assert_eq!(my_s.time, time);

use chrono::serde::ts_microseconds_option;
#[derive(Deserialize, Serialize)]
struct S {
    #[serde(with = "ts_microseconds_option")]
    time: Option<DateTime<Utc>>
}

let time = Some(NaiveDate::from_ymd_opt(2018, 5, 17).unwrap().and_hms_micro_opt(02, 04, 59, 918355).unwrap().and_local_timezone(Utc).unwrap());
let my_s = S {
    time: time.clone(),
};

let as_string = serde_json::to_string(&my_s)?;
assert_eq!(as_string, r#"{"time":1526522699918355}"#);
let my_s: S = serde_json::from_str(&as_string)?;
assert_eq!(my_s.time, time);
```
