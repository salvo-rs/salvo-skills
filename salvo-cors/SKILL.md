---
name: salvo-cors
description: Configure Cross-Origin Resource Sharing (CORS) and security headers. Use for APIs accessed from browsers on different domains.
version: 0.89.3
tags: [security, cors, cross-origin, headers]
---

# Salvo CORS Configuration

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["cors"] }
```

`Cors` is a builder; terminate with `.into_handler()` to get a `CorsHandler` you can `hoop` onto a router.

## Basic Configuration

```rust
use salvo::cors::Cors;
use salvo::http::Method;
use salvo::prelude::*;

#[tokio::main]
async fn main() {
    let cors = Cors::new()
        .allow_origin("https://example.com")
        .allow_methods(vec![Method::GET, Method::POST, Method::PUT, Method::DELETE])
        .allow_headers(vec!["content-type", "authorization"])
        .into_handler();

    let router = Router::new()
        .hoop(cors)
        .push(Router::with_path("api").get(api_handler));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}

#[handler]
async fn api_handler() -> &'static str { "Hello from API" }
```

## Permissive Presets

```rust
use salvo::cors::{Any, Cors};

// Any origin, methods, headers, exposed headers. Does not allow credentials.
let cors = Cors::permissive().into_handler();

// Mirror-request variant that ALSO allows credentials. Prints a warning.
// Only safe for tightly-controlled deployments.
let cors = Cors::very_permissive().into_handler();

// Manual wildcard
let cors = Cors::new()
    .allow_origin(Any)
    .allow_methods(Any)
    .allow_headers(Any)
    .into_handler();
```

## Production Configuration

```rust
use salvo::cors::Cors;
use salvo::http::Method;

let cors = Cors::new()
    .allow_origin(["https://app.example.com", "https://admin.example.com"])
    .allow_methods(vec![Method::GET, Method::POST, Method::PUT, Method::DELETE])
    .allow_headers(vec!["authorization", "content-type", "x-requested-with"])
    .expose_headers(vec!["x-request-id"])
    .allow_credentials(true)
    .max_age(3600) // seconds, or pass a Duration
    .into_handler();
```

## Dynamic Origins

Use `AllowOrigin::dynamic` — the closure signature is `Fn(Option<&HeaderValue>, &Request, &Depot) -> Option<HeaderValue>`. Return `Some(origin)` to accept, `None` to reject.

```rust
use salvo::cors::{AllowOrigin, Cors};
use salvo::http::HeaderValue;

let cors = Cors::new()
    .allow_origin(AllowOrigin::dynamic(|origin, _req, _depot| {
        let o = origin?.to_str().ok()?;
        if o.ends_with(".example.com") || o == "https://example.com" {
            HeaderValue::from_str(o).ok()
        } else {
            None
        }
    }))
    .allow_methods(vec!["GET", "POST"])
    .allow_credentials(true)
    .into_handler();
```

`AllowOrigin::mirror_request()` reflects any origin header back (credentials-safe alternative to `*`).

## Scoped CORS

```rust
let cors = Cors::new()
    .allow_origin("https://app.example.com")
    .allow_methods(vec!["GET", "POST"])
    .into_handler();

let router = Router::new()
    .push(
        Router::with_path("api")
            .hoop(cors)
            .push(Router::with_path("users").get(list_users))
            .push(Router::with_path("posts").get(list_posts)),
    )
    .push(Router::with_path("health").get(health_check));
```

## Security Headers

`salvo-cors` only handles CORS. Add other security headers with a small custom handler:

```rust
use salvo::prelude::*;

#[handler]
async fn security_headers(res: &mut Response) {
    let h = res.headers_mut();
    h.insert("content-security-policy", "default-src 'self'".parse().unwrap());
    h.insert("strict-transport-security", "max-age=31536000; includeSubDomains".parse().unwrap());
    h.insert("x-frame-options", "DENY".parse().unwrap());
    h.insert("x-content-type-options", "nosniff".parse().unwrap());
    h.insert("referrer-policy", "strict-origin-when-cross-origin".parse().unwrap());
}
```

## Builder Options

| Method | Description |
|--------|-------------|
| `allow_origin(impl Into<AllowOrigin>)` | Exact string, list, `Any`, or `AllowOrigin::dynamic`/`dynamic_async`/`mirror_request` |
| `allow_methods(impl Into<AllowMethods>)` | Vec of `Method` or str, or `Any` |
| `allow_headers(impl Into<AllowHeaders>)` | Vec of header names, `Any`, or `AllowHeaders::mirror_request()` |
| `expose_headers(...)` | Response headers readable by the browser |
| `allow_credentials(bool)` | Include cookies / auth headers |
| `max_age(u64 or Duration)` | Cache preflight response |
| `allow_private_network(bool)` | Opt into private network access preflights |
| `vary(...)` | Extra `Vary` headers |
| `into_handler()` | Build the `CorsHandler` (required) |

## Salvo-specific Notes

- `Cors` is a builder; forgetting `.into_handler()` will not compile when passed to `.hoop()`.
- Combining `allow_credentials(true)` with any wildcard (`Any` on origin/methods/headers/expose_headers) panics at `into_handler()` time. Use an explicit list or `mirror_request()` instead.
- `Cors::very_permissive()` intentionally logs a warning on construction — do not ship it.
- Header names passed to `allow_headers` are case-insensitive, but lowercase is conventional.

## Related Skills

- **salvo-csrf**: CSRF protection for cross-origin forms
- **salvo-auth**: CORS for authenticated API endpoints
