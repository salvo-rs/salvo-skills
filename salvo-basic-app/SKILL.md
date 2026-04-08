---
name: salvo-basic-app
description: Create basic Salvo web applications with handlers, routers, and server setup. Use when starting a new Salvo project or adding basic HTTP endpoints.
version: 0.89.3
tags: [core, getting-started, handler, router]
---

# Salvo Basic Application Setup

## Dependencies

```toml
[dependencies]
salvo = "0.89.3"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

## Minimal server

```rust
use salvo::prelude::*;

#[handler]
async fn hello() -> &'static str {
    "Hello World"
}

#[tokio::main]
async fn main() {
    let router = Router::new().get(hello);
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Handlers

The `#[handler]` macro turns an async function into a `Handler`. All parameters are optional and order-independent:

- `req: &mut Request` — HTTP request
- `res: &mut Response` — HTTP response
- `depot: &mut Depot` — request-scoped storage
- `ctrl: &mut FlowCtrl` — middleware flow control

A handler can return any type implementing `Writer` or `Scribe` (including `()`, `&'static str`, `String`, `Json<T>`, `StatusCode`, `Result<T, E>`), or write directly via `res.render(...)`.

## Response types

```rust
use salvo::prelude::*;
use salvo::writing::Text;
use serde::Serialize;

#[derive(Serialize)]
struct User { name: String, age: u8 }

#[handler]
async fn json_body() -> Json<User> {
    Json(User { name: "Alice".into(), age: 30 })
}

#[handler]
async fn html_body() -> Text<&'static str> {
    Text::Html("<h1>Hello</h1>")
}

#[handler]
async fn no_content() -> StatusCode {
    StatusCode::NO_CONTENT
}

#[handler]
async fn go_elsewhere(res: &mut Response) {
    res.render(Redirect::found("https://example.com"));
}
```

HTML rendering uses `Text::Html(...)` (also `Text::Plain`, `Text::Json`, `Text::Xml`, `Text::Css`). There is no `salvo::writing::Html` type.

## Error handling

Return `Result<T, E>` where `E: Writer`. `StatusError` is the canonical error type:

```rust
#[handler]
async fn may_fail() -> Result<Json<User>, StatusError> {
    let user = fetch_user().await
        .map_err(|e| StatusError::internal_server_error().cause(e))?;
    Ok(Json(user))
}
```

Available constructors: `bad_request()`, `unauthorized()`, `forbidden()`, `not_found()`, `internal_server_error()`, etc. Chain `.brief(...)`, `.detail(...)`, `.cause(...)` for context.

## Request access

```rust
#[handler]
async fn inspect(req: &mut Request) -> String {
    let method = req.method();
    let path = req.uri().path();
    let ct: Option<String> = req.header("Content-Type");
    let name: Option<String> = req.query("name");      // ?name=...
    let id: Option<i64> = req.param("id");             // /{id}
    let body: MyData = req.parse_json().await.unwrap(); // JSON body
    format!("{method} {path}")
}
```

`header`, `query`, and `param` are generic over `T: Deserialize` and return `Option<T>`. `parse_json` returns `Result`.

## Response mutation

```rust
#[handler]
async fn created(res: &mut Response) {
    res.status_code(StatusCode::CREATED);
    res.headers_mut().insert("X-Custom", "value".parse().unwrap());
    res.render(Json(serde_json::json!({"ok": true})));
}
```

## Related Skills

- **salvo-routing**: Advanced routing configuration and path parameters
- **salvo-middleware**: Add middleware for logging, auth, and CORS
- **salvo-error-handling**: Graceful error handling patterns
