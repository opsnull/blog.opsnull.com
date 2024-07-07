---
title: "reqwest"
author: ["zhangjun"]
lastmod: 2024-07-07T22:04:16+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 11
---

[reqwest](https://docs.rs/reqwest/latest/reqwest/) 是在 hyper 基础上实现的高层 HTTP client 库，支持异步和同步接口。reqwest 的 APIs 默认是 async
的，reqwest::Client 是异步的，但是也提供了 reqwest::blocking module, 它提供了各种同步版本的 APIs。（需要开启 blocking feature）。

```rust
let body = reqwest::get("https://www.rust-lang.org")
    .await?
    .text()
    .await?;
println!("body = {body:?}");

// 可以复用的 client
let client = reqwest::Client::new();
let res = client.post("http://httpbin.org/post")
    .body("the exact body that is sent")
    .send()
    .await?;

// This will POST a body of `foo=bar&baz=quux`，This can be an array of tuples, or a HashMap, or a
// custom type that implements Serialize.
let params = [("foo", "bar"), ("baz", "quux")];
let client = reqwest::Client::new();
let res = client.post("http://httpbin.org/post")
    .form(&params)
    .send()
    .await?;

// This will POST a body of `{"lang":"rust","body":"json"}`， It can take any value that can be
// serialized into JSON. The feature json is required.
let mut map = HashMap::new();
map.insert("lang", "rust");
map.insert("body", "json");
let client = reqwest::Client::new();
let res = client.post("http://httpbin.org/post")
    .json(&map)
    .send()
    .await?;
```

reqwest 默认支持重定向（最多 10 hops），可以通过 ClientBuilder 的 redirect::Policy 来控制。

reqwest 默认是支持 Proxy，支持 HTTP/HTTPS/SOCKS5 等代理，支持环境变量，如 HTTPS_PROXY or https_proxy
or no_proxy 等, 可以通过 ClientBuilder 的 Proxy 来控制。

当连接 https，默认开启 TLS，而且支持通过 ClientBuilder 来控制：

-   使用 Certificate 类型来为 server 添加证书；
-   使用 Identity 类型来为 Client 添加证书；

支持 gzip/bzip2/deflate/zstd 等压缩算法。

request::get() 发送一个 GET 请求，每次请求都创建临时的 Client，效率不高：

```rust
pub async fn get<T: IntoUrl>(url: T) -> Result<Response>

let body = reqwest::get("https://www.rust-lang.org").await?
    .text().await?;
```

reqwest 的 Request/Response/Body 都是自己定义的新 struct 类型, 但是内部封装使用的是 http/http_body
crate 中的类型.


## <span class="section-num">1</span> Url {#url}

Struct reqwest::Url 代表解析后的 Url：

```rust
pub struct Url { /* private fields */ }

impl Url

// 解析 URL 字符串（自动 URL 编码）
pub fn parse(input: &str) -> Result<Url, ParseError>
let url = Url::parse("https://example.net")?;

// 解析 URL 字符串 + 表单参数（自动 URL 编码）
pub fn parse_with_params<I, K, V>( input: &str, iter: I ) -> Result<Url, ParseError>
where
    I: IntoIterator,
    <I as IntoIterator>::Item: Borrow<(K, V)>,
    K: AsRef<str>,
    V: AsRef<str>,
let url = Url::parse_with_params("https://example.net?dont=clobberme", &[("lang", "rust"), ("browser", "servo")])?;
assert_eq!("https://example.net/?dont=clobberme&lang=rust&browser=servo", url.as_str());

// 添加一级 URL Path
pub fn join(&self, input: &str) -> Result<Url, ParseError>
let base = Url::parse("https://example.net/a/b.html")?;
let url = base.join("c.png")?;
assert_eq!(url.as_str(), "https://example.net/a/c.png");  // Not /a/b.html/c.png
let base = Url::parse("https://example.net/a/b/")?;
let url = base.join("c.png")?;
assert_eq!(url.as_str(), "https://example.net/a/b/c.png");

pub fn make_relative(&self, url: &Url) -> Option<String>
let base = Url::parse("https://example.net/a/b/")?;
let url = Url::parse("https://example.net/a/d/c.png")?;
let relative = base.make_relative(&url);
assert_eq!(relative.as_ref().map(|s| s.as_str()), Some("../d/c.png"));

pub fn options<'a>() -> ParseOptions<'a>
pub fn as_str(&self) -> &str
pub fn into_string(self) -> String

pub fn origin(&self) -> Origin

pub fn scheme(&self) -> &str
let url = Url::parse("file:///tmp/foo")?;
assert_eq!(url.scheme(), "file");

pub fn is_special(&self) -> bool
pub fn has_authority(&self) -> bool
let url = Url::parse("ftp://rms@example.com")?;
assert!(url.has_authority());

pub fn authority(&self) -> &str
let url = Url::parse("file:///tmp/foo")?;
assert_eq!(url.authority(), "");
let url = Url::parse("https://user:password@example.com/tmp/foo")?;
assert_eq!(url.authority(), "user:password@example.com");

pub fn cannot_be_a_base(&self) -> bool
pub fn username(&self) -> &str
let url = Url::parse("ftp://rms@example.com")?;
assert_eq!(url.username(), "rms");

pub fn password(&self) -> Option<&str>
let url = Url::parse("ftp://:secret123@example.com")?;
assert_eq!(url.password(), Some("secret123"));

pub fn has_host(&self) -> bool
pub fn host_str(&self) -> Option<&str>
pub fn host(&self) -> Option<Host<&str>>
pub fn domain(&self) -> Option<&str>
pub fn port(&self) -> Option<u16>
pub fn port_or_known_default(&self) -> Option<u16>

// Resolve a URL’s host and port number to SocketAddr.
pub fn socket_addrs( &self, default_port_number: impl Fn() -> Option<u16> ) -> Result<Vec<SocketAddr>, Error>

pub fn path(&self) -> &str
let url = Url::parse("https://example.com/countries/việt nam")?;
assert_eq!(url.path(), "/countries/vi%E1%BB%87t%20nam");

pub fn path_segments(&self) -> Option<Split<'_, char>>
pub fn query(&self) -> Option<&str>
fn run() -> Result<(), ParseError> {
let url = Url::parse("https://example.com/products?page=2")?;
let query = url.query();
assert_eq!(query, Some("page=2"));
let url = Url::parse("https://example.com/?country=español")?;
let query = url.query();
assert_eq!(query, Some("country=espa%C3%B1ol"));

pub fn query_pairs(&self) -> Parse<'_>
pub fn fragment(&self) -> Option<&str>
let url = Url::parse("https://example.com/data.csv#row=4")?;
assert_eq!(url.fragment(), Some("row=4"));

pub fn set_fragment(&mut self, fragment: Option<&str>)
pub fn set_query(&mut self, query: Option<&str>)
let mut url = Url::parse("https://example.com/products")?;
assert_eq!(url.as_str(), "https://example.com/products");
url.set_query(Some("page=2"));
assert_eq!(url.as_str(), "https://example.com/products?page=2");
assert_eq!(url.query(), Some("page=2"));

pub fn query_pairs_mut(&mut self) -> Serializer<'_, UrlQuery<'_>>
pub fn set_path(&mut self, path: &str)
let mut url = Url::parse("https://example.com/api")?;
url.set_path("data/report.csv");
assert_eq!(url.as_str(), "https://example.com/data/report.csv");
assert_eq!(url.path(), "/data/report.csv");

pub fn path_segments_mut(&mut self) -> Result<PathSegmentsMut<'_>, ()>
pub fn set_port(&mut self, port: Option<u16>) -> Result<(), ()>
pub fn set_host(&mut self, host: Option<&str>) -> Result<(), ParseError>
pub fn set_ip_host(&mut self, address: IpAddr) -> Result<(), ()>
pub fn set_password(&mut self, password: Option<&str>) -> Result<(), ()>
pub fn set_username(&mut self, username: &str) -> Result<(), ()>
pub fn set_scheme(&mut self, scheme: &str) -> Result<(), ()>
pub fn from_file_path<P>(path: P) -> Result<Url, ()> where P: AsRef<Path>,
pub fn from_directory_path<P>(path: P) -> Result<Url, ()> where P: AsRef<Path>,
pub fn to_file_path(&self) -> Result<PathBuf, ()>
```

Url::parse_with_params() 的 URL 字符串和 params 字符串都会自动进行 URL 编码；

```rust
#[tokio::main]
async fn main()  -> Result<(), Box<dyn std::error::Error>>{
    use reqwest::Url;

    let url = Url::parse_with_params(
        "https://example.net?abc=def %中&dont=clobberme#% 中",
        &[("lang", "rust"), ("&browser", "servo se#"), ("%asdfa asdf", "a中文c")],
    )?;
    println!("url: {}", url);
    Ok(())
}

// zj@a:~/.emacs.d/rust-playground/at-2024-06-05-202535$ cargo run
//    Compiling foo v0.1.0 (/Users/alizj/.emacs.d/rust-playground/at-2024-06-05-202535)
//     Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.28s
//      Running `target/debug/foo`
// url: https://example.net/?abc=def%20%%E4%B8%AD&dont=clobberme&lang=rust&%26browser=servo+se%23&%25asdfa+asdf=a%E4%B8%AD%E6%96%87c#%%20%E4%B8%AD
```


## <span class="section-num">2</span> Request/RequestBuilder {#request-requestbuilder}

Request 是可以被 Client::execute() 发送的请求：

```rust
pub struct Request { /* private fields */ }

impl Request
// 创建 Request
pub fn new(method: Method, url: Url) -> Self

pub fn method(&self) -> &Method
pub fn method_mut(&mut self) -> &mut Method

pub fn url(&self) -> &Url
pub fn url_mut(&mut self) -> &mut Url

pub fn headers(&self) -> &HeaderMap
pub fn headers_mut(&mut self) -> &mut HeaderMap

// 返回的是 reqwest::Body struct 类型
pub fn body(&self) -> Option<&Body>
pub fn body_mut(&mut self) -> &mut Option<Body>

pub fn timeout(&self) -> Option<&Duration>
pub fn timeout_mut(&mut self) -> &mut Option<Duration>

// HTTP Version
pub fn version(&self) -> Version
pub fn version_mut(&mut self) -> &mut Version

pub fn try_clone(&self) -> Option<Request>
```

RequestBuilder 用来构造 Request：

```rust
pub struct RequestBuilder { /* private fields */ }

impl RequestBuilder
pub fn from_parts(client: Client, request: Request) -> RequestBuilder

// 为请求添加一个 header
pub fn header<K, V>(self, key: K, value: V) -> RequestBuilder
where
    HeaderName: TryFrom<K>,
    <HeaderName as TryFrom<K>>::Error: Into<Error>,
    HeaderValue: TryFrom<V>,
    <HeaderValue as TryFrom<V>>::Error: Into<Error>,

// 将传入的 headers merge 到 Request 中
pub fn headers(self, headers: HeaderMap) -> RequestBuilder
pub fn basic_auth<U, P>( self, username: U, password: Option<P> ) -> RequestBuilder where U: Display, P: Display,
let client = reqwest::Client::new();
let resp = client.delete("http://httpbin.org/delete").basic_auth("admin", Some("good password")).send().await?;

pub fn bearer_auth<T>(self, token: T) -> RequestBuilder where T: Display,

// 设置请求 body
pub fn body<T: Into<Body>>(self, body: T) -> RequestBuilder

// 开启 Request timeout，从开始建立连接到响应 Body 结束。只影响这一次请求，重载
// ClientBuilder::timeout() 定义的超时。
pub fn timeout(self, timeout: Duration) -> RequestBuilder

// 发送一个  multipart/form-data body，可以包含表单数据和文件数据
pub fn multipart(self, multipart: Form) -> RequestBuilder
let client = reqwest::Client::new();
let form = reqwest::multipart::Form::new().text("key3", "value3").text("key4", "value4");
let response = client.post("your url").multipart(form).send().await?;

// 修改 URL query string，添加指定的 query 内容
pub fn query<T: Serialize + ?Sized>(self, query: &T) -> RequestBuilder
// 例如：.query(&[("foo", "a"), ("foo", "b")]) gives "foo=a&foo=b".

// 修改 HTTP 版本
pub fn version(self, version: Version) -> RequestBuilder

// 设置 Body 为传入的 URL 编码的表单数据，同时设置 Content-Type: application/x-www-form-urlencoded自
// 动对 form 数据进行 URL 编码。
pub fn form<T: Serialize + ?Sized>(self, form: &T) -> RequestBuilder
let mut params = HashMap::new();
params.insert("lang", "rust# 中"); // 会进行 URL 编码
let client = reqwest::Client::new();
let req = client
    .post("http://httpbin.org")
    .form(&params).build().unwrap();
let bytes =  req.body().unwrap().as_bytes().unwrap();
println!("req body: {:?}", String::from_utf8_lossy(bytes));
// req body: "lang=rust%23+%E4%B8%AD"

pub fn json<T: Serialize + ?Sized>(self, json: &T) -> RequestBuilder
let mut map = HashMap::new();
map.insert("lang", "rust# %#"); // 不会进行 URL 编码
map.insert("body", "json");
let client = reqwest::Client::new();
let res = client
    .post("http://httpbin.org/post")
    .json(&map)
    .build()
    .unwrap();
let bytes = res.body().unwrap().as_bytes().unwrap();
println!("req json: {:?}", String::from_utf8_lossy(bytes));
// req json: "{\"lang\":\"rust# %#\",\"body\":\"json\"}"

pub fn fetch_mode_no_cors(self) -> RequestBuilder
pub fn build(self) -> Result<Request>
pub fn build_split(self) -> (Client, Result<Request>)
pub fn send(self) -> impl Future<Output = Result<Response, Error>>
pub fn try_clone(&self) -> Option<RequestBuilder>
```


## <span class="section-num">3</span> multipart Form/Part {#multipart-form-part}

使用 multipart/form 需要开启 multipart feature。 `multipart/form-data` 是一种 MIME 类型，用于提交包含文件和二进制数据的表单。它允许表单字段和文件数据一起提交，适用于文件上传等场景。结构如下：

1.  **边界** ：开始部分 \`--boundary\`
2.  **头部** ：包含内容描述 Header，如 \`Content-Disposition\` 和 \`Content-Type\`
3.  **空行** ：用于分隔头部和内容
4.  **内容** ：表单字段或文件数据
5.  **边界** ：结束部分 \`--boundary--\`

<!--listend-->

```text
POST /upload HTTP/1.1
Host: example.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="field1"

value1
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="field2"

value2
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="example.txt"
Content-Type: text/plain

This is the content of the file.
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

Form 是一个异步的 multipart/form-data 请求，支持同时上传表单数据和文件数据。

```rust
impl Form

pub fn new() -> Form
// 返回该 From 使用的 boundary 字符串
pub fn boundary(&self) -> &str

// 表单数据
pub fn text<T, U>(self, name: T, value: U) -> Form where T: Into<Cow<'static, str>>, U: Into<Cow<'static, str>>,

// 文件数据，part 参数为 multipart::Part 对象，包含 bytes 和 filename
pub fn part<T>(self, name: T, part: Part) -> Form where T: Into<Cow<'static, str>>,

pub fn percent_encode_path_segment(self) -> Form
pub fn percent_encode_attr_chars(self) -> Form
pub fn percent_encode_noop(self) -> Form
```

使用 multipart/form-data 上传表单和文件：

```rust
use reqwest::multipart;
use reqwest::Client;
use std::fs;
use tokio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建 HTTP 客户端
    let client = Client::new();

    // 读取文件内容
    let file_content = fs::read("path/to/your/file")?;

    // 创建多部分表单
    let form = multipart::Form::new()
        .text("field1", "value1")
        .text("field2", "value2")
        .part("file", multipart::Part::bytes(file_content).file_name("your_file.txt"));

    // 发送 POST 请求
    let response = client
        .post("http://example.com/upload")
        .multipart(form)
        .send()
        .await?;

    // 检查响应状态
    if response.status().is_success() {
        println!("Upload successful!");
    } else {
        println!("Upload failed: {:?}", response.status());
    }

    Ok(())
}
```

reqwest::multipart::Part 是 multipart 的一个 part 部分，可以是表单或文件数据。作为 multipart::Form
的 part() 方法的参数来使用：

-   text() 表单数据；
-   bytes() 任意二进制数据；
    -   mime_str() 设置本 part 的 MIME 类型；
    -   file_name() 设置本 part 的 filename；
    -   headers() 设置本 part 的自定义 header；

<!--listend-->

```rust
impl Part

pub fn text<T>(value: T) -> Part where T: Into<Cow<'static, str>>,
pub fn bytes<T>(value: T) -> Part where T: Into<Cow<'static, [u8]>>,
pub fn stream<T: Into<Body>>(value: T) -> Part
pub fn stream_with_length<T: Into<Body>>(value: T, length: u64) -> Part
pub fn mime_str(self, mime: &str) -> Result<Part>
pub fn file_name<T>(self, filename: T) -> Part where T: Into<Cow<'static, str>>,
pub fn headers(self, headers: HeaderMap) -> Part
```


## <span class="section-num">4</span> Body {#body}

两类 Body：

1.  Body trait：在 [http_body crate](https://docs.rs/http-body/1.0.0/http_body/trait.Body.html) 中定义，在 hyper/axum/reqwest 中得到复用；
2.  Body struct：在 reqwest::Body 中定义；

reqwest::Body 没有构建方法，但是实现了多种类型 From trait 的转换：

```rust
pub struct Body { /* private fields */ }

impl Body
pub fn as_bytes(&self) -> Option<&[u8]>
pub fn wrap_stream<S>(stream: S) -> Body
where
    S: TryStream + Send + Sync + 'static,
    S::Error: Into<Box<dyn Error + Send + Sync>>,
    Bytes: From<S::Ok>,

// 实现 http_body::Body trait， 返回的 Data 类型是 bytes::Bytes，可以当作 &[u8] 来使用。
// http_body::Frame 是封装 http stream data 的类型，包括 HeaderMap + data。
impl Body for Body
type Data = Bytes
type Error = Error
fn poll_frame( self: Pin<&mut Self>, cx: &mut Context<'_> ) -> Poll<Option<Result<Frame<Self::Data>, Self::Error>>>


impl Debug for Body
impl Default for Body

// 创建 Body
impl From<&'static [u8]> for Body
impl From<&'static str> for Body
impl From<Bytes> for Body
impl From<File> for Body
impl From<Response> for Body
impl From<String> for Body
impl From<Vec<u8>> for Body
```

body 的使用场景：

1.  reqwest::Request::body() : 获取 body；
2.  reqwest::RequestBuilder::body(): 设置 body；


## <span class="section-num">5</span> Client/ClientBuilder {#client-clientbuilder}

Client 对象用于发送 HTTP 请求，它内部使用 connection pool，它内部使用了 Arc，是线程安全的：

```rust
impl Client
// 创建一个缺省配置的 Client
pub fn new() -> Client

// 自定义 Client
pub fn builder() -> ClientBuilder

// 返回一个 RequestBuilder
pub fn get<U: IntoUrl>(&self, url: U) -> RequestBuilder
pub fn post<U: IntoUrl>(&self, url: U) -> RequestBuilder
pub fn put<U: IntoUrl>(&self, url: U) -> RequestBuilder
pub fn patch<U: IntoUrl>(&self, url: U) -> RequestBuilder
pub fn delete<U: IntoUrl>(&self, url: U) -> RequestBuilder
pub fn head<U: IntoUrl>(&self, url: U) -> RequestBuilder
pub fn request<U: IntoUrl>(&self, method: Method, url: U) -> RequestBuilder

// 执行请求
pub fn execute(&self, request: Request ) -> impl Future<Output = Result<Response, Error>>
```

ClientBuilder 用于自定义 Client 配置：

-   user_agent()
-   default_headers()
-   cookie
-   gzip/brotli/zstd/deflate 压缩
-   proxy/no_proxy
-   timeout/read_timeout/connect_timeout

<!--listend-->

```rust
impl ClientBuilder
pub fn new() -> ClientBuilder
pub fn build(self) -> Result<Client>

pub fn user_agent<V>(self, value: V) -> ClientBuilder where V: TryInto<HeaderValue>, V::Error: Into<Error>,
pub fn default_headers(self, headers: HeaderMap) -> ClientBuilder

pub fn cookie_store(self, enable: bool) -> ClientBuilder
pub fn cookie_provider<C: CookieStore + 'static>(self, cookie_store: Arc<C>) -> ClientBuilder

pub fn gzip(self, enable: bool) -> ClientBuilder
pub fn brotli(self, enable: bool) -> ClientBuilder
pub fn zstd(self, enable: bool) -> ClientBuilder
pub fn deflate(self, enable: bool) -> ClientBuilder
pub fn no_gzip(self) -> ClientBuilder
pub fn no_brotli(self) -> ClientBuilder
pub fn no_zstd(self) -> ClientBuilder
pub fn no_deflate(self) -> ClientBuilder

pub fn redirect(self, policy: Policy) -> ClientBuilder
pub fn referer(self, enable: bool) -> ClientBuilder

pub fn proxy(self, proxy: Proxy) -> ClientBuilder
pub fn no_proxy(self) -> ClientBuilder

// 从建立连接到读取 body 结束的超时时间，默认不超时
pub fn timeout(self, timeout: Duration) -> ClientBuilder
// 读取 body 超时，默认不超时
pub fn read_timeout(self, timeout: Duration) -> ClientBuilder
pub fn connect_timeout(self, timeout: Duration) -> ClientBuilder
pub fn connection_verbose(self, verbose: bool) -> ClientBuilder

// 缺省 90s
pub fn pool_idle_timeout<D>(self, val: D) -> ClientBuilder where D: Into<Option<Duration>>,
pub fn pool_max_idle_per_host(self, max: usize) -> ClientBuilder

pub fn http1_title_case_headers(self) -> ClientBuilder
pub fn http1_allow_obsolete_multiline_headers_in_responses(self, value: bool ) -> ClientBuilder
pub fn http1_ignore_invalid_headers_in_responses( self, value: bool) -> ClientBuilder
pub fn http1_allow_spaces_after_header_name_in_responses( self, value: bool ) -> ClientBuilder
pub fn http1_only(self) -> ClientBuilder
pub fn http09_responses(self) -> ClientBuilder

pub fn http2_prior_knowledge(self) -> ClientBuilder
pub fn http2_initial_stream_window_size( self, sz: impl Into<Option<u32>> ) -> ClientBuilder
pub fn http2_initial_connection_window_size( self, sz: impl Into<Option<u32>> ) -> ClientBuilder
pub fn http2_adaptive_window(self, enabled: bool) -> ClientBuilder
pub fn http2_max_frame_size(self, sz: impl Into<Option<u32>>) -> ClientBuilder
pub fn http2_keep_alive_interval( self, interval: impl Into<Option<Duration>>) -> ClientBuilder
pub fn http2_keep_alive_timeout(self, timeout: Duration) -> ClientBuilder
pub fn http2_keep_alive_while_idle(self, enabled: bool) -> ClientBuilder

// 缺省为 true
pub fn tcp_nodelay(self, enabled: bool) -> ClientBuilder

pub fn local_address<T>(self, addr: T) -> ClientBuilder where T: Into<Option<IpAddr>>,
pub fn interface(self, interface: &str) -> ClientBuilder

pub fn tcp_keepalive<D>(self, val: D) -> ClientBuilder where D: Into<Option<Duration>>,

// 添加自定义 root 证书
pub fn add_root_certificate(self, cert: Certificate) -> ClientBuilder
// 缺省为 true，使用系统 root 证书
pub fn tls_built_in_root_certs( self, tls_built_in_root_certs: bool ) -> ClientBuilder
pub fn tls_built_in_webpki_certs(self, enabled: bool) -> ClientBuilder
pub fn tls_built_in_native_certs(self, enabled: bool) -> ClientBuilder

// 指定 client 证书
pub fn identity(self, identity: Identity) -> ClientBuilder

// 接受无效的证书
pub fn danger_accept_invalid_hostnames( self, accept_invalid_hostname: bool ) -> ClientBuilder
pub fn danger_accept_invalid_certs( self, accept_invalid_certs: bool) -> ClientBuilder

// 缺省为 true
pub fn tls_sni(self, tls_sni: bool) -> ClientBuilder

pub fn min_tls_version(self, version: Version) -> ClientBuilder
pub fn max_tls_version(self, version: Version) -> ClientBuilder
pub fn use_native_tls(self) -> ClientBuilder
pub fn use_rustls_tls(self) -> ClientBuilder
pub fn use_preconfigured_tls(self, tls: impl Any) -> ClientBuilder
pub fn tls_info(self, tls_info: bool) -> ClientBuilder

pub fn https_only(self, enabled: bool) -> ClientBuilder

pub fn trust_dns(self, enable: bool) -> ClientBuilder
pub fn hickory_dns(self, enable: bool) -> ClientBuilder
pub fn no_trust_dns(self) -> ClientBuilder
pub fn no_hickory_dns(self) -> ClientBuilder

// 将 domain 的地址重写为指定值
pub fn resolve(self, domain: &str, addr: SocketAddr) -> ClientBuilder
// 将 domain 的域名解析结果重写为指定 slice
pub fn resolve_to_addrs( self, domain: &str, addrs: &[SocketAddr]) -> ClientBuilder
pub fn dns_resolver<R: Resolve + 'static>( self, resolver: Arc<R>) -> ClientBuilder
```


## <span class="section-num">6</span> Response {#response}

```rust
pub struct Response { /* private fields */ }

impl Response
pub fn status(&self) -> StatusCode
pub fn version(&self) -> Version
pub fn headers(&self) -> &HeaderMap
pub fn headers_mut(&mut self) -> &mut HeaderMap
pub fn content_length(&self) -> Option<u64>
pub fn cookies<'a>(&'a self) -> impl Iterator<Item = Cookie<'a>> + 'a
pub fn url(&self) -> &Url

pub fn remote_addr(&self) -> Option<SocketAddr>
pub fn extensions(&self) -> &Extensions
pub fn extensions_mut(&mut self) -> &mut Extensions

// 返回 utf-8 编码的 body 文本
pub async fn text(self) -> Result<String>
pub async fn text_with_charset(self, default_encoding: &str) -> Result<String>
let content = reqwest::get("http://httpbin.org/range/26") .await? .text() .await?;
println!("text: {content:?}");

pub async fn json<T: DeserializeOwned>(self) -> Result<T>
#[derive(Deserialize)]
struct Ip { origin: String, }
let ip = reqwest::get("http://httpbin.org/ip") .await? .json::<Ip>() .await?;
println!("ip: {}", ip.origin);

pub async fn bytes(self) -> Result<Bytes> // Bytes 为 bytes create 的 Bytes 类型
let bytes = reqwest::get("http://httpbin.org/ip") .await? .bytes() .await?;
println!("bytes: {bytes:?}");

pub async fn chunk(&mut self) -> Result<Option<Bytes>> // 流式响应
let mut res = reqwest::get("https://hyper.rs").await?;
while let Some(chunk) = res.chunk().await? {
    println!("Chunk: {chunk:?}");
}

pub fn bytes_stream(self) -> impl Stream<Item = Result<Bytes>>
use futures_util::StreamExt;
let mut stream = reqwest::get("http://httpbin.org/ip") .await? .bytes_stream();
while let Some(item) = stream.next().await {
    println!("Chunk: {:?}", item?);
}

pub fn error_for_status(self) -> Result<Self>
pub fn error_for_status_ref(&self) -> Result<&Self>

impl Response
pub async fn upgrade(self) -> Result<Upgraded>
```


## <span class="section-num">7</span> header {#header}

reqwest::header 实际是 <http::header> module.
