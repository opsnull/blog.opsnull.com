---
title: "http/http_body crate"
author: ["zhangjun"]
lastmod: 2024-07-07T22:04:18+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 16
---

http/http_body crate 是公共的 http 和 body 定义，在 tokio 系列的 HTTP 库，如 hyper/axum/reqwest 中得到广泛应用，这些 crate 都是通过 import + pub use 的方式将 http/http_body 的类型导入到自己 crate 中来使用。比如 reqwest::method 实际是 <http::method> , 但是也有一些是封装，例如
reqwest::Request/reqwest::Response 是 <http::Request> 和 <http::Response> 的封装。


## <span class="section-num">1</span> request {#request}

<http::request> module 包含三个类型：

1.  struct Request
2.  struct Parts
3.  struct Builder

<http::request::Request%3CT%3E> 和 <http::response::Response%3CT%3E> 都是泛型类型，T 对应 body 的数据类型：

```rust
pub struct Request<T> { /* private fields */ }

// Request::builder() 创建的是 Request<()> 类型对象，所以 request.body(()) 传入的是 ();
use http::{Request, Response};
let mut request = Request::builder()
    .uri("https://www.rust-lang.org/")
    .header("User-Agent", "my-awesome-agent/1.0");
if needs_awesome_header() {
    request = request.header("Awesome", "yes");
}
let response = send(request.body(()).unwrap());
fn send(req: Request<()>) -> Response<()> {
    // ...
}
```

impl Request&lt;()&gt; 的方法（未配置 body，后续可以通过 Builder 的 body() 方法来设置 body 内容) 实现了
builder() 和 get/put/post() 等方法，返回一个 <http::request::Builder> 对象：

```rust
impl Request<()>
pub fn builder() -> Builder

pub fn get<T>(uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
pub fn put<T>(uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
pub fn post<T>(uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
pub fn delete<T>(uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
pub fn options<T>(uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
pub fn head<T>(uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
pub fn connect<T>(uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
pub fn patch<T>(uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
pub fn trace<T>(uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
```

<http::request::Builder>:

-   通过 Builder 的各方法可以设置 method/uri/version/header/extension 和 body。
-   body&lt;T&gt;(self, body: T) 消耗 Builder，使用传入的 body 内容，返回一个 Reuest。

<!--listend-->

```rust
pub struct Builder { /* private fields */ }

impl Builder
pub fn new() -> Builder
pub fn method<T>(self, method: T) -> Builder where Method: TryFrom<T>, <Method as TryFrom<T>>::Error: Into<Error>,
pub fn method_ref(&self) -> Option<&Method>
pub fn uri<T>(self, uri: T) -> Builder where Uri: TryFrom<T>, <Uri as TryFrom<T>>::Error: Into<Error>,
pub fn uri_ref(&self) -> Option<&Uri>

pub fn version(self, version: Version) -> Builder
pub fn version_ref(&self) -> Option<&Version>

pub fn header<K, V>(self, key: K, value: V) -> Builder where
    HeaderName: TryFrom<K>,
    <HeaderName as TryFrom<K>>::Error: Into<Error>,
    HeaderValue: TryFrom<V>,
    <HeaderValue as TryFrom<V>>::Error: Into<Error>,
pub fn headers_ref(&self) -> Option<&HeaderMap<HeaderValue>>
pub fn headers_mut(&mut self) -> Option<&mut HeaderMap<HeaderValue>>
pub fn extension<T>(self, extension: T) -> Builder where T: Clone + Any + Send + Sync + 'static,
pub fn extensions_ref(&self) -> Option<&Extensions>
pub fn extensions_mut(&mut self) -> Option<&mut Extensions>

// body() 消耗 Builder，使用传入的 body 内容，返回一个 Request
pub fn body<T>(self, body: T) -> Result<Request<T>>
```

也可以不使用 Builder 模式，而是使用 `impl<T> Request<T>` 提供的方法来直接设置 Request：

-   http body 类型为 T（没有限界）, 在创建 Request 时需要传入 body；
-   一个 Request 有 Parts + body 组成

<!--listend-->

```rust
impl<T> Request<T>

pub fn new(body: T) -> Request<T>

pub fn from_parts(parts: Parts, body: T) -> Request<T>

pub fn method(&self) -> &Method
pub fn method_mut(&mut self) -> &mut Method
pub fn uri(&self) -> &Uri
pub fn uri_mut(&mut self) -> &mut Uri
pub fn version(&self) -> Version
pub fn version_mut(&mut self) -> &mut Version
pub fn headers(&self) -> &HeaderMap<HeaderValue>
pub fn headers_mut(&mut self) -> &mut HeaderMap<HeaderValue>
pub fn extensions(&self) -> &Extensions
pub fn extensions_mut(&mut self) -> &mut Extensions

pub fn body(&self) -> &T
pub fn body_mut(&mut self) -> &mut T
pub fn into_body(self) -> T

// 返回 http reqeust Parts 和 body 内容
pub fn into_parts(self) -> (Parts, T)

pub fn map<F, U>(self, f: F) -> Request<U>where F: FnOnce(T) -> U,
```

<http::request::Parts> 包含了 HTTP Request 除 body 外的信息(类似的还有 <http::response::Parts> 表示 HTTP
Response 除 body 外的内容)：

```rust
pub struct Parts {
    pub method: Method,
    pub uri: Uri,
    pub version: Version,
    pub headers: HeaderMap<HeaderValue>,
    pub extensions: Extensions,
    /* private fields */
}
```


## <span class="section-num">2</span> response {#response}

<http::response> module 包含：

1.  Response struct
2.  Parts struct
3.  Builder struct

Struct <http::response::Response%EF%BC%9A>

```rust
pub struct Response<T> { /* private fields */ }

use http::{Request, Response, StatusCode};

fn respond_to(req: Request<()>) -> http::Result<Response<()>> {
    let mut builder = Response::builder()
        .header("Foo", "Bar")
        .status(StatusCode::OK);

    if req.headers().contains_key("Another-Header") {
        builder = builder.header("Another-Header", "Ack");
    }

    builder.body(())
}

fn not_found(_req: Request<()>) -> http::Result<Response<()>> {
    Response::builder()
        .status(StatusCode::NOT_FOUND)
        .body(())
}

// 解序列化
fn deserialize<T>(res: Response<Vec<u8>>) -> serde_json::Result<Response<T>>
    where for<'de> T: de::Deserialize<'de>,
{
    let (parts, body) = res.into_parts();
    let body = serde_json::from_slice(&body)?;
    Ok(Response::from_parts(parts, body))
}
```

<http::response::Response> 的方法：

```rust
// 创建一个 Response Builder
impl Response<()>
pub fn builder() -> Builder


// 直接创建 Response，需要在 new() 创建时传入 body
impl<T> Response<T>
pub fn new(body: T) -> Response<T>
pub fn from_parts(parts: Parts, body: T) -> Response<T>

pub fn status(&self) -> StatusCode
pub fn status_mut(&mut self) -> &mut StatusCode
pub fn version(&self) -> Version
pub fn version_mut(&mut self) -> &mut Version
pub fn headers(&self) -> &HeaderMap<HeaderValue>
pub fn headers_mut(&mut self) -> &mut HeaderMap<HeaderValue>
pub fn extensions(&self) -> &Extensions
pub fn extensions_mut(&mut self) -> &mut Extensions

pub fn body(&self) -> &T
pub fn body_mut(&mut self) -> &mut T
pub fn into_body(self) -> T

pub fn into_parts(self) -> (Parts, T)
pub fn map<F, U>(self, f: F) -> Response<U> where F: FnOnce(T) -> U,
```

<http::response::Builder> 对象：

-   body() 方法消耗 Builder，使用传入的 body，返回一个 Response

<!--listend-->

```rust
impl Builder

pub fn new() -> Builder
pub fn status<T>(self, status: T) -> Builderwhere
    StatusCode: TryFrom<T>,
    <StatusCode as TryFrom<T>>::Error: Into<Error>,

pub fn version(self, version: Version) -> Builder

pub fn header<K, V>(self, key: K, value: V) -> Builderwhere
    HeaderName: TryFrom<K>,
    <HeaderName as TryFrom<K>>::Error: Into<Error>,
    HeaderValue: TryFrom<V>,
    <HeaderValue as TryFrom<V>>::Error: Into<Error>,

pub fn headers_ref(&self) -> Option<&HeaderMap<HeaderValue>>
pub fn headers_mut(&mut self) -> Option<&mut HeaderMap<HeaderValue>>

pub fn extension<T>(self, extension: T) -> Builderwhere
    T: Clone + Any + Send + Sync + 'static,
pub fn extensions_ref(&self) -> Option<&Extensions>
pub fn extensions_mut(&mut self) -> Option<&mut Extensions>

// 消耗 Builder，使用传入的 body，返回一个 Response
pub fn body<T>(self, body: T) -> Result<Response<T>>
```

<http::response::Parts> 表示除了 body 外的 HTTP 相应内容：

```rust
pub struct Parts {
    pub status: StatusCode,
    pub version: Version,
    pub headers: HeaderMap<HeaderValue>,
    pub extensions: Extensions,
    /* private fields */
}
```


## <span class="section-num">3</span> header {#header}

<http::header> module 提供了三个重要的类型：

HeaderMap
: HTTP headers 集合

HeaderName
: HTTP header field name

HeaderValue
: HTTP header field value

HeaderName 提供了大小写不敏感的 header field name 能力，内部会进行小写归一化，以加快比较和处理。

HeaderMap 是高度优化的 HTTP Header 集合，内部使用 multimap 解构，运行一个 header 有多个 value 值，它的 APIs 接口和 HashMap 类似。

HeaderMap:

-   T 代表 header value 值类型，默认为 HeaderValue;
-   将 HeaderName 关联到 HeaderValue
-   一个 key 可以通过 append() 方法添加多个 value，后续通过 get_all() 获得该 key 的所有 value

<!--listend-->

```rust
pub struct HeaderMap<T = HeaderValue> { /* private fields */ }

impl HeaderMap
pub fn new() -> HeaderMap

impl<T> HeaderMap<T>
pub fn with_capacity(capacity: usize) -> HeaderMap<T>
pub fn try_with_capacity( capacity: usize ) -> Result<HeaderMap<T>, MaxSizeReached>

pub fn len(&self) -> usize
pub fn keys_len(&self) -> usize
pub fn is_empty(&self) -> bool
pub fn clear(&mut self)

pub fn capacity(&self) -> usize
pub fn reserve(&mut self, additional: usize)
pub fn try_reserve(&mut self, additional: usize) -> Result<(), MaxSizeReached>

// 获得 header 对应的 value，这里的 key 可以是任何实现 AsHeaderName 的类型
pub fn get<K>(&self, key: K) -> Option<&T> where K: AsHeaderName,
pub fn get_mut<K>(&mut self, key: K) -> Option<&mut T> where K: AsHeaderName,
pub fn get_all<K>(&self, key: K) -> GetAll<'_, T> where K: AsHeaderName,
pub fn contains_key<K>(&self, key: K) -> bool where K: AsHeaderName,

pub fn iter(&self) -> Iter<'_, T> ⓘ
pub fn iter_mut(&mut self) -> IterMut<'_, T> ⓘ

pub fn keys(&self) -> Keys<'_, T> ⓘ
pub fn values(&self) -> Values<'_, T> ⓘ
pub fn values_mut(&mut self) -> ValuesMut<'_, T> ⓘ

pub fn drain(&mut self) -> Drain<'_, T> ⓘ

pub fn entry<K>(&mut self, key: K) -> Entry<'_, T> where K: IntoHeaderName,
pub fn try_entry<K>( &mut self, key: K) -> Result<Entry<'_, T>, InvalidHeaderName> where K: AsHeaderName,

// 添加 key-value，一个 key 可以通过 append() 添加多个 value
pub fn insert<K>(&mut self, key: K, val: T) -> Option<T> where K: IntoHeaderName,
pub fn try_insert<K>( &mut self, key: K, val: T) -> Result<Option<T>, MaxSizeReached> where K: IntoHeaderName,
pub fn append<K>(&mut self, key: K, value: T) -> bool where K: IntoHeaderName,
pub fn try_append<K>( &mut self, key: K, value: T) -> Result<bool, MaxSizeReached> where K: IntoHeaderName,

pub fn remove<K>(&mut self, key: K) -> Option<T> where K: AsHeaderName,

// HeaderMap 实现了 Index trait，可以使用 headers[key] 操作
impl<K, T> Index<K> for HeaderMap<T> where K: AsHeaderName,
  fn index(&self, index: K) -> &T
  type Output = T
```

HeaderMap 实现了几个 Extend/FromIterators/TryFrom trait，可以快速创建 HeaderMap：

```rust
impl<T> Extend<(HeaderName, T)> for HeaderMap<T>
fn extend<I: IntoIterator<Item = (HeaderName, T)>>(&mut self, iter: I)

impl<T> Extend<(Option<HeaderName>, T)> for HeaderMap<T>
fn extend<I: IntoIterator<Item = (Option<HeaderName>, T)>>(&mut self, iter: I)

impl<T> FromIterator<(HeaderName, T)> for HeaderMap<T>
fn from_iter<I>(iter: I) -> Self where I: IntoIterator<Item = (HeaderName, T)>,

impl<'a, K, V, T> TryFrom<&'a HashMap<K, V>> for HeaderMap<T>where
    K: Eq + Hash,
    HeaderName: TryFrom<&'a K>,
    <HeaderName as TryFrom<&'a K>>::Error: Into<Error>,
    T: TryFrom<&'a V>,
    T::Error: Into<Error>,
```

示例：

```rust
let mut headers = HeaderMap::new();
headers.insert(HOST, "example.com".parse().unwrap()); // parse() 将 &str 转换为 HeaderValue
headers.insert(CONTENT_LENGTH, "123".parse().unwrap());
assert!(headers.contains_key(HOST));
assert!(!headers.contains_key(LOCATION));
assert_eq!(headers[HOST], "example.com");
headers.remove(HOST);
assert!(!headers.contains_key(HOST));

let mut map = HeaderMap::new();
assert!(map.get("host").is_none());
map.insert(HOST, "hello".parse().unwrap());
assert_eq!(map.get(HOST).unwrap(), &"hello");
assert_eq!(map.get("host").unwrap(), &"hello");
map.append(HOST, "world".parse().unwrap());
assert_eq!(map.get("host").unwrap(), &"hello");


// 一个 key 可以有多个 value
let mut map = HeaderMap::new();
map.insert(HOST, "hello".parse().unwrap());
map.append(HOST, "goodbye".parse().unwrap());
let view = map.get_all("host");
let mut iter = view.iter();
assert_eq!(&"hello", iter.next().unwrap());
assert_eq!(&"goodbye", iter.next().unwrap());
assert!(iter.next().is_none());
```

HeaderName：代表一个 HTTP header name，在 HeaderMap 的相关方法中使用：

-   <http::header> module 中提供了标准的 HeaderName, 如 CONTENT_ENCODING 等；
-   from_XX() 方法将传入的 key name 转换为 `HTTP 标准 key name` ；

<!--listend-->

```rust
pub struct HeaderName { /* private fields */ }

impl HeaderName
// 创建 HeaderName
pub fn from_bytes(src: &[u8]) -> Result<HeaderName, InvalidHeaderName>
pub fn from_lowercase(src: &[u8]) -> Result<HeaderName, InvalidHeaderName>
pub const fn from_static(src: &'static str) -> HeaderName
pub fn as_str(&self) -> &str

// Parsing a lower case header
let hdr = HeaderName::from_lowercase(b"content-length").unwrap();
assert_eq!(CONTENT_LENGTH, hdr);
// Parsing a header that contains uppercase characters
assert!(HeaderName::from_lowercase(b"Content-Length").is_err());

// Parsing a standard header
let hdr = HeaderName::from_static("content-length");
assert_eq!(CONTENT_LENGTH, hdr);

// Parsing a custom header
let CUSTOM_HEADER: &'static str = "custom-header";

let a = HeaderName::from_lowercase(b"custom-header").unwrap();
let b = HeaderName::from_static(CUSTOM_HEADER);
assert_eq!(a, b);
```

HeaderValue：

```rust
let val = HeaderValue::from_static("hello");
assert_eq!(val, "hello");

let val = HeaderValue::from_str("hello").unwrap();
assert_eq!(val, "hello");

let val = HeaderValue::from_bytes(b"hello\xfa").unwrap();
assert_eq!(val, &b"hello\xfa"[..]);
```

<http::header> 提供了大量标准的 HeaderName 常量定义：

```rust
pub const ACCEPT: HeaderName;
pub const RANGE: HeaderName;
// ...
```


## <span class="section-num">4</span> method {#method}

<http::method> module 提供了 http Method struct 定义，它包含一些标准的 http method 常量定义：

```rust
pub struct Method(/* private fields */);

// Method 关联的常量
impl Method
pub const GET: Method = _
pub const POST: Method = _
pub const PUT: Method = _
pub const DELETE: Method = _
pub const HEAD: Method = _
pub const OPTIONS: Method = _
pub const CONNECT: Method = _
pub const PATCH: Method = _
pub const TRACE: Method = _

// 创建 Method
pub fn from_bytes(src: &[u8]) -> Result<Method, InvalidMethod>

pub fn is_safe(&self) -> bool
pub fn is_idempotent(&self) -> bool
pub fn as_str(&self) -> &str
```

示例：

```rust
use http::Method;
assert_eq!(Method::GET, Method::from_bytes(b"GET").unwrap());
assert!(Method::GET.is_idempotent());
assert_eq!(Method::POST.as_str(), "POST");
```


## <span class="section-num">5</span> status {#status}

<http::status> module 提供了 StatusCode struct 类型和标准 HTTP status 常量：

```rust
pub struct StatusCode(/* private fields */);

impl StatusCode
// 创建 StatusCode
pub fn from_u16(src: u16) -> Result<StatusCode, InvalidStatusCode>
pub fn from_bytes(src: &[u8]) -> Result<StatusCode, InvalidStatusCode>

pub fn as_u16(&self) -> u16
pub fn as_str(&self) -> &str
pub fn canonical_reason(&self) -> Option<&'static str>
pub fn is_informational(&self) -> bool
pub fn is_success(&self) -> bool
pub fn is_redirection(&self) -> bool
pub fn is_client_error(&self) -> bool
pub fn is_server_error(&self) -> bool
```

示例：

```rust
use http::StatusCode;

assert_eq!(StatusCode::from_u16(200).unwrap(), StatusCode::OK);
assert_eq!(StatusCode::NOT_FOUND.as_u16(), 404);
assert!(StatusCode::OK.is_success());
```

标准 HTTP StatusCode：

```rust
impl StatusCode
pub const CONTINUE: StatusCode = _
pub const SWITCHING_PROTOCOLS: StatusCode = _
pub const PROCESSING: StatusCode = _
pub const OK: StatusCode = _
//..
```


## <span class="section-num">6</span> extensions {#extensions}

struct <http::Extensions> 是在 Request 和 Response 中使用的, 用来保存额外的数据:

-   具有 get/insert/remove() 方法.
-   get/insert 都是按照 type 来获取或设置值的.

<!--listend-->

```rust
let mut ext = Extensions::new();
assert!(ext.insert(5i32).is_none());
assert!(ext.insert(4u8).is_none());
assert_eq!(ext.insert(9i32), Some(5i32));

let mut ext = Extensions::new();
assert!(ext.get::<i32>().is_none());
ext.insert(5i32);
assert_eq!(ext.get::<i32>(), Some(&5i32));

let mut ext = Extensions::new();
ext.insert(5i32);
assert_eq!(ext.remove::<i32>(), Some(5i32));
assert!(ext.get::<i32>().is_none());
```


## <span class="section-num">7</span> uri {#uri}

<http::uri> module 提供了 Uri/Parts/Port/Schema 等类型, Uri 并不等于 URL, Uri 可能只包含 Full URL 的部分内容.

```rust
use http::Uri;

let uri = "/foo/bar?baz".parse::<Uri>().unwrap();
assert_eq!(uri.path(), "/foo/bar");
assert_eq!(uri.query(), Some("baz"));
assert_eq!(uri.host(), None);

let uri = "https://www.rust-lang.org/install.html".parse::<Uri>().unwrap();
assert_eq!(uri.scheme_str(), Some("https"));
assert_eq!(uri.host(), Some("www.rust-lang.org"));
assert_eq!(uri.path(), "/install.html");
```


## <span class="section-num">8</span> http_body crate {#http-body-crate}

http_body crate 提供了 async HTTP Body trait 定义。

Body trait 代表 Request 或 Response 的 straming body。每次 poll 时返回一个 Frame&lt;Data&gt; 对象。

Frames 包含 Self::Data 类型的数据 buffer，而且也可以包含 HTTP/2.0 协议使用的 trailers（header）。

```rust
pub trait Body {
    type Data: Buf; // bytes::Buf trait 类型，只读，内部包含读指针。
    type Error;

    // Required method
    fn poll_frame(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>
    ) -> Poll<Option<Result<Frame<Self::Data>, Self::Error>>>; // 每次 poll 返回一个 Frame<Self.Data> 数据

    // Provided methods
    fn is_end_stream(&self) -> bool { ... }
    fn size_hint(&self) -> SizeHint { ... }
}
```

<http::Request>, <http::Response>, String 类型均实现了 http_body::Body trait:

-   Request&lt;T&gt; 和 Response&lt;T&gt; 的 body() 传入的是 T 类型的 body 内容。

<!--listend-->

```rust
impl Body for String
type Data = Bytes // 返回 bytes::Bytes struct 类型
type Error = Infallible
fn poll_frame(
    self: Pin<&mut Self>,
    _cx: &mut Context<'_>
) -> Poll<Option<Result<Frame<Self::Data>, Self::Error>>>


// B: Body 也是 http_body::Body trait
impl<B: Body> Body for Request<B>
type Data = <B as Body>::Data
type Error = <B as Body>::Error
fn poll_frame(
    self: Pin<&mut Self>,
    cx: &mut Context<'_>
) -> Poll<Option<Result<Frame<Self::Data>, Self::Error>>>


impl<B: Body> Body for Response<B>
type Data = <B as Body>::Data
type Error = <B as Body>::Error
fn poll_frame(
    self: Pin<&mut Self>,
    cx: &mut Context<'_>
) -> Poll<Option<Result<Frame<Self::Data>, Self::Error>>>
```

示例：

```rust
#![deny(warnings)]
#![warn(rust_2018_idioms)]
use std::env;
use bytes::Bytes;
use http_body_util::{BodyExt, Empty};
use hyper::Request;
use tokio::io::{self, AsyncWriteExt as _};
use tokio::net::TcpStream;

#[path = "../benches/support/mod.rs"]
mod support;
use support::TokioIo;

// A simple type alias so as to DRY.
type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;

#[tokio::main]
async fn main() -> Result<()> {
    pretty_env_logger::init();

    // Some simple CLI args requirements...
    let url = match env::args().nth(1) {
        Some(url) => url,
        None => {
            println!("Usage: client <url>");
            return Ok(());
        }
    };

    // HTTPS requires picking a TLS implementation, so give a better
    // warning if the user tries to request an 'https' URL.
    let url = url.parse::<hyper::Uri>().unwrap();
    if url.scheme_str() != Some("http") {
        println!("This example only works with 'http' URLs.");
        return Ok(());
    }

    fetch_url(url).await
}

async fn fetch_url(url: hyper::Uri) -> Result<()> {
    let host = url.host().expect("uri has no host");
    let port = url.port_u16().unwrap_or(80);
    let addr = format!("{}:{}", host, port);
    let stream = TcpStream::connect(addr).await?;
    let io = TokioIo::new(stream);

    let (mut sender, conn) = hyper::client::conn::http1::handshake(io).await?;
    tokio::task::spawn(async move {
        if let Err(err) = conn.await {
            println!("Connection failed: {:?}", err);
        }
    });

    let authority = url.authority().unwrap().clone();

    let path = url.path();
    let req = Request::builder()
        .uri(path)
        .header(hyper::header::HOST, authority.as_str())
        .body(Empty::<Bytes>::new())?;

    let mut res = sender.send_request(req).await?;

    println!("Response: {}", res.status());
    println!("Headers: {:#?}\n", res.headers());

    // Stream the body, writing each chunk to stdout as we get it (instead of buffering and printing
    // at the end).
    while let Some(next) = res.frame().await {
        let frame = next?;
        if let Some(chunk) = frame.data_ref() { // 获取 frame 中数据
            io::stdout().write_all(&chunk).await?;
        }
    }

    println!("\n\nDone!");

    Ok(())
}
```
