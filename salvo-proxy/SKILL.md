---
name: salvo-proxy
description: Implement reverse proxy to forward requests to backend services. Use for load balancing, API gateways, and microservices routing.
version: 0.89.3
tags: [advanced, proxy, reverse-proxy, gateway]
---

# Salvo Reverse Proxy

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["proxy"] }
```

The `proxy` feature enables the `hyper-client` subfeature by default. For
`reqwest-client` or `unix-sock-client`, enable those features explicitly on
`salvo-proxy`.

## Creating a Proxy

`Proxy::new(upstreams, client)` is the full constructor. `upstreams` is anything
implementing `Upstreams` — `&'static str`, `String`, `Vec<_>`, `[_; N]`, or a
custom impl. `Proxy<_, _>` implements `Handler`, so mount it with `.goal(...)`.

```rust
use salvo::prelude::*;
use salvo::proxy::{HyperClient, Proxy};

let proxy = Proxy::new(vec!["http://localhost:3000"], HyperClient::default());

let router = Router::new().push(Router::with_path("{**rest}").goal(proxy));
```

Shortcut: `Proxy::use_hyper_client(upstreams)` is equivalent to
`Proxy::new(upstreams, HyperClient::default())`.

### ReqwestClient

```rust
use salvo::proxy::{Proxy, ReqwestClient};

let proxy = Proxy::new(vec!["http://localhost:3000"], ReqwestClient::default());
```

### Forward client IP

`client_ip_forwarding(true)` (or `Proxy::with_client_ip_forwarding(...)`) prepends
the caller's address to `X-Forwarded-For`. Off by default.

```rust
let proxy = Proxy::new(vec!["http://backend:3000"], HyperClient::default())
    .client_ip_forwarding(true);
```

## Load balancing

Passing multiple upstreams uses a built-in round-robin rotation via the
`Vec<_>`/slice `Upstreams` impl:

```rust
let proxy = Proxy::new(
    vec!["http://backend1:3000", "http://backend2:3000", "http://backend3:3000"],
    HyperClient::default(),
);
```

For custom selection (health-aware, weighted, sticky), implement the `Upstreams`
trait on your own type.

## Path-based routing / API gateway

```rust
use salvo::prelude::*;
use salvo::proxy::{HyperClient, Proxy};

#[tokio::main]
async fn main() {
    let users = Proxy::new(vec!["http://user-service:3001"], HyperClient::default());
    let orders = Proxy::new(vec!["http://order-service:3002"], HyperClient::default());
    let products = Proxy::new(vec!["http://product-service:3003"], HyperClient::default());

    let router = Router::with_path("api/v1")
        .hoop(auth_middleware)
        .push(Router::with_path("users/{**rest}").goal(users))
        .push(Router::with_path("orders/{**rest}").goal(orders))
        .push(Router::with_path("products/{**rest}").goal(products));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Authentication before proxying

`.hoop(...)` middleware runs before `.goal(proxy)`, so call `ctrl.skip_rest()` to
block the proxy when auth fails.

```rust
#[handler]
async fn auth_middleware(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    match req.header::<String>("Authorization") {
        Some(t) if validate_token(&t) => ctrl.call_next(req, depot, res).await,
        _ => {
            res.status_code(StatusCode::UNAUTHORIZED);
            res.render("Unauthorized");
            ctrl.skip_rest();
        }
    }
}

let router = Router::with_path("api/{**rest}")
    .hoop(auth_middleware)
    .goal(Proxy::new(vec!["http://backend:3000"], HyperClient::default()));
```

## Adding forwarded headers manually

`client_ip_forwarding(true)` handles `X-Forwarded-For`. For other headers
(`X-Forwarded-Proto`, etc.), mutate the request in a `hoop`:

```rust
#[handler]
async fn add_proxy_headers(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    req.headers_mut().insert("X-Forwarded-Proto", "https".parse().unwrap());
    ctrl.call_next(req, depot, res).await;
}
```

## Customising URL/host rewriting

```rust
let proxy = Proxy::new(vec!["http://backend:3000"], HyperClient::default())
    .url_path_getter(|req, _depot| {
        // Strip "/api" prefix before forwarding
        req.uri().path().strip_prefix("/api").map(String::from)
    })
    .host_header_getter(|uri, _req, _depot| uri.host().map(String::from));
```

`url_query_getter` is also available for query rewriting.

## Rate limiting

```rust
use salvo::rate_limiter::{BasicQuota, FixedGuard, MokaStore, RateLimiter, RemoteIpIssuer};

let limiter = RateLimiter::new(
    FixedGuard::new(),
    MokaStore::new(),
    RemoteIpIssuer,
    BasicQuota::per_second(100),
);

let proxy = Proxy::new(vec!["http://backend:3000"], HyperClient::default());
let router = Router::with_path("{**rest}").hoop(limiter).goal(proxy);
```

## WebSocket proxy

WebSocket upgrades are handled transparently by `Proxy` — no extra config needed:

```rust
let ws_proxy = Proxy::new(vec!["http://ws-backend:9000"], HyperClient::default());
let router = Router::with_path("ws/{**rest}").goal(ws_proxy);
```

## Complete gateway example

```rust
use salvo::compression::Compression;
use salvo::cors::Cors;
use salvo::prelude::*;
use salvo::proxy::{HyperClient, Proxy};

#[handler]
async fn logging(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let start = std::time::Instant::now();
    let method = req.method().clone();
    let path = req.uri().path().to_string();
    ctrl.call_next(req, depot, res).await;
    let status = res.status_code().unwrap_or(StatusCode::OK);
    println!("{} {} -> {} ({:?})", method, path, status, start.elapsed());
}

#[tokio::main]
async fn main() {
    let users = Proxy::new(vec!["http://users:3001"], HyperClient::default())
        .client_ip_forwarding(true);
    let orders = Proxy::new(vec!["http://orders:3002"], HyperClient::default())
        .client_ip_forwarding(true);

    let router = Router::new()
        .hoop(logging)
        .hoop(Cors::permissive().into_handler())
        .hoop(Compression::new())
        .push(Router::with_path("users/{**rest}").goal(users))
        .push(Router::with_path("orders/{**rest}").goal(orders));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Related Skills

- **salvo-rate-limiter**: Rate limit proxied requests
- **salvo-auth**: Authenticate before proxying
- **salvo-timeout**: Set timeouts for upstream requests
