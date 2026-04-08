---
name: salvo-caching
description: Implement caching strategies for improved performance. Use for reducing database load and speeding up responses.
version: 0.89.3
tags: [performance, caching, cache-control, etag]
---

# Salvo Caching Strategies

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["cache", "caching-headers"] }
```

The `cache` feature activates `salvo-cache` with the default `moka-store` backend.

## Response cache middleware

`salvo::cache::Cache` caches full responses (status, headers, body). By default it caches GET only and skips streaming responses and error bodies.

```rust
use std::time::Duration;
use salvo::cache::{Cache, MokaStore, RequestIssuer};
use salvo::prelude::*;

#[handler]
async fn expensive() -> String {
    format!("computed at {}", chrono::Utc::now())
}

#[tokio::main]
async fn main() {
    let cache = Cache::new(
        MokaStore::builder()
            .time_to_live(Duration::from_secs(60))
            .max_capacity(10_000)
            .build(),
        RequestIssuer::default(),
    );

    let router = Router::new().hoop(cache).get(expensive);
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

`Cache::new(store, issuer)` is the only constructor. Tunable:
- `.skipper(impl Skipper)` — default is `MethodSkipper::new().skip_all().skip_get(false)` (GET only). Pass a closure `|req, depot| bool` to override.

## RequestIssuer

Builds cache keys from the request URI and method. Toggle parts:

```rust
let issuer = RequestIssuer::new()
    .use_scheme(false)
    .use_authority(false)
    .use_path(true)
    .use_query(true)
    .use_method(true);
```

All five are `true` by default.

## Custom CacheIssuer

Implement `CacheIssuer` (or pass a closure) to vary cache keys by user, tenant, etc.:

```rust
use salvo::cache::CacheIssuer;

let issuer = |req: &mut Request, depot: &Depot| -> Option<String> {
    let user_id = depot.get::<String>("user_id").ok()?;
    Some(format!("{user_id}:{}", req.uri().path()))
};
let cache = Cache::new(store, issuer);
```

Returning `None` disables caching for that request.

## CachingHeaders (ETag + Last-Modified)

`salvo::caching_headers::CachingHeaders` adds ETag and Last-Modified handling to downstream handlers, responding `304` when `If-None-Match` / `If-Modified-Since` match:

```rust
use salvo::caching_headers::CachingHeaders;

let router = Router::new()
    .hoop(CachingHeaders::new())
    .get(handler);
```

For ETag-only or Last-Modified-only, use `ETag::new()` or `Modified::new()` from the same module.

## HTTP cache headers manually

```rust
#[handler]
async fn cached(res: &mut Response) -> &'static str {
    res.headers_mut().insert(
        "cache-control",
        "public, max-age=3600, stale-while-revalidate=86400".parse().unwrap(),
    );
    "hello"
}
```

Directives cheat sheet:
- `public, max-age=N` — shared caches may store for N seconds
- `private, max-age=N` — browser only
- `no-store` — never cache
- `no-cache` — must revalidate with origin
- `stale-while-revalidate=N` — serve stale for N seconds while revalidating

## Manual 304 response

`StatusError` has no `not_modified()`. Set the status directly:

```rust
#[handler]
async fn with_etag(req: &mut Request, res: &mut Response) {
    let etag = compute_etag();
    if req.header::<String>("if-none-match").as_deref() == Some(etag.as_str()) {
        res.status_code(StatusCode::NOT_MODIFIED);
        return;
    }
    res.headers_mut().insert("etag", etag.parse().unwrap());
    res.render(Json(load_data().await));
}
```

## Data-layer caching with Moka

For caching values inside handlers (not full responses), use `moka::future::Cache` directly and share via `affix_state`:

```toml
moka = { version = "0.12", features = ["future"] }
```

```rust
use moka::future::Cache as MokaCache;
use std::sync::Arc;

type UserCache = Arc<MokaCache<i64, User>>;

#[handler]
async fn get_user(req: &mut Request, depot: &mut Depot) -> Result<Json<User>, StatusError> {
    let id = req.param::<i64>("id").ok_or_else(StatusError::bad_request)?;
    let cache = depot.obtain::<UserCache>().unwrap();
    let pool = depot.obtain::<PgPool>().unwrap();

    if let Some(user) = cache.get(&id).await { return Ok(Json(user)); }

    let user = sqlx::query_as::<_, User>("select id, name from users where id = $1")
        .bind(id).fetch_optional(pool).await
        .map_err(|_| StatusError::internal_server_error())?
        .ok_or_else(StatusError::not_found)?;

    cache.insert(id, user.clone()).await;
    Ok(Json(user))
}

#[tokio::main]
async fn main() {
    let cache: UserCache = Arc::new(
        MokaCache::builder()
            .max_capacity(1_000)
            .time_to_live(std::time::Duration::from_secs(60))
            .build(),
    );
    let router = Router::new()
        .hoop(affix_state::inject(cache))
        .push(Router::with_path("users/{id}").get(get_user));
    // ...
}
```

Invalidate on write:

```rust
#[handler]
async fn update_user(req: &mut Request, depot: &mut Depot) -> StatusCode {
    let id = req.param::<i64>("id").unwrap();
    // update db...
    depot.obtain::<UserCache>().unwrap().invalidate(&id).await;
    StatusCode::OK
}
```

`invalidate_all()` clears everything; iterate over IDs for bulk invalidation.

## Gotchas

- `Cache` is full-response. It needs a deterministic key — beware caching per-user data under a shared `RequestIssuer`; use a custom issuer keyed on user ID.
- Streaming responses (`ResBody::Stream`) and error bodies are never cached, silently.
- `Cache::new` does not return a builder — configure the store via `MokaStore::builder()`.
- Method names: `MokaStore::builder()` (not `Cache::builder`), `RequestIssuer::default()` or `new()`.
- `StatusError::not_modified()` does NOT exist — set `StatusCode::NOT_MODIFIED` directly.

## Related Skills

- **salvo-compression**: Compress before `Cache` so the stored body is already gzipped.
- **salvo-static-files**: `serve-static` sets ETag/Last-Modified automatically.
- **salvo-database**: Cache hot query results at the data layer.
