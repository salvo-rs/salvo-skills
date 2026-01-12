---
name: salvo-basic-app
description: Create basic Salvo web applications with handlers, routers, and server setup. Use when starting a new Salvo project or adding basic HTTP endpoints.
---

# Salvo Basic Application Setup

This skill helps create basic Salvo web applications with proper structure and best practices.

## Core Concepts

### Handler
Handlers process HTTP requests. Use the `#[handler]` macro on async functions:

```rust
use salvo::prelude::*;

#[handler]
async fn hello() -> &'static str {
    "Hello World"
}

#[handler]
async fn greet(req: &mut Request) -> String {
    let name = req.query::<String>("name").unwrap_or("World".to_string());
    format!("Hello, {}!", name)
}
```

Handler parameters can be in any order and are optional:
- `req: &mut Request` - HTTP request
- `res: &mut Response` - HTTP response
- `depot: &mut Depot` - Request-scoped storage
- `ctrl: &mut FlowCtrl` - Flow control for middleware

### Router
Routers define URL paths and attach handlers:

```rust
use salvo::prelude::*;

let router = Router::new()
    .get(hello)
    .push(Router::with_path("greet").get(greet));
```

### Server Setup
Basic server configuration:

```rust
use salvo::prelude::*;

#[tokio::main]
async fn main() {
    let router = Router::new().get(hello);
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Common Patterns

### Returning Different Types
Handlers can return any type implementing `Writer`:

```rust
#[handler]
async fn json_response() -> Json<serde_json::Value> {
    Json(serde_json::json!({"status": "ok"}))
}

#[handler]
async fn text_response() -> Text<&'static str> {
    Text("Plain text")
}

#[handler]
async fn status_response() -> StatusCode {
    StatusCode::NO_CONTENT
}
```

### Error Handling
Return `Result<T, E>` where both implement `Writer`:

```rust
#[handler]
async fn may_fail() -> Result<Json<Data>, StatusError> {
    let data = fetch_data().await?;
    Ok(Json(data))
}
```

## Dependencies

Add to `Cargo.toml`:

```toml
[dependencies]
salvo = "0.88"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

## Best Practices

1. Use `#[handler]` macro for all handlers
2. Keep handlers focused on single responsibility
3. Use appropriate return types (Json, Text, StatusCode)
4. Handle errors with Result types
5. Use `TcpListener` for basic HTTP servers
