---
name: salvo-compression
description: Compress HTTP responses using gzip, brotli, zstd, or deflate. Use for reducing bandwidth and improving load times.
version: 0.89.3
tags: [performance, compression, gzip, brotli]
---

# Salvo Response Compression

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["compression"] }
```

## Basic usage

`Compression::new()` enables all four algorithms (brotli, zstd, gzip, deflate) at `Default` level, with `min_length = 1024` and a default text/* content-type allowlist.

```rust
use salvo::prelude::*;
use salvo::compression::Compression;

#[handler]
async fn body() -> String {
    "lorem ipsum ".repeat(200)
}

#[tokio::main]
async fn main() {
    let router = Router::new().hoop(Compression::new()).get(body);
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## CompressionLevel

```rust
use salvo::compression::CompressionLevel;

CompressionLevel::Fastest      // speed > ratio
CompressionLevel::Default      // balanced
CompressionLevel::Minsize      // best ratio, slowest  (NOT `Best`)
CompressionLevel::Precise(6)   // algo-specific level
```

GOTCHA: the variant is `Minsize`, not `Best`.

## Selecting algorithms

Enable/disable by name (each returns `Self`):

```rust
use salvo::compression::{Compression, CompressionLevel};

// Only gzip
let only_gzip = Compression::new()
    .disable_all()
    .enable_gzip(CompressionLevel::Default);

// Gzip + brotli, different levels
let mixed = Compression::new()
    .disable_all()
    .enable_brotli(CompressionLevel::Minsize)
    .enable_gzip(CompressionLevel::Fastest);
```

Available: `enable_gzip`, `enable_brotli`, `enable_zstd`, `enable_deflate` (plus matching `disable_*`).

## min_length

```rust
Compression::new().min_length(2048);  // skip bodies smaller than 2KB
```

## content_types

GOTCHA: `content_types` takes `&[mime::Mime]`, NOT `&[&str]`. `mime` is re-exported at `salvo::http::mime`.

```rust
use salvo::compression::Compression;
use salvo::http::mime;

let comp = Compression::new().content_types(&[
    mime::TEXT_HTML,
    mime::TEXT_CSS,
    mime::APPLICATION_JSON,
    mime::APPLICATION_JAVASCRIPT,
    "image/svg+xml".parse().unwrap(),
]);
```

The default list already covers `text/*`, `application/javascript`, `application/json`, and `image/svg+xml` — only call `content_types` to restrict further.

## Algorithm priority

By default, if the client accepts multiple encodings, the algorithm with the highest client q-value wins. Set `force_priority(true)` to use server preference order (brotli > zstd > gzip > deflate as inserted by `new()`).

```rust
Compression::new().force_priority(true);
```

## Per-route compression

```rust
let static_comp = Compression::new()
    .disable_all()
    .enable_brotli(CompressionLevel::Minsize)
    .enable_gzip(CompressionLevel::Minsize);

let api_comp = Compression::new()
    .disable_all()
    .enable_gzip(CompressionLevel::Fastest)
    .min_length(256);

let router = Router::new()
    .push(Router::with_path("static/{**path}").hoop(static_comp).get(StaticDir::new("./public")))
    .push(Router::with_path("api/{**rest}").hoop(api_comp).get(api_handler));
```

## Pre-compressed static assets

If you pre-compress files at build time (`gzip -k9 bundle.js`, `brotli -k bundle.js`), serve them with `StaticDir` which detects the pre-compressed variants automatically and serves them directly.

## Related Skills

- **salvo-static-files**: serve pre-compressed files
- **salvo-caching**: combine with caching middleware
