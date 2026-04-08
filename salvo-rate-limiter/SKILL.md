---
name: salvo-rate-limiter
description: Implement rate limiting to protect APIs from abuse. Use for preventing DDoS attacks and ensuring fair resource usage.
version: 0.89.3
tags: [security, rate-limiting, throttling]
---

# Salvo Rate Limiting

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["rate-limiter"] }
```

## Components

A `RateLimiter` combines four pieces:

| Component | Purpose | Built-ins |
|-----------|---------|-----------|
| `RateIssuer` | Identify client | `RemoteIpIssuer`, `RealIpIssuer` |
| `RateGuard` | Limiting algorithm | `FixedGuard` (needs `BasicQuota`), `SlidingGuard` (needs `CelledQuota`) |
| `RateStore` | Persist state | `MokaStore` |
| `QuotaGetter` | Lookup quota for key | any `Clone` `Quota`, or custom impl |

## Fixed Window by IP

```rust
use salvo::prelude::*;
use salvo::rate_limiter::{BasicQuota, FixedGuard, MokaStore, RateLimiter, RemoteIpIssuer};

#[handler]
async fn api() -> &'static str { "ok" }

#[tokio::main]
async fn main() {
    let limiter = RateLimiter::new(
        FixedGuard::default(),
        MokaStore::default(),
        RemoteIpIssuer,
        BasicQuota::per_second(10),
    );

    let router = Router::new().hoop(limiter).push(Router::with_path("api").get(api));
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## BasicQuota constructors

```rust
BasicQuota::per_second(10)
BasicQuota::per_minute(100)
BasicQuota::per_hour(1000)
BasicQuota::set_seconds(50, 30)   // 50 per 30s
BasicQuota::set_minutes(500, 5)
BasicQuota::set_hours(5000, 2)
// Raw: BasicQuota::new(limit, time::Duration::seconds(n))  -- NOTE: time::Duration, not std::time
```

## Sliding Window (smoother limiting)

`SlidingGuard` requires `CelledQuota` (limit split into N cells). Passing `BasicQuota` will not compile.

```rust
use salvo::rate_limiter::{CelledQuota, SlidingGuard, MokaStore, RateLimiter, RemoteIpIssuer};

let limiter = RateLimiter::new(
    SlidingGuard::default(),
    MokaStore::default(),
    RemoteIpIssuer,
    CelledQuota::per_minute(60, 6),  // 60 req/min split into 6 x 10s cells
);
```

`CelledQuota` has the same per_*/set_* constructors as `BasicQuota` but each takes an extra `cells: usize` parameter.

## Behind a reverse proxy

`RemoteIpIssuer` uses the direct connection IP (the proxy). Use `RealIpIssuer` to read `X-Forwarded-For` / `X-Real-IP`:

```rust
use salvo::rate_limiter::RealIpIssuer;

let limiter = RateLimiter::new(
    FixedGuard::default(), MokaStore::default(),
    RealIpIssuer::new(),
    BasicQuota::per_minute(100),
);
```

WARNING: only use `RealIpIssuer` behind a TRUSTED proxy that overwrites these headers, otherwise clients can forge them.

## Custom issuer (user ID / API key)

`RateIssuer::issue` takes `(&mut Request, &Depot)`:

```rust
use salvo::prelude::*;
use salvo::rate_limiter::RateIssuer;

struct UserIdIssuer;
impl RateIssuer for UserIdIssuer {
    type Key = String;
    async fn issue(&self, _req: &mut Request, depot: &Depot) -> Option<Self::Key> {
        depot.get::<String>("user_id").ok().cloned()
    }
}

struct ApiKeyIssuer;
impl RateIssuer for ApiKeyIssuer {
    type Key = String;
    async fn issue(&self, req: &mut Request, _depot: &Depot) -> Option<Self::Key> {
        req.header::<String>("x-api-key")
    }
}
```

Hybrid (user-if-authed, IP otherwise):

```rust
struct SmartIssuer;
impl RateIssuer for SmartIssuer {
    type Key = String;
    async fn issue(&self, req: &mut Request, depot: &Depot) -> Option<Self::Key> {
        if let Ok(id) = depot.get::<String>("user_id") {
            Some(format!("user:{id}"))
        } else {
            Some(format!("ip:{}", req.remote_addr().ip()?))
        }
    }
}
```

A closure `Fn(&mut Request, &Depot) -> Option<K>` also implements `RateIssuer` directly.

## Dynamic per-user quotas

GOTCHA: `QuotaGetter::get` takes only `&Q` — no `Depot`. Look up quota by key alone (e.g. from a static map or DB).

```rust
use std::borrow::Borrow;
use std::hash::Hash;
use salvo::Error;
use salvo::rate_limiter::{BasicQuota, QuotaGetter};

struct TieredQuota;
impl QuotaGetter<String> for TieredQuota {
    type Quota = BasicQuota;
    type Error = Error;

    async fn get<Q>(&self, key: &Q) -> Result<Self::Quota, Self::Error>
    where
        String: Borrow<Q>,
        Q: Hash + Eq + Sync,
    {
        // Lookup tier by key (from DB, cache, etc.)
        Ok(BasicQuota::per_minute(100))
    }
}
```

Any `Clone + Send + Sync` quota type auto-implements `QuotaGetter` returning itself, which is why `BasicQuota::per_second(10)` works as the fourth argument directly.

## Response headers

Enable built-in `X-RateLimit-Limit` / `-Remaining` / `-Reset` headers — no manual middleware needed:

```rust
let limiter = RateLimiter::new(FixedGuard::default(), MokaStore::default(),
    RemoteIpIssuer, BasicQuota::per_minute(100))
    .add_headers(true);
```

When the limit is exceeded, Salvo returns `429 Too Many Requests`.

## Per-route limits

```rust
let login_limiter = RateLimiter::new(FixedGuard::default(), MokaStore::default(),
    RemoteIpIssuer, BasicQuota::per_minute(5));
let api_limiter = RateLimiter::new(FixedGuard::default(), MokaStore::default(),
    RemoteIpIssuer, BasicQuota::per_minute(100));

let router = Router::new()
    .push(Router::with_path("login").hoop(login_limiter).post(login))
    .push(Router::with_path("api").hoop(api_limiter).get(api));
```

## Skipper

Skip rate limiting for some requests:

```rust
let limiter = RateLimiter::new(/* ... */)
    .with_skipper(|req: &mut Request, _: &Depot| {
        req.uri().path().starts_with("/health")
    });
```

## Related Skills

- **salvo-concurrency-limiter**: limit concurrent requests
- **salvo-auth**: combine with authentication
- **salvo-timeout**: set timeouts alongside rate limits
