---
title: "hyper"
author: ["zhangjun"]
lastmod: 2024-07-29T10:56:07+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 8
---

hyper is a fast and correct HTTP implementation written in and for Rust. Features

-   HTTP/1 and HTTP/2
-   Asynchronous design
-   Leading in performance
-   Tested and correct
-   Extensive production use
-   Client and Server APIs

“Low-level”: hyper is `a lower-level HTTP library`, meant to be a building block for libraries and
applications. If looking for just a convenient HTTP client, consider the `reqwest` crate.

[hyper](https://docs.rs/hyper/latest/hyper/) 提供了 HTTP1/HTTP2 的 client 和 server 实现，属于较底层的 lib。

-   对于 client 建议使用 `reqwest` ；
-   对于 server 建议使用 `axum` 这两个高层库。

Hyper requires you to take care of lower level details, like parsing the URI (example from the
documentation):

```rust
let client = Client::new();
let uri = "http://httpbin.org/ip".parse()?;
let mut resp = client.get(uri).await?;

while let Some(chunk) = resp.body_mut().data().await {
    stdout().write_all(&chunk?).await?;
}
```

while `reqwest` handle this for you (from docs.rs) :

```rust
let body = reqwest::get("https://www.rust-lang.org").await?.text().await?;
```

hyper pub export 了 `http crate` 中的 header, Method, Request, Response, StatusCode, Uri, Version 等对象类型，所以这些类型不是 hyper 原生定义的。这些类型在 axum/tower 等 crate 中都得到了复用。

```rust
pub use crate::http::{header, Method, Request, Response, StatusCode, Uri, Version};

#[doc(no_inline)]
pub use crate::http::HeaderMap;

pub use crate::error::{Error, Result};
```

Request：

-   into_pargs()
-   extensions()
-   body()

<!--listend-->

```rust
pub fn into_parts(self) -> (Parts, T)
// Consumes the request returning the head and body parts.

let request = Request::new(());
let (parts, body) = request.into_parts();
assert_eq!(parts.method, Method::GET);

let request: Request<()> = Request::default();
assert!(request.extensions().get::<i32>().is_none());
```

Response:

```rust
use http::{Request, Response};

fn get(url: &str) -> http::Result<Response<()>> {
    // ...
}

let response = get("https://www.rust-lang.org/").unwrap();

if !response.status().is_success() {
    panic!("failed to get a successful response status!");
}

if let Some(date) = response.headers().get("Date") {
    // we've got a `Date` header!
}

let body = response.body();
// ...
```

Uri:

```rust
// abc://username:password@example.com:123/path/data?key=value&key2=value2#fragid1
// |-|   |-------------------------------||--------| |-------------------| |-----|
//  |                  |                       |               |              |
// scheme          authority                 path            query         fragment

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

Error:

```rust
pub struct Error { /* private fields */ }
```

Result:

```rust
pub type Result<T> = Result<T, Error>;
```

hyper::client module: <https://docs.rs/hyper/latest/hyper/client/index.html>

提供了 http1 和 http2 两个 client。

HTTP Client:hyper provides HTTP over a single connection. See the conn module.

hyper::client::conn module: The types in this module are to provide a lower-level API based around
`a single connection`. Connecting to a host, pooling connections, and the like are not handled at
this level. This module provides the building blocks to customize those things externally.

If don’t have need to manage connections yourself, consider using the higher-level Client API.

http1:

```rust
#![deny(warnings)]
#![warn(rust_2018_idioms)]

use bytes::Bytes;
use http_body_util::{BodyExt, Empty};
use hyper::{body::Buf, Request};
use serde::Deserialize;
use tokio::net::TcpStream;

#[path = "../benches/support/mod.rs"]
mod support;
use support::TokioIo;

// A simple type alias so as to DRY.
type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;

#[tokio::main]
async fn main() -> Result<()> {
    let url = "http://jsonplaceholder.typicode.com/users".parse().unwrap();
    let users = fetch_json(url).await?;
    // print users
    println!("users: {:#?}", users);

    // print the sum of ids
    let sum = users.iter().fold(0, |acc, user| acc + user.id);
    println!("sum of ids: {}", sum);
    Ok(())
}

async fn fetch_json(url: hyper::Uri) -> Result<Vec<User>> {
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

    // Fetch the url...
    let req = Request::builder()
        .uri(url)
        .header(hyper::header::HOST, authority.as_str())
        .body(Empty::<Bytes>::new())?;

    let res = sender.send_request(req).await?;

    // asynchronously aggregate the chunks of the body
    let body = res.collect().await?.aggregate();

    // try to parse as json with serde_json
    let users = serde_json::from_reader(body.reader())?;

    Ok(users)
}

#[derive(Deserialize, Debug)]
struct User {
    id: i32,
    #[allow(unused)]
    name: String,
}
```

hyper::client::conn::http1::Builder 可以用于配置 HTTP connection，最终提供了 handleshake() 方法来获得异步响应。配置的内容（部分）：

-   pub fn max_buf_size(&amp;mut self, max: usize) -&gt; &amp;mut Self
-   pub fn read_buf_exact_size(&amp;mut self, sz: Option&lt;usize&gt;) -&gt; &amp;mut Builder

hyper::client::conn::http2::Builder 可以配置的内容：

1.  pub fn keep_alive_interval(）
2.  pub fn keep_alive_timeout()
3.  pub fn max_frame_size()
4.  pub fn max_send_buf_size()
