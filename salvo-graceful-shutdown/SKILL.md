---
name: salvo-graceful-shutdown
description: Implement graceful server shutdown to handle in-flight requests before stopping. Use for zero-downtime deployments and proper resource cleanup.
version: 0.89.3
tags: [operations, shutdown, deployment]
---

# Salvo Graceful Shutdown

Graceful shutdown lives in `salvo-core` behind the `server-handle` feature,
which is on by default when you enable `server`. No extra feature flag needed
for the default `salvo = "0.89.3"` dependency.

```toml
[dependencies]
salvo = "0.89.3"
tokio = { version = "1", features = ["full"] }
```

## API

- `Server::handle(&self) -> ServerHandle` — clone a handle before calling
  `serve(...)`.
- `ServerHandle::stop_graceful(impl Into<Option<Duration>>)` — waits for
  in-flight requests; `None` waits indefinitely, `Some(dur)` forces stop after
  `dur`.
- `ServerHandle::stop_forcible()` — immediate stop, no waiting.

`ServerHandle` is `Clone + Send`, so spawn a task that listens for signals and
calls `stop_graceful` on it.

## Basic pattern

```rust
use salvo::prelude::*;
use salvo::server::ServerHandle;
use tokio::signal;

#[handler]
async fn hello() -> &'static str { "Hello, World!" }

#[tokio::main]
async fn main() {
    let router = Router::new().get(hello);
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;

    let server = Server::new(acceptor);
    let handle = server.handle();
    tokio::spawn(shutdown_signal(handle));

    server.serve(router).await;
}

async fn shutdown_signal(handle: ServerHandle) {
    signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    // Wait indefinitely for in-flight requests
    handle.stop_graceful(None);
}
```

## Shutdown with timeout

```rust
use std::time::Duration;

async fn shutdown_signal(handle: ServerHandle) {
    tokio::signal::ctrl_c().await.unwrap();
    handle.stop_graceful(Duration::from_secs(30));  // Into<Option<Duration>>
}
```

## Cross-platform signal handling

On Unix, also listen for `SIGTERM` (used by Kubernetes/systemd). On Windows,
add `ctrl_break` and `ctrl_close`.

```rust
use salvo::server::ServerHandle;
use std::time::Duration;
use tokio::signal;

async fn shutdown_signal(handle: ServerHandle) {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        use tokio::signal::unix::{signal, SignalKind};
        let mut sigterm = signal(SignalKind::terminate()).expect("install SIGTERM");
        let mut sigquit = signal(SignalKind::quit()).expect("install SIGQUIT");
        tokio::select! {
            _ = sigterm.recv() => {}
            _ = sigquit.recv() => {}
        }
    };

    #[cfg(windows)]
    let terminate = async {
        use tokio::signal::windows;
        let mut ctrl_break = windows::ctrl_break().expect("install ctrl_break");
        let mut ctrl_close = windows::ctrl_close().expect("install ctrl_close");
        tokio::select! {
            _ = ctrl_break.recv() => {}
            _ = ctrl_close.recv() => {}
        }
    };

    tokio::select! {
        _ = ctrl_c => {}
        _ = terminate => {}
    }

    handle.stop_graceful(Duration::from_secs(30));
}
```

## Running cleanup before stopping

`stop_graceful` returns immediately (it only sends a command over an internal
channel). Do any teardown before the call, then allow in-flight requests to
drain as usual.

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

async fn shutdown_signal(handle: ServerHandle, state: Arc<RwLock<AppState>>) {
    tokio::signal::ctrl_c().await.unwrap();

    {
        let state = state.write().await;
        state.db_pool.close().await;
        state.cache.flush().await;
    }

    handle.stop_graceful(std::time::Duration::from_secs(30));
}
```

## Draining traffic via health checks

Flip a flag first, wait for the load balancer to notice, then call
`stop_graceful`. This avoids rejected requests during the handoff.

```rust
use salvo::prelude::*;
use salvo::server::ServerHandle;
use std::sync::atomic::{AtomicBool, Ordering};
use std::time::Duration;

static SHUTTING_DOWN: AtomicBool = AtomicBool::new(false);

#[handler]
async fn health(res: &mut Response) {
    if SHUTTING_DOWN.load(Ordering::Relaxed) {
        res.status_code(StatusCode::SERVICE_UNAVAILABLE);
        res.render(Json(serde_json::json!({"status": "shutting_down"})));
    } else {
        res.render(Json(serde_json::json!({"status": "healthy"})));
    }
}

async fn shutdown_signal(handle: ServerHandle) {
    tokio::signal::ctrl_c().await.unwrap();
    SHUTTING_DOWN.store(true, Ordering::Relaxed);

    // Let the load balancer mark us unhealthy before we stop accepting.
    tokio::time::sleep(Duration::from_secs(5)).await;
    handle.stop_graceful(Duration::from_secs(25));
}
```

## Kubernetes / container orchestrators

Kubernetes sends `SIGTERM` then `SIGKILL` after `terminationGracePeriodSeconds`
(30s default). Use a timeout shorter than the grace period so in-flight work
finishes before SIGKILL.

```rust
use salvo::prelude::*;
use salvo::server::ServerHandle;
use std::time::Duration;

async fn shutdown_signal(handle: ServerHandle) {
    #[cfg(unix)]
    {
        use tokio::signal::unix::{signal, SignalKind};
        signal(SignalKind::terminate()).unwrap().recv().await;
    }
    #[cfg(windows)]
    {
        tokio::signal::ctrl_c().await.unwrap();
    }
    handle.stop_graceful(Duration::from_secs(25));  // < 30s grace period
}
```

## Complete production example

```rust
use salvo::prelude::*;
use salvo::server::ServerHandle;
use std::sync::atomic::{AtomicBool, Ordering};
use std::time::Duration;
use tokio::signal;

static SHUTTING_DOWN: AtomicBool = AtomicBool::new(false);

#[handler]
async fn hello() -> &'static str { "Hello, World!" }

#[handler]
async fn health(res: &mut Response) {
    if SHUTTING_DOWN.load(Ordering::Relaxed) {
        res.status_code(StatusCode::SERVICE_UNAVAILABLE);
        res.render(Json(serde_json::json!({"status": "shutting_down"})));
    } else {
        res.render(Json(serde_json::json!({"status": "healthy"})));
    }
}

async fn shutdown_signal(handle: ServerHandle) {
    let ctrl_c = async { signal::ctrl_c().await.unwrap(); };

    #[cfg(unix)]
    let terminate = async {
        use tokio::signal::unix::{signal, SignalKind};
        signal(SignalKind::terminate()).unwrap().recv().await;
    };
    #[cfg(windows)]
    let terminate = async {
        use tokio::signal::windows;
        windows::ctrl_close().unwrap().recv().await;
    };

    tokio::select! {
        _ = ctrl_c => println!("Ctrl+C received"),
        _ = terminate => println!("Terminate signal received"),
    }

    SHUTTING_DOWN.store(true, Ordering::Relaxed);
    tokio::time::sleep(Duration::from_secs(5)).await;  // drain
    handle.stop_graceful(Duration::from_secs(25));
}

#[tokio::main]
async fn main() {
    let router = Router::new()
        .push(Router::with_path("health").get(health))
        .push(Router::with_path("api").get(hello));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    let server = Server::new(acceptor);
    tokio::spawn(shutdown_signal(server.handle()));
    server.serve(router).await;
}
```

## Related Skills

- **salvo-tls-acme**: HTTPS server shutdown
- **salvo-timeout**: Request timeouts during shutdown
- **salvo-logging**: Log shutdown events
