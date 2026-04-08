---
name: salvo-static-files
description: Serve static files, directories, and embedded assets. Use for CSS, JavaScript, images, and downloadable content.
version: 0.89.3
tags: [data, static-files, assets, serve-static]
---

# Salvo Static File Serving

## Setup

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["serve-static"] }
rust-embed = "8"   # only for embedded assets
```

Catch-all path segment uses `{*path}` (greedy, single segment required) or `{**path}` (greedy, may be empty).

## StaticDir

`StaticDir::new(roots)` accepts one or more root directories tried in order (first match wins). Builder methods:

- `defaults(names)` — file(s) to try when the URL points to a directory (e.g. `"index.html"`)
- `fallback(name)` — file served when no match is found (for SPAs)
- `auto_list(bool)` — enable directory listing
- `include_dot_files(bool)` — include files starting with `.`
- `exclude(fn)` — filter predicate
- `chunk_size(u64)` — streaming chunk size
- `compressed_variation(algo, exts)` — serve pre-compressed variants

Note: `StaticDir` does NOT expose a `cache_control()` builder. For cache headers, add a middleware that sets them on the response.

```rust
use salvo::prelude::*;
use salvo::serve_static::StaticDir;

#[tokio::main]
async fn main() {
    let router = Router::with_path("{*path}").get(
        StaticDir::new(["static", "public"])
            .defaults("index.html")
            .auto_list(true),
    );
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Setting Cache Headers

Since `StaticDir` has no builder for cache headers, use a middleware:

```rust
use salvo::prelude::*;
use salvo::http::header::{CACHE_CONTROL, HeaderValue};

#[handler]
async fn set_cache(res: &mut Response) {
    res.headers_mut().insert(
        CACHE_CONTROL,
        HeaderValue::from_static("public, max-age=31536000, immutable"),
    );
}

let assets = Router::with_path("assets/{*path}")
    .hoop(set_cache)
    .get(StaticDir::new(["static/assets"]));
```

## StaticFile (Single File)

```rust
use salvo::prelude::*;
use salvo::serve_static::StaticFile;

let router = Router::new()
    .push(Router::with_path("favicon.ico").get(StaticFile::new("static/favicon.ico")))
    .push(Router::with_path("robots.txt").get(StaticFile::new("static/robots.txt")));
```

`StaticFile` wraps a `NamedFileBuilder`, so it handles ETag, `Range`, and conditional requests.

## Embedded Assets (`rust-embed`)

Embed files at compile time for single-binary deployment.

```rust
use rust_embed::RustEmbed;
use salvo::prelude::*;
use salvo::serve_static::static_embed;

#[derive(RustEmbed)]
#[folder = "dist"]
struct Assets;

#[tokio::main]
async fn main() {
    let router = Router::with_path("{*path}").get(
        static_embed::<Assets>().fallback("index.html"),  // SPA fallback
    );
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

`StaticEmbed` builder methods: `defaults(names)`, `fallback(name)`. No `cache_control` here either.

## API + Static Hybrid

Put API routes before the catch-all static route so they match first.

```rust
use salvo::prelude::*;
use salvo::serve_static::StaticDir;

#[handler]
async fn api_users() -> Json<Vec<&'static str>> { Json(vec!["Alice", "Bob"]) }

#[tokio::main]
async fn main() {
    let router = Router::new()
        .push(Router::with_path("api/users").get(api_users))
        .push(
            Router::with_path("{*path}").get(
                StaticDir::new(["static"]).defaults("index.html"),
            ),
        );
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Protected Static Content

```rust
use salvo::prelude::*;
use salvo::serve_static::StaticDir;
use salvo::session::SessionDepotExt;

#[handler]
async fn require_login(depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let logged_in = depot
        .session_mut()
        .and_then(|s| s.get::<bool>("logged_in"))
        .unwrap_or(false);
    if !logged_in {
        res.status_code(StatusCode::UNAUTHORIZED);
        ctrl.skip_rest();
    }
}

let router = Router::new()
    .push(Router::with_path("public/{*path}").get(StaticDir::new(["static/public"])))
    .push(
        Router::with_path("private/{*path}")
            .hoop(require_login)
            .get(StaticDir::new(["static/private"])),
    );
```

## Custom Embedded Handler

If you need custom cache rules or content-type overrides, write a handler directly over `RustEmbed::get`.

```rust
use rust_embed::RustEmbed;
use salvo::http::header::{CACHE_CONTROL, CONTENT_TYPE, HeaderValue};
use salvo::prelude::*;

#[derive(RustEmbed)]
#[folder = "static"]
struct Assets;

#[handler]
async fn serve(req: &mut Request, res: &mut Response) {
    let path = req.param::<String>("path").unwrap_or_default();
    let Some(asset) = Assets::get(&path).or_else(|| Assets::get("index.html")) else {
        res.status_code(StatusCode::NOT_FOUND);
        return;
    };
    let ct = mime_guess::from_path(&path).first_or_octet_stream().to_string();
    res.headers_mut().insert(CONTENT_TYPE, ct.parse().unwrap());
    if path.contains('.') {
        res.headers_mut().insert(
            CACHE_CONTROL,
            HeaderValue::from_static("public, max-age=31536000, immutable"),
        );
    }
    let _ = res.write_body(asset.data.to_vec());
}
```

## Compression

`Compression` builder takes `CompressionLevel`, not raw integers or `flate2::Compression`.

```rust
use salvo::compression::{Compression, CompressionLevel};
use salvo::prelude::*;
use salvo::serve_static::StaticDir;

#[tokio::main]
async fn main() {
    let compression = Compression::new()
        .enable_gzip(CompressionLevel::Default)
        .enable_brotli(CompressionLevel::Default)
        .min_length(1024);

    let router = Router::new()
        .hoop(compression)
        .push(Router::with_path("{*path}").get(
            StaticDir::new(["dist"]).defaults("index.html"),
        ));
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

Requires `compression` feature on `salvo`.

## Salvo-Specific Notes

- `{*path}` = one or more segments; `{**path}` = zero or more.
- `StaticDir`/`StaticEmbed` have no `cache_control` builder — set headers via middleware.
- `StaticDir::new([...])` multi-root = lookup fallback order, great for theme overrides.
- Use `fallback("index.html")` for SPA client-side routing; use `defaults("index.html")` only for directory index behavior.

## Related Skills

- **salvo-file-handling**: File uploads and downloads via `NamedFile`
- **salvo-compression**: Response compression details
- **salvo-caching**: ETag/cache middleware
