---
name: salvo-concurrency-limiter
description: Limit concurrent requests to protect resources. Use for file uploads, expensive operations, and preventing resource exhaustion.
version: 0.89.3
tags: [performance, concurrency, limiter]
---

# Salvo Concurrency Limiter

`max_concurrency(n)` from `salvo::concurrency_limiter` is a hoop that wraps a `tokio::sync::Semaphore`. It `try_acquire`s on each request — on failure it short-circuits with `429 Too Many Requests`. Unlike rate limiters it caps **parallel** requests, not request frequency.

## Setup

Lives in `salvo-extra`; enable the feature:

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["concurrency-limiter"] }
```

Import path: `salvo::concurrency_limiter::{max_concurrency, MaxConcurrency}`.

## Basic usage

```rust
use salvo::prelude::*;
use salvo::concurrency_limiter::max_concurrency;

#[handler]
async fn upload(res: &mut Response) {
    tokio::time::sleep(std::time::Duration::from_secs(10)).await;
    res.render("Upload complete");
}

#[tokio::main]
async fn main() {
    let router = Router::new().push(
        Router::with_path("upload")
            .hoop(max_concurrency(1))   // at most 1 in flight
            .post(upload),
    );
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

Each `hoop(max_concurrency(n))` call creates its **own** semaphore scoped to that route subtree. Reuse across subtrees by hoisting the hoop on a shared parent.

## Per-route limits

```rust
let router = Router::new()
    .push(Router::with_path("upload")
        .hoop(max_concurrency(2))
        .post(upload_handler))
    .push(Router::with_path("reports/generate")
        .hoop(max_concurrency(1))
        .post(generate_report))
    .push(Router::with_path("api/{**rest}")
        .hoop(max_concurrency(100))
        .goal(api_handler))
    .push(Router::with_path("health").get(health_check));   // no limit
```

## Combining middleware

`hoop` order matters — outer hoops run first. Put rate-limit/timeout outside the concurrency cap so rejected requests don't hold permits:

```rust
Router::with_path("api/{**rest}")
    .hoop(rate_limiter)             // runs first
    .hoop(Timeout::new(Duration::from_secs(30)))
    .hoop(max_concurrency(50))      // permit acquired last, released on handler exit
    .goal(api_handler)
```

## Customizing the 429 response

The middleware renders `StatusError::too_many_requests().brief("Max concurrency reached.")`. Override in a catcher:

```rust
use salvo::catcher::Catcher;

#[handler]
async fn on_overload(res: &mut Response, ctrl: &mut FlowCtrl) {
    if res.status_code() == Some(StatusCode::TOO_MANY_REQUESTS) {
        res.render(Json(serde_json::json!({
            "error": "busy",
            "retry_after": 5,
        })));
        ctrl.skip_rest();
    }
}

let service = Service::new(router).catcher(Catcher::default().hoop(on_overload));
```

## Sizing guidance

| Workload              | Limit                  |
|-----------------------|------------------------|
| File upload           | 1–5                    |
| Image/video transcode | ~`num_cpus::get()`     |
| Heavy DB queries      | DB pool size           |
| External API callouts | Vendor's concurrency   |
| General API           | 50–200                 |

## Gotchas

- Shares no state across processes — for multi-replica deployments, use a distributed limiter upstream.
- Each `hoop(max_concurrency(..))` is a **separate** semaphore; instantiating it inside a loop or per-request builder creates independent limits and defeats the cap.
- Permits are released when the inner handler returns. Pair with `Timeout` so stuck handlers eventually free the slot.
- The middleware uses `try_acquire`: requests are rejected, **not** queued. If you want queueing, put a `tower::limit` layer in via `tower-compat` instead.

## Related Skills

- **salvo-rate-limiter**: Request frequency (per-second/minute) caps
- **salvo-timeout**: Combine with request deadlines so stuck handlers don't block slots
- **salvo-realtime**: Cap simultaneous SSE/WebSocket connections
