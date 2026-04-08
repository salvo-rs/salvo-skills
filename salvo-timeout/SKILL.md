---
name: salvo-timeout
description: Configure request timeouts to prevent slow requests from blocking resources. Use for protecting APIs from long-running operations.
version: 0.89.3
tags: [performance, timeout, request-timeout]
---

# Salvo Request Timeout

`Timeout` lives in `salvo_extra` behind a feature flag — it is NOT on by default.

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["timeout"] }
```

`Timeout` is re-exported via `salvo::prelude::Timeout`.

## Basic usage

```rust
use std::time::Duration;
use salvo::prelude::*;

#[handler]
async fn slow() -> &'static str {
    tokio::time::sleep(Duration::from_secs(10)).await;
    "done"
}

#[tokio::main]
async fn main() {
    let router = Router::new()
        .hoop(Timeout::new(Duration::from_secs(5)))
        .push(Router::with_path("slow").get(slow));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

GOTCHA: on timeout Salvo returns **`503 Service Unavailable`**, not `408 Request Timeout`. This is intentional — some browsers retry 408 automatically. The response also includes `Connection: close`.

## Custom error

Override the error via `.error(|| StatusError)`:

```rust
use salvo::http::StatusError;

let timeout = Timeout::new(Duration::from_secs(5))
    .error(|| StatusError::gateway_timeout().brief("Upstream too slow."));
```

## Per-route timeouts

```rust
let router = Router::new()
    .push(Router::with_path("quick")
        .hoop(Timeout::new(Duration::from_secs(2)))
        .get(quick_handler))
    .push(Router::with_path("upload")
        .hoop(Timeout::new(Duration::from_secs(120)))
        .post(upload_handler))
    .push(Router::with_path("report")
        .hoop(Timeout::new(Duration::from_secs(300)))
        .post(report_handler));
```

## Global default + per-route override

Inner `hoop` runs after the outer one; a longer timeout on an inner route effectively overrides the shorter global timeout because the first handler to complete (or expire) wins the `tokio::select!` race.

```rust
let router = Router::new()
    .hoop(Timeout::new(Duration::from_secs(30)))  // default
    .push(Router::with_path("health").get(health))
    .push(
        Router::with_path("reports/generate")
            .hoop(Timeout::new(Duration::from_secs(300)))
            .post(generate_report),
    );
```

## Handling timeouts in a Catcher

A `Catcher` can customize the 503 body:

```rust
use salvo::catcher::Catcher;

#[handler]
async fn on_timeout(res: &mut Response, ctrl: &mut FlowCtrl) {
    if res.status_code == Some(StatusCode::SERVICE_UNAVAILABLE) {
        res.render(Json(serde_json::json!({
            "error": "timeout",
            "message": "The request took too long to process",
        })));
        ctrl.skip_rest();
    }
}

let service = Service::new(router).catcher(Catcher::default().hoop(on_timeout));
```

## Combining with rate limiting

```rust
let router = Router::new()
    .hoop(rate_limiter)                              // 429 on abuse
    .hoop(Timeout::new(Duration::from_secs(30)))     // 503 on stall
    .push(Router::with_path("api/{**rest}").get(api_handler));
```

## WebSocket / SSE note

Do NOT wrap long-lived endpoints (WebSocket upgrade, SSE) with `Timeout` — the middleware aborts any handler that runs longer than the duration, which will terminate these connections.

## Related Skills

- **salvo-concurrency-limiter**: cap in-flight requests
- **salvo-rate-limiter**: rate limiting
- **salvo-graceful-shutdown**: handle in-flight requests on shutdown
