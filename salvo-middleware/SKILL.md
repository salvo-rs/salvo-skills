---
name: salvo-middleware
description: Implement middleware for authentication, logging, CORS, and request processing. Use for cross-cutting concerns and request/response modification.
version: 0.89.3
tags: [core, middleware, hoop, flow-ctrl]
---

# Salvo Middleware

In Salvo, middleware is just a `Handler` that calls `ctrl.call_next(...)`. Attach it with `hoop()`; it runs for the current router and all descendants.

## Basic middleware

```rust
use salvo::prelude::*;

#[handler]
async fn logger(
    req: &mut Request,
    depot: &mut Depot,
    res: &mut Response,
    ctrl: &mut FlowCtrl,
) {
    println!("--> {} {}", req.method(), req.uri().path());
    ctrl.call_next(req, depot, res).await;
    println!("<-- {}", res.status_code().unwrap_or(StatusCode::OK));
}

let router = Router::new().hoop(logger).get(handler);
```

Omit `ctrl.call_next(...)` only at the end of the chain; otherwise downstream handlers will not run.

## Scoping

`hoop()` on a router applies to that router and its descendants. Compose scopes with `push()`:

```rust
let router = Router::new()
    .hoop(logger)                                       // global
    .push(
        Router::with_path("api")
            .hoop(auth_check)                           // only /api/**
            .hoop(rate_limiter)
            .push(Router::with_path("users").get(list_users))
    )
    .push(Router::with_path("public").get(public_handler));  // no auth
```

## FlowCtrl

- `call_next(req, depot, res).await` — run the next handler
- `skip_rest()` — skip all remaining handlers (the response you set is returned)
- `cease()` — `skip_rest()` plus mark as ceased; subsequent middleware should check `is_ceased()`
- `is_ceased()` — has the flow been explicitly ceased

Salvo also auto-skips the rest of the chain when a handler sets an error (4xx/5xx) or redirect (3xx) status.

## Authentication pattern

```rust
#[handler]
async fn auth_check(
    req: &mut Request,
    depot: &mut Depot,
    res: &mut Response,
    ctrl: &mut FlowCtrl,
) {
    let token = req.header::<String>("Authorization");
    match token.as_deref().and_then(validate_token) {
        Some(user) => {
            depot.insert("user", user);
            ctrl.call_next(req, depot, res).await;
        }
        None => {
            res.render(StatusError::unauthorized());
            ctrl.skip_rest();
        }
    }
}
```

Setting an error status via `StatusError` auto-stops the chain, but calling `skip_rest()` explicitly is clearer.

## Depot for sharing data

```rust
#[handler]
async fn protected(depot: &mut Depot) -> String {
    // Depot::get returns Result<&V, _>, not Option
    let user: &User = depot.get("user").unwrap();
    format!("hello, {}", user.name)
}
```

Depot API:

- `depot.insert(key, value)` — stores by string key
- `depot.get::<V>(key)` — returns `Result<&V, Option<&Box<dyn Any>>>`
- `depot.inject(value)` — stores by type (single value per type)
- `depot.obtain::<V>()` — retrieves by type, returns `Result`

Prefer `inject` / `obtain` when the type itself is the key; use `insert` / `get` when you need multiple values of the same type distinguished by name.

## Early response

```rust
#[handler]
async fn guard(
    req: &mut Request,
    _depot: &mut Depot,
    res: &mut Response,
    ctrl: &mut FlowCtrl,
) {
    if !is_valid(req) {
        res.render(StatusError::bad_request().brief("invalid input"));
        ctrl.skip_rest();
    }
    // fall through to call_next if you want to continue
}
```

## CORS

```rust
use salvo::cors::Cors;
use salvo::http::Method;

let cors = Cors::new()
    .allow_origin("https://example.com")
    .allow_methods(vec![Method::GET, Method::POST])
    .allow_headers(vec!["Content-Type", "Authorization"])
    .into_handler();

let router = Router::new().hoop(cors).get(handler);
```

`Cors` is a builder; call `.into_handler()` before passing to `hoop()`. `Cors::permissive()` also requires `.into_handler()`.

## Rate limiting

```rust
use salvo::rate_limiter::{BasicQuota, FixedGuard, MokaStore, RateLimiter, RemoteIpIssuer};

let limiter = RateLimiter::new(
    FixedGuard::new(),
    MokaStore::new(),
    RemoteIpIssuer,
    BasicQuota::per_second(10),
);

let router = Router::new().hoop(limiter).get(handler);
```

## Built-in middleware

| Type | Feature flag | Crate path |
|---|---|---|
| `Logger` | `logging` | `salvo::logging` |
| `Compression` | `compression` | `salvo::compression` |
| `Cors` (+ `.into_handler()`) | `cors` | `salvo::cors` |
| `Timeout` | `timeout` | `salvo::timeout` |
| `Csrf` | `csrf` | `salvo::csrf` |
| `RateLimiter` | `rate-limiter` | `salvo::rate_limiter` |
| `MaxConcurrency` | `concurrency-limiter` | `salvo::concurrency_limiter` |
| `MaxSize` | `size-limiter` | `salvo::size_limiter` |
| `CachingHeaders` | `caching-headers` | `salvo::caching_headers` |
| `CatchPanic` | `catch-panic` | `salvo::catch_panic` |
| `RequestId` | `request-id` | `salvo::request_id` |

```rust
use std::time::Duration;
use salvo::logging::Logger;
use salvo::compression::Compression;
use salvo::timeout::Timeout;

let router = Router::new()
    .hoop(Logger::new())
    .hoop(Compression::new())
    .hoop(Timeout::new(Duration::from_secs(30)));
```

## Execution order (onion model)

```rust
Router::new()
    .hoop(a)  // outermost
    .hoop(b)
    .hoop(c)  // innermost
    .get(handler);
// a-before -> b-before -> c-before -> handler
//                                  -> c-after -> b-after -> a-after
```

Code after `ctrl.call_next(...)` runs on the way out. Put logging/timing outermost, auth before authorization, body parsing before handlers that need it.

## Related Skills

- **salvo-auth**: Authentication and authorization middleware
- **salvo-cors**: CORS middleware configuration
- **salvo-compression**: Response compression middleware
- **salvo-logging**: Request logging and tracing middleware
