# Axum

`axum` is a *web application framework* that focuses on ergonomics and modularity. Featured with:

- **Route requests to handlers using extractors.**
- Declaratively **parse** requests using **extractors**.
- Simple and predictable error handling model.
- Generate responses with minimal boilerplate.
- Take full advantage of the `tower` and `tower-http` ecosystem of middleware, services, and utilities.
- Builtin `tokio` support.

### Compatibility

axum is design to work with `tokio` and `hyper`.

## Routing

**`Router` is used to setup which paths goes to which services.**  
The `Router` type for composing handlers and services.

```Rust
use axum::{Router, routing::get};

// Router
let app = Router::new()
    .route("/", get(root))
    .route("/foo", get(get_foo).post(post_foo))
    .route("/foo/bar", get(foo_bar));

// Which calls the following handlers (async functions)
async fn root() {}
async fn get_foo() {}
async fn post_foo() {}
async fn foo_bar() {}
```

`impl`:

- `pub fn new() -> Self`  
Create a new `Router`. For requests not routed, a `404 Not Found` will response.
- `pub fn route(self, path: &str, method_router: MethodRouter<S, B>) -> Self`  
Add another route to the router.  
`path` is a string of path segments separated by `/`. Each segment can be either *static*, a *capture*, or a *wildcard*.
    - Static paths: e.g. `/`, `/foo`, `/users/wayne`. If the incoming request matches the path exactly the corresponding service will be called.
    - Captures: e.g. `/:key`, `/users/:id`, `/users/:id/tweets`. Paths can contain segments like `/:key` which matches any single segment and will store the value captured at `key`.
    - Wildcards: e.g. `/*key`, `/assets/*path`, `/:id/:repo/*tree`. The leading slash is not included, i.e. for the route `/foo/*rest` and the path `/foo/bar/baz` the value of `rest` is `bar/baz`.

    `method_router` is the `MethodRouter` that should receive the request if the path matches `path`. `method_router` will commonlybe a handler wrapped in a method router like `get`.  
    To accept multiple methods for the same route, these methods can be chained with `.`.

- `pub fn route_service<T>(self, path: &str, service: T) -> Self`  
Add another route to the router that calls a `Service`.
- `pub fn nest(self, path: &str, router: Router<S, B>) -> Self`  
Nest a `Router` at some path. This allows to break application into smaller pieces and compose them together.
    > Note that nested routes will not see the original request URI but instead have the matched prefix stripped. This is necessary for services like static file serving to work. Use **`OriginalUrl`** if the original request URI is needed.
- `pub fn fallback_service<T>(self, svc: T) -> Self`  
Add a fallback `Service` to the router.
- `pub fn with_state<S2>(self, state: S) -> Router<S2, B>`  
Provide the state for the router. The state struct has to derive `Clone`.

## Handlers

- Async functions that can be used to handle requests.
- Accepts zero or more "extractors" as arguments and returns thing that can be converted into a response.
- Where the logic lives.
- Building block of axum applications.

```Rust
use axum::{body::Bytes, http::StatusCode};

async fn unit_handler() {}

async fn string_handler() -> String {
    "Hello, World!".to_string()
}

async fn echo(body: Bytes) -> Result<String, StatusCode> {
    if let Ok(string) = String::from_utf8(body.to_vec()) {
        Ok(string)
    } else {
        Err(StatusCode::BAD_REQUEST)
    }
}
```

## Extractors

A type that implements `FromRequest` or `FromRequestParts`.

For example, `Json` is an extractor that consumes the request body and **deserializes** it as JSON into some target type:

```Rust
use axum::{
    extract::Json,
    routing::post,
    handler::Hander,
    Router,
};
use serde::Deserialize;

#[derive(Deserialize)]
struct CreateUser {
    email: String,
    password: String,
}

async fn create_user(Json(payload): Json<CreateUser>) {
    // ...
}

let app = Router::new().route("/users", post(create_user));
```

## Responses

Anything that implements `IntoResponse` can be returned from handlers. axum provides implementations for common types:

```Rust
use axum:: {
    Json,
    response::{Html, IntoResponse},
    http::{StatusCode, Uri, header::{self, HeaderMap, HeaderName}},
};

// `()` gives an empty response
async fn empty() {}

// String will get a `text/plain; charset=utf-8` content-type
async fn plain_text(uri: Uri) -> String {
    format!("Hi from {}", usr.path())
}

// Bytes will get a `application/octet-stream` content-type
async fn bytes() -> Vec<u8> {
    vec![1, 2, 3, 4]
}

// `Json` will get a `application/json` content-type and work with anything that implements `serde::Serialize`
async fn json() -> Json<Vec<String>> {
    Json(vec!["foo".to_owned(), "bar".to_owned()])
}

// `Html` will get a `text/html` content-type
async fn html() -> Html<&'static str> {
    Html("<p>Hello, world!</p>")
}

// `StatusCode` gives an empty response with that status code
asunc fn status() -> StatusCode {
    StatusCode::NOT_FOUND
}

// `HeaderMap` gives an empty response with some headers
async fn headers() -> HeaderMap {
    let mut header = HeaderMap::new();
    headers.insert(header::SERVER, "axum".parse().unwrap());
    headers
}

// An array of tuples also gives headers
async fn array_headers() -> [(HeaderName, &'static str); 2] {
    [
        (header::SERVER, "axum"),
        (header::CONTENT_TYPE, "text/plain")
    ]
}

// Use `impl IntoResponse` to avoid writing the whole type
async fn impl_trait() -> impl IntoResponse {
    [
        (header::SERVER, "axum"),
        (header::CONTENT_TYPE, "text/plain")
    ]
}
```

**Additionally, tuples could be returned to build more complex responses from individual parts.

```Rust
use axum::{
    Json,
    response::IntoResponse,
    http::{StatusCode, HeaderMap, Uri, header},
    extract::Extension,
}

// `(StatusCode, impl IntoResponse)` will override the status code of the response
async fn with_status(uri: Uri) -> (StatusCode, String) {
    (StatusCode::NOT_FOUND, format!("Not Found: {}", uri.path()))
}

// Use `impl IntoResponse` to avoid having to type the while type
async fn impl_trait(uri: Uri) -> impl IntoResponse {
    (StatusCode::NOT_FOUND, format!("Not Found: {}", uri.path()))
}

// `(Header, imple IntoResponse)` to add additional header
async fn with_headers() -> impl IntoResponse {
    let mut headers = HeaderMap::new();
    headers.insert(header::CONTENT_TYPE, "text/plain".parse().unwrap());
    (headers, "foo")
}

// Or an array of tuples to more easily build the headers
async fn with_array_headers() -> impl IntoResponse {
    ([(header:CONTENT_TYPE, "text/plain")], "foo")
}

// Use string keys for custom headers
async fn with_array_headers_custom() -> impl IntoResponse {
    ([("x-custom", "custom")], "foo")
}

// `(StatusCode, headers, impl IntoResponse)` to set status and add headers
// `headers` can be either a `HeaderMap` or an array of tuples
async fn with_status_and_array_headers() -> impl IntoResponse {
    (
        StatusCode::NOT_FOUND,
        [(header::CONTENT_TYPE), "text/plain")],
        "foo",
    )
}

// `(Extension<_>, impl IntoResponse)` to set response extensions
async fn with_status_extensions() -> impl IntoResponse {
    (
        Extension(Foo("foo")),
        "foo",
    )
}

struct Foo(&'static str);

// Or mix and match all the things
async fn all_the_things(uri: Uri) -> impl IntoResponse {
    let mut header_map = HeaderMap::new();
    if uri.path() == "/" {
        header_map.insert(header::SERVER, "axum".parse().unwrap());
    }

    (
        // Set status code
        StatusCode::NOT_FOUND,
        // Headers with an array
        [("x-custom", "custom")],
        // Some extensions
        Extension(Foo("foo")),
        Extension(Foo("bar")),
        // More headers, built dynamically
        header_map,
        // And finally the body
        "foo",
    )
}
```

> In general, tuples can be returned are like:  
> - `(StatusCode, impl IntoResponse)`
> - `(Parts, impl IntoResponse)`
> - `(Response<()>, impl IntoResponse)`
> - `(T1, .., Tn, impl IntoResponse)` where `T1` to `Tn` all implement `IntoResponse`
> - `(StatusCode, T1, .., Tn, impl IntoResponse)` where `T1` to `Tn` all implement `IntoResponse`
> - `(Parts, T1, .., Tn, impl IntoResponse)` where `T1` to `Tn` all implement `IntoResponse`
> - `(Response<()>, T1, .., Tn, impl IntoResponse)` where `T1` to `Tn` all implement `IntoResponse`

This means the status or body cannot be accidentally overrided as `IntoResponse` only allows setting headers and extensions.

Use `Response` for more low level control:

```Rust
use axum::{
    Json,
    response::{IntoResponse, Response},
    body::{Full, Bytes},
    http::StatusCode,
}

async fn response() -> Response<Full<Bytes>> {
    Response::builder()
        .status(StatusCode::NOT_FOUND)
        .header("x-foo", "custom header")
        .body(Full::from("not found"))
        .unwrap()
}
```
