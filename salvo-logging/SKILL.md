---
name: salvo-logging
description: Implement request logging, tracing, and observability. Use for debugging, monitoring, and production observability.
version: 0.89.3
tags: [operations, logging, tracing, observability]
---

# Salvo Logging and Tracing

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["logging"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

## Logger middleware

`salvo::logging::Logger` emits a `tracing` span per request with `remote_addr`, `version`, `method`, `path`, and response `status` + `duration`. It must be installed on a `Service` (not a `Router`) so it wraps the catcher and sees the final status:

```rust
use salvo::logging::Logger;
use salvo::prelude::*;

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt().init();

    let router = Router::new().get(hello);
    let service = Service::new(router).hoop(Logger::new());

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(service).await;
}
```

`Logger::new().log_status_error(false)` disables auto-logging of `StatusError` bodies (on by default).

## Custom request logger

When `Logger` is not enough, write a handler that measures time around `call_next`:

```rust
use salvo::prelude::*;
use std::time::Instant;
use tracing::info;

#[handler]
async fn request_logger(
    req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl,
) {
    let start = Instant::now();
    let method = req.method().clone();
    let path = req.uri().path().to_string();

    ctrl.call_next(req, depot, res).await;

    info!(
        %method,
        %path,
        status = res.status_code.map(|s| s.as_u16()).unwrap_or(0),
        duration_ms = start.elapsed().as_millis() as u64,
        "request",
    );
}
```

Gotcha: inside a handler `res.status_code` is a `Option<StatusCode>` field (not a method). If the handler hasn't set it, it's `None` until the catcher runs.

## Structured logging in handlers

```rust
use tracing::{debug, info, warn, instrument};

#[handler]
#[instrument(skip_all, fields(user_id))]
async fn get_user(req: &mut Request, res: &mut Response) {
    let user_id: u32 = req.param("id").unwrap_or(0);
    tracing::Span::current().record("user_id", user_id);

    debug!("fetching user");
    match fetch_user(user_id).await {
        Ok(user) => { info!("found"); res.render(Json(user)); }
        Err(e) => { warn!(error = %e, "not found"); res.status_code(StatusCode::NOT_FOUND); }
    }
}
```

## Log filtering

Configure levels via `RUST_LOG` (e.g. `RUST_LOG=info,salvo=debug,hyper=warn`):

```rust
use tracing_subscriber::{EnvFilter, fmt, layer::SubscriberExt, util::SubscriberInitExt};

fn init_logging() {
    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info,salvo=debug,hyper=warn"));

    tracing_subscriber::registry()
        .with(filter)
        .with(fmt::layer())
        .init();
}
```

## JSON logging

```rust
tracing_subscriber::registry()
    .with(EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info")))
    .with(fmt::layer().json().with_current_span(true).with_span_list(true))
    .init();
```

## File logging with rotation

```toml
tracing-appender = "0.2"
```

```rust
use tracing_appender::{rolling, non_blocking};

let file_appender = rolling::daily("logs", "app.log");
let (writer, _guard) = non_blocking(file_appender);

tracing_subscriber::registry()
    .with(EnvFilter::new("info"))
    .with(fmt::layer().with_writer(writer).with_ansi(false))
    .init();
// _guard MUST be held for the process lifetime or buffered logs are dropped.
```

## Request ID

```rust
use uuid::Uuid;

#[handler]
async fn add_request_id(
    req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl,
) {
    let id = req.header::<String>("x-request-id")
        .unwrap_or_else(|| Uuid::new_v4().to_string());
    depot.insert("request_id", id.clone());
    res.headers_mut().insert("x-request-id", id.parse().unwrap());

    let span = tracing::info_span!("request", request_id = %id, method = %req.method(), path = %req.uri().path());
    let _enter = span.enter();
    ctrl.call_next(req, depot, res).await;
}
```

Salvo also ships `salvo::request_id::RequestId` middleware (feature `request-id`) that does this automatically.

## Slow request alerting

```rust
#[handler]
async fn timing(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let start = Instant::now();
    ctrl.call_next(req, depot, res).await;
    let ms = start.elapsed().as_millis();
    if ms > 1000 {
        tracing::warn!(path = %req.uri().path(), duration_ms = ms as u64, "slow request");
    }
}
```

## OpenTelemetry

Salvo provides `salvo_otel` (feature `otel`) for trace propagation. For raw tracing-otel:

```toml
opentelemetry = "0.22"
opentelemetry-otlp = "0.15"
tracing-opentelemetry = "0.23"
```

```rust
let tracer = opentelemetry_otlp::new_pipeline()
    .tracing()
    .with_exporter(opentelemetry_otlp::new_exporter().tonic().with_endpoint("http://localhost:4317"))
    .install_batch(opentelemetry_sdk::runtime::Tokio)
    .unwrap();

tracing_subscriber::registry()
    .with(tracing_subscriber::fmt::layer())
    .with(tracing_opentelemetry::layer().with_tracer(tracer))
    .init();

// On shutdown:
opentelemetry::global::shutdown_tracer_provider();
```

## Gotchas

- Install `Logger` on `Service`, not `Router` — otherwise it logs before the catcher rewrites the status and body.
- `Logger` requires a `tracing` subscriber; without one, nothing is emitted.
- `tracing-appender` writers must keep their `WorkerGuard` alive or pending log lines are discarded at shutdown.
- Never log request bodies or headers like `Authorization` / `Cookie` unredacted.

## Related Skills

- **salvo-error-handling**: `Logger` auto-logs `StatusError` bodies.
- **salvo-middleware**: Logger is just a hoop.
- **salvo-testing**: `tracing-test` crate verifies log output.
