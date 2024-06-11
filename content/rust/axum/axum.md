---
title: "axum"
author: ["zhangjun"]
lastmod: 2024-06-11T20:27:39+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 10
---

axum 是基于 hyper 实现的高层 HTTP Server，支持异步。

1.  定义 Router：通过 Route.route() 方法来定义 PATH 和关联的 Service；
2.  RouterMethod 类型实现了 Service trait，routing::get/post/patch() 等函数返回 RouterMehtod 对象，这些函数的输入是 Handler；
3.  Handler 一般是闭包：
    1.  输入参数是 Extractor，实现 FromRequest or FromRequestParts. 来提取所需信息；
    2.  返回值是 IntoResponse 对象，而不是包含 Error 的 Result，所以 axum 不会返回 Error，如果要返回出错信息，也是自定义 Error type 并实现 IntoResponse；
4.  Router/RouterMethod/Handler 三级都可以通过 layer() 方法来关联 middleware，从而在次之前先做一些处理；
    1.  Router.with_state() 对所有 req 都适用，是全局的；
    2.  RouterMethod.with_state() 只对该 PATH 的所有 Handler 适用；
    3.  Handler.with_state() 只对 PATH 下的特定 method 如 get 适用；
    4.  另外 middleware 也可以通过 request extension 来向 handler 传递 state；

<!--listend-->

```rust
use axum::{
    routing::get,
    Router,
};

#[tokio::main]
async fn main() {
    // build our application with a single route
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));
    // run our app with hyper, listening globally on port 3000
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```


## <span class="section-num">1</span> axum::serve() {#axum-serve}

axum::serve() 函数是 auxm 的入口：

1.  make_service 参数实现了 Service trait, 它的 Response 也实现了另一个 Service，该 Service 的输入是
    Request，响应是 Response。所以 make_service 名称的含义是 make service。
2.  返回的 Servie 支持 tokio 的 graceful shutdown。

<!--listend-->

```rust
pub fn serve<M, S>(tcp_listener: TcpListener, make_service: M) -> Serve<M, S>
where
    M: for<'a> Service<IncomingStream<'a>, Error = Infallible, Response = S>,
    S: Service<Request, Response = Response, Error = Infallible> + Clone + Send + 'static,
    S::Future: Send,
```

Router/MethodRouter/Handler 都实现了 Service&lt;IncomingStream&lt;'_&gt;&gt;, 它们都可以作为 server 的
make_service 参数：

```rust
// Serving a Router:
use axum::{Router, routing::get};
let router = Router::new().route("/", get(|| async { "Hello, World!" }));
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, router).await.unwrap();

// Serving a MethodRouter:
use axum::routing::get;
let router = get(|| async { "Hello, World!" });
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, router).await.unwrap();

// Serving a Handler:
use axum::handler::HandlerWithoutStateExt;
async fn handler() -> &'static str { "Hello, World!"}
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, handler.into_make_service()).await.unwrap();
```


## <span class="section-num">2</span> Router {#router}

struct Router 类型用于组合 handlers 和 services，它实现了 Service&lt;IncomingStream&lt;'_&gt;&gt;，故可以作为
axum::server() 的入口参数：

route()
: 为 path 指定 MethodRouter 封装的处理逻辑，auxm::routing modle 提供的函数
    get/delete/put/post() 返回预定义的 MethodRouter。
    -   MethodRouter 封装了请求 Method 及其 Handler 处理逻辑，而且可以关联多种请求 Method，实现链式匹配和处理。

route_service()
: 添加一个 Service；

<!--listend-->

```rust
// S 为 Router 的 State，缺省为 ();
pub struct Router<S = ()> { /* private fields */ }


// 添加一个对 path 的 MethodRouter 处理
pub fn route(self, path: &str, method_router: MethodRouter<S>) -> Self
use axum::{Router, routing::{get, delete}, extract::Path};
let app = Router::new()
    .route("/", get(root))
    .route("/users", get(list_users).post(create_user))
    .route("/users/:id", get(show_user))
    .route("/api/:version/users/:id/action", delete(do_users_action))
    .route("/assets/*path", get(serve_asset));
async fn root() {}
async fn list_users() {}
async fn create_user() {}
async fn show_user(Path(id): Path<u64>) {}
async fn do_users_action(Path((version, id)): Path<(String, u64)>) {}
async fn serve_asset(Path(path): Path<String>) {}


// 添加一个对 path 的 Service 处理，Service 由 tower::Service 定义，可以 tower::service_fn() 来从函
// 数或闭包快速创建：
pub fn route_service<T>(self, path: &str, service: T) -> Self
where
    T: Service<Request, Error = Infallible> + Clone + Send + 'static,
    T::Response: IntoResponse,
    T::Future: Send + 'static,
use axum::{ Router, body::Body, routing::{any_service, get_service}, extract::Request, http::StatusCode,
    error_handling::HandleErrorLayer,};
use tower_http::services::ServeFile;
use http::Response;
use std::{convert::Infallible, io};
use tower::service_fn;
let app = Router::new()
    .route( "/", any_service(service_fn(|_: Request| async {
        let res = Response::new(Body::from("Hi from `GET /`"));
        Ok::<_, Infallible>(res)
    })))
    .route_service( "/foo", service_fn(|req: Request| async move { // 从闭包创建 Service
        let body = Body::from(format!("Hi from `{} /foo`", req.method()));
        let res = Response::new(body);
        Ok::<_, Infallible>(res)
    }))
    .route_service( "/static/Cargo.toml", ServeFile::new("Cargo.toml"), );
use axum::{ extract::Request, Router, routing::{MethodFilter, on_service}, body::Body,};
use http::Response;
use std::convert::Infallible;
let service = tower::service_fn(|request: Request| async { Ok::<_, Infallible>(Response::new(Body::empty()))});
// Requests to `DELETE /` will go to `service`
let app = Router::new().route("/", on_service(MethodFilter::DELETE, service));


// 添加嵌套的 Router
pub fn nest(self, path: &str, router: Router<S>) -> Self
pub fn nest_service<T>(self, path: &str, service: T) -> Self
where
    T: Service<Request, Error = Infallible> + Clone + Send + 'static,
    T::Response: IntoResponse,
    T::Future: Send + 'static,


// 将多个 Router 的合并到一起
pub fn merge<R>(self, other: R) -> Self where R: Into<Router<S>>,
let user_routes = Router::new() .route("/users", get(users_list)) .route("/users/:id", get(users_show));
let team_routes = Router::new() .route("/teams", get(teams_list));
let app = Router::new() .merge(user_routes) .merge(team_routes);


// 为 Router 所有的 Route 都添加 layer middleware
// layer() 获取所有权，返回一个 Router<S>
pub fn layer<L>(self, layer: L) -> Router<S>
where
    L: Layer<Route> + Clone + Send + 'static,
    L::Service: Service<Request> + Clone + Send + 'static,
    <L::Service as Service<Request>>::Response: IntoResponse + 'static,
    <L::Service as Service<Request>>::Error: Into<Infallible> + 'static,
    <L::Service as Service<Request>>::Future: Send + 'static,
use axum::{routing::get, Router};
use tower_http::trace::TraceLayer;
let app = Router::new()
    .route("/foo", get(|| async {}))
    .route("/bar", get(|| async {}))
    .layer(TraceLayer::new_for_http());


// 只为匹配 Route 的请求添加 layer
pub fn route_layer<L>(self, layer: L) -> Self
where
    L: Layer<Route> + Clone + Send + 'static,
    L::Service: Service<Request> + Clone + Send + 'static,
    <L::Service as Service<Request>>::Response: IntoResponse + 'static,
    <L::Service as Service<Request>>::Error: Into<Infallible> + 'static,
    <L::Service as Service<Request>>::Future: Send + 'static,
use axum::{ routing::get, Router,};
use tower_http::validate_request::ValidateRequestHeaderLayer;
let app = Router::new() .route("/foo",
    get(|| async {})) .route_layer(ValidateRequestHeaderLayer::bearer("password"));
// `GET /foo` with a valid token will receive `200 OK`
// `GET /foo` with a invalid token will receive `401 Unauthorized`
// `GET /not-found` with a invalid token will receive `404 Not Found`


// 没有匹配的 Route 时 fallback 到的 handler
pub fn fallback<H, T>(self, handler: H) -> Self where H: Handler<T, S>, T: 'static,
pub fn fallback_service<T>(self, service: T) -> Self
where
    T: Service<Request, Error = Infallible> + Clone + Send + 'static,
    T::Response: IntoResponse,
    T::Future: Send + 'static,
let app = Router::new()
    .route("/foo", get(|| async { /* ... */ }))
    .fallback(fallback);
async fn fallback(uri: Uri) -> (StatusCode, String) {}


// 为 Router 提供 state，后续通过 handler 的 extract 来获得该  state
// state 可以是实现 Clone 的任意自定义对象类型
// 为 Router 的所有请求提供全局数据，不适合但给请求的数据（用 Extension）。
pub fn with_state<S2>(self, state: S) -> Router<S2>
use axum::{Router, routing::get, extract::State};
#[derive(Clone)]
struct AppState {}
let routes = Router::new()
    // 使用 axum::extract::State 来为请求获得 global state
    .route("/", get(|State(state): State<AppState>| async {
        // 使用 state
    })).with_state(AppState {});
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, routes).await.unwrap();


pub fn as_service<B>(&mut self) -> RouterAsService<'_, B, S>
use tower::{Service, ServiceExt};
let mut router = Router::new().route("/", get(|| async {}));
let request = Request::new(Body::empty());
let response = router.as_service().ready().await?.call(request).await?;


// 将 Route 转换为 MakeService, 它是创建另一个 Service 的 Service
// 主要的使用场景是作为 axum::serve 的参数。
pub fn into_make_service(self) -> IntoMakeService<Self>
use axum::{routing::get, Router, };
let app = Router::new().route("/", get(|| async { "Hi!" }));
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app.into_make_service()).await.unwrap();
```


## <span class="section-num">3</span> MethodRouter {#methodrouter}

MethodRouter 类型是 Router::route(path, method_router) 方法的参数，为 path 提供处理逻辑。

MethodRouter 封装了请求 Method 及其 Handler 处理逻辑，可以链式调用提供的方法，实现根据 Method 来进行不同的 Hander 处理：

-   Handler 一般用闭包来实现。

<!--listend-->

```rust
impl<S> MethodRouter<S, Infallible> where S: Clone,

// MethodRouter 的方法
// on()/on_service() 是通用方法，是其他方法如 get()/delete() 等的基础
pub fn on<H, T>(self, filter: MethodFilter, handler: H) -> Self
where
    H: Handler<T, S>,
    T: 'static,
    S: Send + Sync + 'static,

pub fn on_service<T>(self, filter: MethodFilter, svc: T) -> Self
where
    T: Service<Request, Error = E> + Clone + Send + 'static,
    T::Response: IntoResponse + 'static,
    T::Future: Send + 'static,

// 其他返回 MethodRouter 的方法，是 on() 方法的封装，可以链式调用
pub fn delete<H, T>(self, handler: H) -> Self
where
    H: Handler<T, S>,
    T: 'static,
    S: Send + Sync + 'static,

pub fn get<H, T>(self, handler: H) -> Self
where
    H: Handler<T, S>,
    T: 'static,
    S: Send + Sync + 'static,

pub fn head<H, T>(self, handler: H) -> Self
where
    H: Handler<T, S>,
    T: 'static,
    S: Send + Sync + 'static,
```

auxm::routing modle 提供了一些特定 Method 的 MethodRouter 的创建函数 get/delete/put/post() 等，便于在 Router::router() 中快速使用：

```rust
// auxm::routing::get()
pub fn get<H, T, S>(handler: H) -> MethodRouter<S, Infallible>
where
    H: Handler<T, S>,
    T: 'static,
    S: Clone + Send + Sync + 'static,

// 示例
use axum::{routing::get, Router, routing::MethodFilter};
async fn handler() {}
async fn other_handler() {}
// 链式调用
// Requests to `GET /` will go to `handler` and `DELETE /` will go to `other_handler`
let app = Router::new().route("/", get(handler).on(MethodFilter::DELETE, other_handler));
```

MethodRouter 支持添加 state 和 layer middleware，但只对该 MethodRouter 的 Handler 有效：

```rust
pub fn with_state<S2>(self, state: S) -> MethodRouter<S2, E>
pub fn route_layer<L>(self, layer: L) -> MethodRouter<S, E>
where
    L: Layer<Route<E>> + Clone + Send + 'static,
    L::Service: Service<Request, Error = E> + Clone + Send + 'static,
    <L::Service as Service<Request>>::Response: IntoResponse + 'static,
    <L::Service as Service<Request>>::Future: Send + 'static,
    E: 'static,
    S: 'static,

use axum::{ routing::get, Router, };
use tower_http::validate_request::ValidateRequestHeaderLayer;
let app = Router::new().route(
    "/foo",
    get(|| async {}).route_layer(ValidateRequestHeaderLayer::bearer("password"))
);
// `GET /foo` with a valid token will receive `200 OK`
// `GET /foo` with a invalid token will receive `401 Unauthorized`
// `POST /FOO` with a invalid token will receive `405 Method Not Allowed`
```

MethodRouter 是否实现 tower::Service&lt;Request&gt; trait，取决于它的 State 情况：

```rust
use tower::Service;
use axum::{routing::get, extract::{State, Request}, body::Body};

// this `MethodRouter` doesn't require any state, i.e. the state is `()`,
let method_router = get(|| async {});
// and thus it implements `Service`
assert_service(method_router);

// this requires a `String` and doesn't implement `Service`
let method_router = get(|_: State<String>| async {});
// until you provide the `String` with `.with_state(...)`
let method_router_with_state = method_router.with_state(String::new());
// and then it implements `Service`
assert_service(method_router_with_state);

// helper to check that a value implements `Service`
fn assert_service<S>(service: S) where S: Service<Request>, {}
```


## <span class="section-num">4</span> Handler {#handler}

Handler 是 MethodRouter 的各方法 on/get/delete/put/post() 使用的处理逻辑，一般是 async 闭包实现。

-   Handler 除了 async fun 实现外，也可以是任何实现 IntoResponse 的非函数对象；

Handler 闭包函数的输入是 0 个或多个 extractor，返回是实现 IntoResponse 的对象。

-   Handler aysnc fn 的响应是 IntoResponse 对象，而非包含 Error 的 Result 对象，所以不能直接返 回Error，所以 axum 不会返回 Error，如果要返回出错信息，也是自定义 Error type 并实现 IntoResponse；

<!--listend-->

```rust
// 必须是异步闭包或函数，输入是 extractors，输出是 IntoResponse

use axum::{body::Bytes, http::StatusCode};

// Handler that immediately returns an empty `200 OK` response.
async fn unit_handler() {}

// Handler that immediately returns an empty `200 OK` response with a plain text body.
async fn string_handler() -> String {
    "Hello, World!".to_string()
}

// Handler that buffers the request body and returns it.
//
// This works because `Bytes` implements `FromRequest` and therefore can be used as an extractor.
//
// `String` and `StatusCode` both implement `IntoResponse` and therefore `Result<String,
// StatusCode>` also implements `IntoResponse`
async fn echo(body: Bytes) -> Result<String, StatusCode> {
    if let Ok(string) = String::from_utf8(body.to_vec()) {
        Ok(string)
    } else {
        Err(StatusCode::BAD_REQUEST)
    }
}
```

如果使用 async fn 作为 Handler，则它必须要实现 Handler trait，axum 默认提供为 16 个参数内的 FnOnce闭包实现了 Handler（blanket implementation）：

-   FnOnce 必须实现 Clone + Send + 'static;
-   FnOnce 返回 的 Output 必须实现 IntoResponse；
-   FnOnce(T1), T1 必须实现 FromRequest；
-   其他带更多参数的 FnOnce(T1, T2, T3) 等的 FnOnce，前面的参数必须实现 FromRequestParts，最后一个参数实现 FromRequest；
    -   FromRequestParts 不消耗 body，而 FromRequest 消耗 body 而且只能消耗一次；

这些 FnOnce 函数或闭包返回的结果是 Future&lt;Output = IntoResponse&gt;，并没有 Error 信息，所以Handler 没有提供出错返回的机制。

```rust
// 一些实现 Handler 的闭包函数示例
impl<F, Fut, Res, S> Handler<((),), S> for F
where
    F: FnOnce() -> Fut + Clone + Send + 'static,
    Fut: Future<Output = Res> + Send,
    Res: IntoResponse,

impl<F, Fut, S, Res, M, T1> Handler<(M, T1), S> for F
where
    F: FnOnce(T1) -> Fut + Clone + Send + 'static,
    Fut: Future<Output = Res> + Send,
    S: Send + Sync + 'static,
    Res: IntoResponse,
    T1: FromRequest<S, M> + Send,

impl<F, Fut, S, Res, M, T1, T2> Handler<(M, T1, T2), S> for F
where
    F: FnOnce(T1, T2) -> Fut + Clone + Send + 'static,
    Fut: Future<Output = Res> + Send,
    S: Send + Sync + 'static,
    Res: IntoResponse,
    T1: FromRequestParts<S> + Send,
    T2: FromRequest<S, M> + Send,

impl<F, Fut, S, Res, M, T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11, T12, T13, T14, T15, T16> Handler<(M, T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11, T12, T13, T14, T15, T16), S> for F
where
    F: FnOnce(T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11, T12, T13, T14, T15, T16) -> Fut + Clone + Send + 'static,
    Fut: Future<Output = Res> + Send,
    S: Send + Sync + 'static,
    Res: IntoResponse,
    T1: FromRequestParts<S> + Send,
    T2: FromRequestParts<S> + Send,
    T3: FromRequestParts<S> + Send,
    T4: FromRequestParts<S> + Send,
    T5: FromRequestParts<S> + Send,
    T6: FromRequestParts<S> + Send,
    T7: FromRequestParts<S> + Send,
    T8: FromRequestParts<S> + Send,
    T9: FromRequestParts<S> + Send,
    T10: FromRequestParts<S> + Send,
    T11: FromRequestParts<S> + Send,
    T12: FromRequestParts<S> + Send,
    T13: FromRequestParts<S> + Send,
    T14: FromRequestParts<S> + Send,
    T15: FromRequestParts<S> + Send,
    T16: FromRequest<S, M> + Send,

impl<H, S, T, L> Handler<T, S> for Layered<L, H, T, S>
where
    L: Layer<HandlerService<H, T, S>> + Clone + Send + 'static,
    H: Handler<T, S>,
    L::Service: Service<Request, Error = Infallible> + Clone + Send + 'static,
    <L::Service as Service<Request>>::Response: IntoResponse,
    <L::Service as Service<Request>>::Future: Send,
    T: 'static,
    S: 'static,

impl<S> Handler<(), S> for MethodRouter<S>
where
    S: Clone + 'static,
```

Handler 也提供了 layer() 和 wit_state() 方法，可以用来添加中间件和状态：

```rust
fn layer<L>(self, layer: L) -> Layered<L, Self, T, S>
where
    L: Layer<HandlerService<Self, T, S>> + Clone,
    L::Service: Service<Request>,
fn with_state(self, state: S) -> HandlerService<Self, T, S>

// 示例
use axum::{ routing::get, handler::Handler, Router, };
use tower::limit::{ConcurrencyLimitLayer, ConcurrencyLimit};
async fn handler() { /* ... */ }
let layered_handler = handler.layer(ConcurrencyLimitLayer::new(64));
let app = Router::new().route("/", get(layered_handler));
```


## <span class="section-num">5</span> IntoResponse {#intoresponse}

除了 FnOnce 闭包函数外，任何实现 IntoResponse 的类型也实现了 Handler （虽然文档没有提：
<https://docs.rs/axum/latest/src/axum/handler/mod.rs.html#258>）， 可以作为 MethodRouter 的各方法
on/get/delete/put/post() 使用的处理逻辑：

```rust
pub trait IntoResponse {
    // Required method
    fn into_response(self) -> Response<Body>;
}

mod private {
    // Marker type for `impl<T: IntoResponse> Handler for T`
    #[allow(missing_debug_implementations)]
    pub enum IntoResponseHandler {}
}

impl<T, S> Handler<private::IntoResponseHandler, S> for T
where
    T: IntoResponse + Clone + Send + 'static,
{
    type Future = std::future::Ready<Response>;

    fn call(self, _req: Request, _state: S) -> Self::Future {
        std::future::ready(self.into_response())
    }
}
```

比如使用一个 tuple 来作为 Handler：

```rust
use axum::{
    Router,
    routing::{get, post},
    Json,
    http::StatusCode,
};
use serde_json::json;

let app = Router::new()
    // respond with a fixed string
    .route("/", get("Hello, World!")) // &str 实现了 IntoResponse
    // or return some mock data
    .route("/users", post(
        // tuple 的成员都实现了 IntoResponse， 所以 tuple 也实现了 IntoResponse
        (StatusCode::CREATED, Json(json!({ "id": 1, "username": "alice" })),
    )));
```

axum 默认实现 IntoResponse 的类型：

-   字符串、数组、Bytes、HeaderMap、Parts、StatusCode 等都实现了 IntoResponse：
-   部分 tuple 类型，如 (Parts, R) (Response, R)  (StatusCode, R) (R,) 也实现了 IntoResponse：

<!--listend-->

```rust
impl IntoResponse for &'static str
impl IntoResponse for &'static [u8]
impl IntoResponse for Cow<'static, str>
impl IntoResponse for Cow<'static, [u8]>
impl IntoResponse for Infallible
impl IntoResponse for ()
impl IntoResponse for Box<str>
impl IntoResponse for Box<[u8]>
impl IntoResponse for String
impl IntoResponse for Vec<u8>
impl IntoResponse for Bytes
impl IntoResponse for BytesMut
impl IntoResponse for Extensions
impl IntoResponse for HeaderMap
impl IntoResponse for Parts
impl IntoResponse for StatusCode

impl<B> IntoResponse for Response<B>
where
    B: Body<Data = Bytes> + Send + 'static,
    <B as Body>::Error: Into<Box<dyn Error + Send + Sync>>,

impl<K, V, const N: usize> IntoResponse for [(K, V); N]
where
    K: TryInto<HeaderName>,
    <K as TryInto<HeaderName>>::Error: Display,
    V: TryInto<HeaderValue>,
    <V as TryInto<HeaderValue>>::Error: Display,

// tuple 类型
impl<R> IntoResponse for (Parts, R) where R: IntoResponse,
impl<R> IntoResponse for (Response<()>, R) where R: IntoResponse,
impl<R> IntoResponse for (StatusCode, R) where R: IntoResponse,
impl<R> IntoResponse for (R,) where R: IntoResponse,

impl<R, T1> IntoResponse for (Parts, T1, R) where T1: IntoResponseParts, R: IntoResponse,
impl<R, T1> IntoResponse for (Response<()>, T1, R) where T1: IntoResponseParts, R: IntoResponse,
impl<R, T1> IntoResponse for (StatusCode, T1, R) where T1: IntoResponseParts, R: IntoResponse,
impl<R, T1> IntoResponse for (T1, R) where T1: IntoResponseParts, R: IntoResponse,
impl<R> IntoResponse for (R,) where R: IntoResponse,

fn into_response(self) -> Response<Body>
impl<R, T1> IntoResponse for (Parts, T1, R) where T1: IntoResponseParts, R: IntoResponse,
impl<R, T1> IntoResponse for (Response<()>, T1, R) where T1: IntoResponseParts, R: IntoResponse,
impl<R, T1> IntoResponse for (StatusCode, T1, R) where T1: IntoResponseParts, R: IntoResponse,
impl<R, T1> IntoResponse for (T1, R) where T1: IntoResponseParts, R: IntoResponse,
```


## <span class="section-num">6</span> extractor {#extractor}

extractor 是实现了 `FromRequest or FromRequestParts` trait 的类型，它们作为 Handler 闭包函数的输入参数，用于提取相关信息，auxum 会自动从请求中调用这些 extrator 实现的 FromRequest or FromRequestParts 来构造 extractor 对象，然后供 Handler 使用。

对于有多个 extractor 输入参数的 Handler 闭包函数，如 FnOnce(T1, T2, T3) ，前面的参数必须实现
FromRequestParts，最后一个参数实现 FromRequest；

-   FromRequestParts 不消耗 body，而 FromRequest 消耗 body 而且只能消耗一次；

<!--listend-->

```rust
use axum::{
    // Path/Query/Json 均是 extractor 对象类型（struct tuple）
    extract::{Request, Json, Path, Extension, Query},
    routing::post,
    http::header::HeaderMap,
    body::{Bytes, Body},
    Router,
};
use serde_json::Value;
use std::collections::HashMap;


// 下面的 path/query/headers async fn 均是可以创建 Handler 的异步函数，它们的参数
// Path/Query/HeaderMap 都是 extractor 对象类型，实现了 FromRequest 或 FromRequestParts

// `Path` gives you the path parameters and deserializes them. See its docs for more details
async fn path(Path(user_id): Path<u32>) {}

// `Query` gives you the query parameters and deserializes them.
async fn query(Query(params): Query<HashMap<String, String>>) {}

// `HeaderMap` gives you all the headers
async fn headers(headers: HeaderMap) {}

// 获得整个 Request body 的内容
// `String` consumes the request body and ensures it is valid utf-8
async fn string(body: String) {}

// `Bytes` gives you the raw request body
async fn bytes(body: Bytes) {}

// We've already seen `Json` for parsing the request body as json
async fn json(Json(payload): Json<Value>) {}

// `Request` gives you the whole request for maximum control
async fn request(request: Request) {}

// `Extension` extracts data from "request extensions" This is commonly used to share state with
// handlers
async fn extension(Extension(state): Extension<State>) {}

#[derive(Clone)]
struct State { /* ... */ }
let app = Router::new()
    .route("/path/:user_id", post(path))
    .route("/query", post(query))
    .route("/string", post(string))
    .route("/bytes", post(bytes))
    .route("/json", post(json))
    .route("/request", post(request))
    .route("/extension", post(extension));
```

axum::extract module 提供了一些常用的 extractor 类型：

-   JSON
-   Form
-   Request
-   HeaderMap
-   Extension
-   ConnectInfo
-   Host
-   MatchedPath
-   MultiPart
-   NestedPath
-   OriginalUrl
-   Path
-   Query
-   RawForm
-   RawPathParams
-   RawQuery
-   State

JSON：实现了 FromRequest 和 IntoResponse trait，可以作为 Handler 的输入和输出类型：

```rust
pub struct Json<T>(pub T);

// Extractor example
use axum::{
    extract,
    routing::post,
    Router,
};
use serde::Deserialize;

#[derive(Deserialize)]
struct CreateUser {
    email: String,
    password: String,
}

async fn create_user(extract::Json(payload): extract::Json<CreateUser>) {
    // payload is a `CreateUser`
}

let app = Router::new().route("/users", post(create_user));

// Response example
use axum::{
    extract::Path,
    routing::get,
    Router,
    Json,
};
use serde::Serialize;
use uuid::Uuid;

#[derive(Serialize)]
struct User {
    id: Uuid,
    username: String,
}

async fn get_user(Path(user_id) : Path<Uuid>) -> Json<User> {
    let user = find_user(user_id).await;
    Json(user)
}

async fn find_user(user_id: Uuid) -> User {
    // ...
}

let app = Router::new().route("/users/:id", get(get_user));
```

-   Form：实现了 FromRequest 和 IntoResponse trait，可以作为 Handler 的输入和输出类型：

<!--listend-->

```rust
pub struct Form<T>(pub T);  // 实现了 FromRequest

use axum::Form;
use serde::Deserialize;

#[derive(Deserialize)]
struct SignUp {
    username: String,
    password: String,
}

async fn accept_form(Form(sign_up): Form<SignUp>) {
    // ...
}


// Response
use axum::Form;
use serde::Serialize;

#[derive(Serialize)]
struct Payload {
    value: String,
}

async fn handler() -> Form<Payload> {
    Form(Payload { value: "foo".to_owned() })
}
```

Request：\`Request\` gives you the whole request for maximum control

```rust
// `Request` gives you the whole request for maximum control
async fn request(request: Request) {}
```

HeaderMap：

```rust
// `HeaderMap` gives you all the headers
async fn headers(headers: HeaderMap) {}
```

Extension 是从 http request extension 向 handler 传递消息的机制。

-   Extension 实现了 FromRequestParts 和 Layer&lt;S&gt; 和 IntoResponse, 所以可以作为 extranctor、layer 中间件、和响应：
-   主要的使用场景：layer middleware 使用 http rquest extension 来向 handler 传递数据。
    <https://docs.rs/axum/latest/axum/middleware/index.html#passing-state-from-middleware-to-handlers>

<!--listend-->

```rust
// `Extension` extracts data from "request extensions" This is commonly used to share state with
// handlers
async fn extension(Extension(state): Extension<State>) {}

// As extractor：This is commonly used to share state across handlers.
use axum::{
    Router,
    Extension,
    routing::get,
};
use std::sync::Arc;

// Some shared state used throughout our application
struct State {
    // ...
}
async fn handler(state: Extension<Arc<State>>) {
    // ...
}
let state = Arc::new(State { /* ... */ });
let app = Router::new().route("/", get(handler))
    // Add middleware that inserts the state into all incoming request's
    // extensions.
    .layer(Extension(state));


// 作为响应
use axum::{
    Extension,
    response::IntoResponse,
};
async fn handler() -> (Extension<Foo>, &'static str) {
    (
        Extension(Foo("foo")),
        "Hello, World!"
    )
}
#[derive(Clone)]
struct Foo(&'static str);

// Passing state from middleware to handlers
// State can be passed from middleware to handlers using request extensions:
use axum::{
    Router,
    http::StatusCode,
    routing::get,
    response::{IntoResponse, Response},
    middleware::{self, Next},
    extract::{Request, Extension},
};

#[derive(Clone)]
struct CurrentUser { /* ... */ }

async fn auth(mut req: Request, next: Next) -> Result<Response, StatusCode> {
    let auth_header = req.headers()
        .get(http::header::AUTHORIZATION)
        .and_then(|header| header.to_str().ok());

    let auth_header = if let Some(auth_header) = auth_header {
        auth_header
    } else {
        return Err(StatusCode::UNAUTHORIZED);
    };

    if let Some(current_user) = authorize_current_user(auth_header).await {
        // insert the current user into a request extension so the handler can
        // extract it
        req.extensions_mut().insert(current_user);
        Ok(next.run(req).await)
    } else {
        Err(StatusCode::UNAUTHORIZED)
    }
}

async fn authorize_current_user(auth_token: &str) -> Option<CurrentUser> {
    // ...
}

async fn handler(
    // extract the current user, set by the middleware
    Extension(current_user): Extension<CurrentUser>,
) {
    // ...
}

let app = Router::new()
    .route("/", get(handler))
    .route_layer(middleware::from_fn(auth));
```

ConnectInfo： Extractor for getting connection information produced by a Connected.

-   需要和 Router.into_make_service_with_connect_info() 连用；

<!--listend-->

```rust
use axum::{
    extract::connect_info::{ConnectInfo, Connected},
    routing::get,
    serve::IncomingStream,
    Router,
};

let app = Router::new().route("/", get(handler));

async fn handler(
    ConnectInfo(my_connect_info): ConnectInfo<MyConnectInfo>,
) -> String {
    format!("Hello {my_connect_info:?}")
}

#[derive(Clone, Debug)]
struct MyConnectInfo {
    // ...
}

impl Connected<IncomingStream<'_>> for MyConnectInfo {
    fn connect_info(target: IncomingStream<'_>) -> Self {
        MyConnectInfo {
            // ...
        }
    }
}

let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app.into_make_service_with_connect_info::<MyConnectInfo>()).await.unwrap();
```

DefaultBodyLimit： `Layer` for configuring the default request body limit. 其实并不是 extractor

```rust
use axum::{
    Router,
    routing::post,
    body::Body,
    extract::{Request, DefaultBodyLimit},
};

let app = Router::new()
    // change the default limit
    .layer(DefaultBodyLimit::max(1024))
    // this route has a different limit
    .route("/", post(|request: Request| async {}).layer(DefaultBodyLimit::max(1024)))
    // this route still has the default limit
    .route("/foo", post(|request: Request| async {}));
```

Host：    Extractor that resolves the hostname of the request. 实现了 FromRequestParts

Hostname is resolved through the following, in order:
  Forwarded header
  X-Forwarded-Host header
  Host header
  request target / URI

MatchedPath： Access the path in the router that matches the request. 返回的 path 为 Router 原始路基该字符串。

```rust
    Router,
    extract::MatchedPath,
    routing::get,
};

let app = Router::new().route(
    "/users/:id",
    get(|path: MatchedPath| async move {
        let path = path.as_str();
        // `path` will be "/users/:id"
    })
);
```

Multipart： Extractor that parses multipart/form-data requests (commonly used with file uploads).
实现了 FromRequest&lt;S&gt; 消耗 body，所以只能作为 handler 函数最后一个参数且使用一次：

```rust
use axum::{
    extract::Multipart,
    routing::post,
    Router,
};
use futures_util::stream::StreamExt;

async fn upload(mut multipart: Multipart) {
    while let Some(mut field) = multipart.next_field().await.unwrap() {
        let name = field.name().unwrap().to_string();
        let data = field.bytes().await.unwrap();

        println!("Length of `{}` is {} bytes", name, data.len());
    }
}

let app = Router::new().route("/upload", post(upload));
```

NestedPath：    Access the path the matched the route is nested at. 实现了 FromRequestParts

```rust
use axum::{
    Router,
    extract::NestedPath,
    routing::get,
};

let api = Router::new().route(
    "/users",
    get(|path: NestedPath| async move {
        // `path` will be "/api" because thats what this
        // router is nested at when we build `app`
        let path = path.as_str();
    })
);

let app = Router::new().nest("/api", api);
```

OriginalUri：    Extractor that gets the original request URI regardless of nesting.

```rust
use axum::{
    routing::get,
    Router,
    extract::OriginalUri,
    http::Uri
};

let api_routes = Router::new()
    .route(
        "/users",
        get(|uri: Uri, OriginalUri(original_uri): OriginalUri| async {
            // `uri` is `/users`
            // `original_uri` is `/api/users`
        }),
    );

let app = Router::new().nest("/api", api_routes);
```

Path：    Extractor that will get captures from the URL and parse them using serde.

```rust
use axum::{
    extract::Path,
    routing::get,
    Router,
};
use uuid::Uuid;

async fn users_teams_show(
    Path((user_id, team_id)): Path<(Uuid, Uuid)>,
) {
    // ...
}

let app = Router::new().route("/users/:user_id/team/:team_id", get(users_teams_show));
```

Query：    Extractor that deserializes query strings into some type.

```rust
use axum::{
    extract::Query,
    routing::get,
    Router,
};
use serde::Deserialize;

#[derive(Deserialize)]
struct Pagination {
    page: usize,
    per_page: usize,
}

// This will parse query strings like `?page=2&per_page=30` into `Pagination`
// structs.
async fn list_things(pagination: Query<Pagination>) {
    let pagination: Pagination = pagination.0;

    // ...
}

let app = Router::new().route("/list_things", get(list_things));
```

RawForm：    Extractor that extracts raw form requests.  实现了 FromReqeust

```rust
use axum::{
    extract::RawForm,
    routing::get,
    Router
};

async fn handler(RawForm(form): RawForm) {}

let app = Router::new().route("/", get(handler));
```

RawPathParams：    Extractor that will get captures from the URL without deserializing them.

```rust
pub struct RawPathParams(/* private fields */);
impl<'a> IntoIterator for &'a RawPathParams
type Item = (&'a str, &'a str)
The type of the elements being iterated over.

use axum::{
    extract::RawPathParams,
    routing::get,
    Router,
};

async fn users_teams_show(params: RawPathParams) {
    for (key, value) in &params {
        println!("{key:?} = {value:?}");
    }
}

let app = Router::new().route("/users/:user_id/team/:team_id", get(users_teams_show));
```

RawQuery：    Extractor that extracts the raw query string, without parsing it.

```rust
pub struct RawQuery(pub Option<String>);

// Extractor that extracts the raw query string, without parsing it.
// Example

use axum::{
    extract::RawQuery,
    routing::get,
    Router,
};
use futures_util::StreamExt;

async fn handler(RawQuery(query): RawQuery) {
    // ...
}

let app = Router::new().route("/users", get(handler));
```

State： Extractor for state. As state is `global within a Router` you can’t directly get a mutable
reference to the state. The most basic solution is to use an Arc&lt;Mutex&lt;_&gt;&gt;. Which kind of mutex you
need depends on your use case. See the tokio docs for more details.

```rust
use axum::{Router, routing::get, extract::State};

// the application state
//
// here you can put configuration, database connection pools, or whatever
// state you need
//
// see "When states need to implement `Clone`" for more details on why we need
// `#[derive(Clone)]` here.
#[derive(Clone)]
struct AppState {}

let state = AppState {};

// create a `Router` that holds our state
let app = Router::new()
    .route("/", get(handler))
    // provide the state so the router can access it
    .with_state(state);

async fn handler(
    // access the state via the `State` extractor
    // extracting a state of the wrong type results in a compile error
    State(state): State<AppState>,
) {
    // use `state`...
}
```

substate：State only allows a single state type but you can use FromRef to extract “substates”:

```rust
use axum::{Router, routing::get, extract::{State, FromRef}};

// the application state
#[derive(Clone)]
struct AppState {
    // that holds some api specific state
    api_state: ApiState,
}

// the api specific state
#[derive(Clone)]
struct ApiState {}

// support converting an `AppState` in an `ApiState`
impl FromRef<AppState> for ApiState {
    fn from_ref(app_state: &AppState) -> ApiState {
        app_state.api_state.clone()
    }
}

let state = AppState {
    api_state: ApiState {},
};

let app = Router::new()
    .route("/", get(handler))
    .route("/api/users", get(api_users))
    .with_state(state);

async fn api_users(
    // access the api specific state
    State(api_state): State<ApiState>,
) {
}

async fn handler(
    // we can still access to top level state
    State(state): State<AppState>,
) {
}
```

WebSocketUpgrade：    Extractor for establishing WebSocket connections. 实现了  FromRequestParts

```rust
use axum::{
    extract::ws::{WebSocketUpgrade, WebSocket},
    routing::get,
    response::{IntoResponse, Response},
    Router,
};

let app = Router::new().route("/ws", get(handler));

async fn handler(ws: WebSocketUpgrade) -> Response {
    ws.protocols(["graphql-ws", "graphql-transport-ws"])
        .on_upgrade(|socket| async {
            // ...
        })
}


use axum::{
    extract::ws::{WebSocketUpgrade, WebSocket},
    routing::get,
    response::{IntoResponse, Response},
    Router,
};

let app = Router::new().route("/ws", get(handler));

async fn handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        let msg = if let Ok(msg) = msg {
            msg
        } else {
            // client disconnected
            return;
        };

        if socket.send(msg).await.is_err() {
            // client disconnected
            return;
        }
    }
}


// If you need to read and write concurrently from a WebSocket you can use StreamExt::split:
use axum::{Error, extract::ws::{WebSocket, Message}};
use futures_util::{sink::SinkExt, stream::{StreamExt, SplitSink, SplitStream}};

async fn handle_socket(mut socket: WebSocket) {
    let (mut sender, mut receiver) = socket.split();

    tokio::spawn(write(sender));
    tokio::spawn(read(receiver));
}

async fn read(receiver: SplitStream<WebSocket>) {
    // ...
}

async fn write(sender: SplitSink<WebSocket, Message>) {
    // ...
}
```


## <span class="section-num">7</span> layer {#layer}

可以在 Router、MethodRouter 和 Handler 三个层次上，通过 layer() 方法来添加中间件 middleware。

-   或者使用 `tower::ServiceBuilder()`  来创建 Layer，它支持一次添加多个 layer。

middleware 由 tower create 的 Layer trait 定义，所以 axum 可以复用大量 tower/tower_http crate 提供的大量 middleware。

Router 类型的 layer() :

-   需要传入的 layer 类型是 Layer&lt;Route&gt;, 而 Route 实现了 Service&lt;Request&gt;, 所以不是任意的 tower layer
    都可以使用. (例如 tower crate 提供了各种 module 不能使用).
-   符合该 Layer&lt;Route&gt; 签名的 Layer 通常来自于 `tower_http` crate;

<!--listend-->

```rust
pub fn layer<L>(self, layer: L) -> Router<S>
where
    L: Layer<Route> + Clone + Send + 'static,
    L::Service: Service<Request> + Clone + Send + 'static,
    <L::Service as Service<Request>>::Response: IntoResponse + 'static,
    <L::Service as Service<Request>>::Error: Into<Infallible> + 'static,
    <L::Service as Service<Request>>::Future: Send + 'static,
```

建议使用 `tower::ServiceBuilder`  来一次创建多个 layer，而不是多次调用 layer() 方法：

```rust
use axum::{routing::get, Router};
async fn handler() {}
let app = Router::new()
    .route("/", get(handler))
    .layer(layer_one) // 添加中间件，在 handler 执行前执行
    .layer(layer_two)
    .layer(layer_three);

// 或者
use tower::ServiceBuilder;
use axum::{routing::get, Router};
async fn handler() {}
let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new() // 使用 ServiceBuilder 来组合多个 layer 中间件
            .layer(layer_one)
            .layer(layer_two)
            .layer(layer_three),
    );

// 添加后的 layer 按相反的顺序被调用
        requests
           |
           v
+----- layer_three -----+
| +---- layer_two ----+ |
| | +-- layer_one --+ | |
| | |               | | |
| | |    handler    | | |
| | |               | | |
| | +-- layer_one --+ | |
| +---- layer_two ----+ |
+----- layer_three -----+
           |
           v
        responses
```

创建中间件 Layer 的方式：

1.  `axum::middleware::from_fn`

Use axum::middleware::from_fn to write your middleware when:

You’re not comfortable with implementing your own futures and would rather use the familiar
async/await syntax.  You don’t intend to publish your middleware as a crate for others to
use. Middleware written like this are only compatible with axum.

1.  `axum::middleware::from_extractor`

Use axum::middleware::from_extractor to write your middleware when:

You have a type that you sometimes want to use as an extractor and sometimes as a middleware. If
you only need your type as a middleware prefer middleware::from_fn.

1.  tower’s combinators

tower has several utility combinators that can be used to perform simple modifications to requests
or responses. The most commonly used ones are

ServiceBuilder::map_request
ServiceBuilder::map_response
ServiceBuilder::then
ServiceBuilder::and_then

You should use these when

You want to perform a small ad hoc operation, such as adding a header.
You don’t intend to publish your middleware as a crate for others to use.

1.  `tower::Service and Pin<Box<dyn Future>>`

For maximum control (and a more low level API) you can write you own middleware by implementing
tower::Service:

Use tower::Service with Pin&lt;Box&lt;dyn Future&gt;&gt; to write your middleware when:

Your middleware needs to be configurable for example via builder methods on your tower::Layer
such as tower_http::trace::TraceLayer.  You do intend to publish your middleware as a crate for
others to use.  You’re not comfortable with implementing your own futures.

```rust
use axum::{
    response::Response,
    body::Body,
    extract::Request,
};
use futures_util::future::BoxFuture;
use tower::{Service, Layer};
use std::task::{Context, Poll};

#[derive(Clone)]
struct MyLayer;

impl<S> Layer<S> for MyLayer {
    type Service = MyMiddleware<S>;

    fn layer(&self, inner: S) -> Self::Service {
        MyMiddleware { inner }
    }
}

#[derive(Clone)]
struct MyMiddleware<S> {
    inner: S,
}

// Service 必须是 Request 类型, 才能在 Routing.layer() 中使用.
impl<S> Service<Request> for MyMiddleware<S>
where
    S: Service<Request, Response = Response> + Send + 'static,
    S::Future: Send + 'static,
{
    type Response = S::Response;
    type Error = S::Error;
    // `BoxFuture` is a type alias for `Pin<Box<dyn Future + Send + 'a>>`
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, request: Request) -> Self::Future {
        let future = self.inner.call(request);
        Box::pin(async move {
            let response: Response = future.await?;
            Ok(response)
        })
    }
}
```

可以使用 request extensions 来在 middleware 和 handlers 之间传递 State：

```rust
use axum::{
    Router,
    http::StatusCode,
    routing::get,
    response::{IntoResponse, Response},
    middleware::{self, Next},
    extract::{Request, Extension},
};

#[derive(Clone)]
struct CurrentUser { /* ... */ }

async fn auth(mut req: Request, next: Next) -> Result<Response, StatusCode> {
    let auth_header = req.headers()
        .get(http::header::AUTHORIZATION)
        .and_then(|header| header.to_str().ok());

    let auth_header = if let Some(auth_header) = auth_header {
        auth_header
    } else {
        return Err(StatusCode::UNAUTHORIZED);
    };

    if let Some(current_user) = authorize_current_user(auth_header).await {
        // insert the current user into =a request extension= so the handler can
        // extract it
        req.extensions_mut().insert(current_user);
        Ok(next.run(req).await)
    } else {
        Err(StatusCode::UNAUTHORIZED)
    }
}

async fn authorize_current_user(auth_token: &str) -> Option<CurrentUser> {
    // ...
}

async fn handler(
    // extract the current user, set by the middleware
    Extension(current_user): Extension<CurrentUser>,
) {
    // ...
}

let app = Router::new()
    .route("/", get(handler))
    .route_layer(middleware::from_fn(auth));
```


## <span class="section-num">8</span> error_handling {#error-handling}

Handler aysnc fn 的响应是 IntoResponse 对象，而非包含 Error 的 Result 对象，所以 axum 不会返回 Error，如果要返回出错信息，也是自定义 Error type 并实现 IntoResponse；

```rust
use axum::http::StatusCode;

// String/StatusCode 都实现了 IntoResponse，所以 Result 也实现了 IntoResponse，所以最终会被转换为
// Response 返回给 client
async fn handler() -> Result<String, StatusCode> {
    // ...
}
```

如果 Handler 要给 client 返回实际的自定义错误信息，需要自定义错误类型实现 IntoResponse，例如：

```rust
// https://github.com/tokio-rs/axum/blob/main/examples/anyhow-error-response/src/main.rs

//! Run with
//!
//! ```not_rust
//! cargo run -p example-anyhow-error-response
//! ```

use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(handler));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000") .await .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}


// Handler 返回的 Result 会被 IntoResponse 转换为  Response 返回给 client
async fn handler() -> Result<(), AppError> {
    try_thing()?;
    Ok(())
}

fn try_thing() -> Result<(), anyhow::Error> {
    anyhow::bail!("it failed!")
}

// Make our own error that wraps `anyhow::Error`.
struct AppError(anyhow::Error);

// Tell axum how to convert `AppError` into a response.
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
	// axum 为 (StatusCode, String) 实现了 IntoRespose
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}

// This enables using `?` on functions that return `Result<_, anyhow::Error>` to turn them into
// `Result<_, AppError>`. That way you don't need to do that manually.
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
```

Router.route_service() 使用 Service 作为请求的 Handler，该 Service 的约束是 Service&lt;Request, Error =
Infallible&gt; + Clone + Send + 'static, 这里的 Error = Infallible，表示不能传入任何实际的 Error，而一般通过 tower::service_fn(fn) 创建的 Service 约束是 FnMut(Request) -&gt; Future&lt;Output = Result&lt;R, E&gt;&gt;，所以包含具体的 Error，两者不匹配。

为了能在 Router.route_service()中使用可以返回 Result&lt;R, E&gt; 的 Service，axum 提供了
axum::error_handling::HandleError 两将它们转换为 Response。

HandleError 用于将一个返回 Result Error 的 Service 转换为 Service&lt;Request&gt;:

-   S: S 实现了 Service&lt;Request&lt;B&gt;&gt;,返回 IntoResponse 和 Error
-   F: 是一个闭包 FnOnce($($ty),\*, S::Error) -&gt; Fut + Clone + Send + 'static,S::Error, 输入可以是一个或多个 extractor + 最后一个 Error, 返回一个 IntoResponse

<!--listend-->

```rust
pub struct HandleError<S, F, T> { /* private fields */ }

// 方法
impl<S, F, T> HandleError<S, F, T>
// Create a new HandleError.
pub fn new(inner: S, f: F) -> Self

//  HandlerError 实现了 Service<Request>
// S 实现了 Service<Request<B>>,返回 IntoResponse 和 Error
// F 是一个闭包, 输入是 S::Error, 返回一个 IntoResponse
impl<S, F, B, Fut, Res> Service<Request<B>> for HandleError<S, F, ()>
where
    S: Service<Request<B>> + Clone + Send + 'static,
    S::Response: IntoResponse + Send,
    S::Error: Send,
    S::Future: Send,
    F: FnOnce(S::Error) -> Fut + Clone + Send + 'static,
    Fut: Future<Output = Res> + Send,
    Res: IntoResponse,
    B: Send + 'static,

macro_rules! impl_service {
    ( $($ty:ident),* $(,)? ) => {
        impl<S, F, B, Res, Fut, $($ty,)*> Service<Request<B>>
            for HandleError<S, F, ($($ty,)*)>
        where
            S: Service<Request<B>> + Clone + Send + 'static,
            S::Response: IntoResponse + Send,
            S::Error: Send,
            S::Future: Send,
            F: FnOnce($($ty),*, S::Error) -> Fut + Clone + Send + 'static,
            Fut: Future<Output = Res> + Send,
            Res: IntoResponse,
            $( $ty: FromRequestParts<()> + Send,)*
            B: Send + 'static,
        {
            //...
        }
    }
}
```

示例:

```rust
use axum::{
    Router,
    body::Body,
    http::{Request, Response, StatusCode},
    error_handling::HandleError,
};

async fn thing_that_might_fail() -> Result<(), anyhow::Error> {
    // ...
}

// 使用 service_fn 将异步函数转换为实现 Service 的 ServiceFn 对象, 而 ServiceFn 在实现 Service 时对
// 于fn 的定义是 FnMut(Request) -> Future<Output = Result<R, E>>,所以 fn 可以返回包含 Error 的
// Result

// this service might fail with `anyhow::Error`
let some_fallible_service = tower::service_fn(|_req| async {
    thing_that_might_fail().await?;
    Ok::<_, anyhow::Error>(Response::new(Body::empty()))
});


// route_service() 输入的 Service 的约束是：Service<Request, Error = Infallible> + Clone + Send +
// 'static, 这里的 Error = Infallible，于 tower::service_fn() 返回的 Error = xxx 不匹配，所以需要
// HandleError::new() 来进行转换。
let app = Router::new().route_service(
    "/",
    // we cannot route to `some_fallible_service` directly since it might fail.  we have to use
    // `handle_error` which converts its errors into responses and changes its error type from
    // `anyhow::Error` to `Infallible`.
    HandleError::new(some_fallible_service, handle_anyhow_error),
);

// handle errors by converting them into something that implements `IntoResponse`
async fn handle_anyhow_error(err: anyhow::Error) -> (StatusCode, String) {
    (
        StatusCode::INTERNAL_SERVER_ERROR,
        format!("Something went wrong: {err}"),
    )
}
```

对于 layer middleware，也存在和 Service 类似的情况，axum::error_handling::HandleErrorLayer 提供了能处理 middleware Error 的转换能力。HandleErrorLayer 实现了 Layer&lt;S&gt; 和 Service, 它内部使用
HandlerError 来将传入的返回 Error 的 Service转换为axum 可以使用的 Service&lt;Request, Error =
Infallible&gt;

-   new(f) 输入是闭包函数 FnOnce($($ty),\*, S::Error) -&gt; Fut + Clone + Send + 'static, 输入是 0 或多个
    extractor + 最后一个 Error, 返回一个 IntoResponse 对象.

<!--listend-->

```rust
pub struct HandleErrorLayer<F, T> { /* private fields */ }

impl<F, T> HandleErrorLayer<F, T>
pub fn new(f: F) -> Self // f 闭包函数的输入是 Error,返回一个 IntoResponse 对象
// Create a new HandleErrorLayer.

impl<S, F, T> Layer<S> for HandleErrorLayer<F, T>
where
    F: Clone,
{
    type Service = HandleError<S, F, T>;

    fn layer(&self, inner: S) -> Self::Service {
        HandleError::new(inner, self.f.clone()) // self.f 传给 HandlerError::new(), 所以需要满足它对 F
    }
}

impl<S, F, B, Fut, Res> Service<Request<B>> for HandleError<S, F, ()>
where
    S: Service<Request<B>> + Clone + Send + 'static,
    S::Response: IntoResponse + Send,
    S::Error: Send,
    S::Future: Send,
    F: FnOnce(S::Error) -> Fut + Clone + Send + 'static,
    Fut: Future<Output = Res> + Send,
    Res: IntoResponse,
    B: Send + 'static,
{
    //...
}
```

示例:

```rust
use axum::{
    Router,
    BoxError,
    routing::get,
    http::StatusCode,
    error_handling::HandleErrorLayer,
};
use std::time::Duration;
use tower::ServiceBuilder;

let app = Router::new()
    .route("/", get(|| async {}))
    .layer(
        ServiceBuilder::new()
            // `timeout` will produce an error if the handler takes
            // too long so we must handle those
	        // new 传入的闭包函数的输入是 Error,返回一个 IntoResponse 对象
            .layer(HandleErrorLayer::new(handle_timeout_error))
            .timeout(Duration::from_secs(30))
    );

// 闭包函数的输入是 Error,返回一个 IntoResponse 对象
async fn handle_timeout_error(err: BoxError) -> (StatusCode, String) {
    if err.is::<tower::timeout::error::Elapsed>() {
        (
            StatusCode::REQUEST_TIMEOUT,
            "Request took too long".to_string(),
        )
    } else {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Unhandled internal error: {err}"),
        )
    }
}
```

HandleErrorLayer also supports running extractors:

```rust
use axum::{
    Router,
    BoxError,
    routing::get,
    http::{StatusCode, Method, Uri},
    error_handling::HandleErrorLayer,
};
use std::time::Duration;
use tower::ServiceBuilder;

let app = Router::new()
    .route("/", get(|| async {}))
    .layer(
        ServiceBuilder::new()
            // `timeout` will produce an error if the handler takes
            // too long so we must handle those
            .layer(HandleErrorLayer::new(handle_timeout_error))
            .timeout(Duration::from_secs(30))
    );

async fn handle_timeout_error(
    // `Method` and `Uri` are extractors so they can be used here
    method: Method,
    uri: Uri,
    // the last argument must be the error itself
    err: BoxError,
) -> (StatusCode, String) {
    (
        StatusCode::INTERNAL_SERVER_ERROR,
        format!("`{method} {uri}` failed with {err}"),
    )
}
```


## <span class="section-num">9</span> tracing log {#tracing-log}

配置项目 Cargo.toml 文件，为 axum 指定 tracing feature，同时引入 tower-http 的 trace feature，后续可以为 axum Router 添加 tracer middleware：

```toml
[dependencies]
axum = { path = "../../axum", features = ["tracing"] }
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.5.0", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

[代码示例](https://github.com/tokio-rs/axum/blob/main/examples/tracing-aka-logging/src/main.rs)：

-   RUST_LOG=debug cargo run 表示将所有 crate 的日子级别设置为 debug。

<!--listend-->

```rust
//! Run with
//!
//! ```not_rust
//! cargo run -p example-tracing-aka-logging
//! ```

use axum::{
    body::Bytes,
    extract::MatchedPath,
    http::{HeaderMap, Request},
    response::{Html, Response},
    routing::get,
    Router,
};
use std::time::Duration;
use tokio::net::TcpListener;
use tower_http::{classify::ServerErrorsFailureClass, trace::TraceLayer};
use tracing::{info_span, Span};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            // 在运行时如果未指定环境变量 RUST_LOG=debug，则这里指定各 crate 的缺省值
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                // axum logs rejections from built-in extractors with the `axum::rejection`
                // target, at `TRACE` level. `axum::rejection=trace` enables showing those events
                "example_tracing_aka_logging=debug,tower_http=debug,axum::rejection=trace".into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // build our application with a route
    let app = Router::new()
        .route("/", get(handler))
        // `TraceLayer` is provided by tower-http so you have to add that as a dependency.
        // It provides good defaults but is also very customizable.
        //
        // See https://docs.rs/tower-http/0.1.1/tower_http/trace/index.html for more details.
        //
        // If you want to customize the behavior using closures here is how.
        .layer(
            TraceLayer::new_for_http() // 这里是关键，只有加了 TraceLayer 后才会打印 HTTP 日志
                .make_span_with(|request: &Request<_>| {
                    // Log the matched route's path (with placeholders not filled in).
                    // Use request.uri() or OriginalUri if you want the real path.
                    let matched_path = request
                        .extensions()
                        .get::<MatchedPath>()
                        .map(MatchedPath::as_str);

                    info_span!(
                        "http_request",
                        method = ?request.method(),
                        matched_path,
                        some_other_field = tracing::field::Empty,
                    )
                })
                .on_request(|_request: &Request<_>, _span: &Span| {
                    // You can use `_span.record("some_other_field", value)` in one of these
                    // closures to attach a value to the initially empty field in the info_span
                    // created above.
                })
                .on_response(|_response: &Response, _latency: Duration, _span: &Span| {
                    // ...
                })
                .on_body_chunk(|_chunk: &Bytes, _latency: Duration, _span: &Span| {
                    // ...
                })
                .on_eos(
                    |_trailers: Option<&HeaderMap>, _stream_duration: Duration, _span: &Span| {
                        // ...
                    },
                )
                .on_failure(
                    |_error: ServerErrorsFailureClass, _latency: Duration, _span: &Span| {
                        // ...
                    },
                ),
        );

    // run it
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```
