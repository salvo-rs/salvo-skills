---
name: salvo-tls-acme
description: Configure TLS/HTTPS with automatic certificate management via ACME (Let's Encrypt). Use for production deployments with secure connections.
version: 0.89.3
tags: [security, tls, https, acme, certificate]
---

# Salvo TLS and ACME

## Static TLS with Rustls

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["rustls"] }
```

```rust
use salvo::prelude::*;
use salvo::conn::rustls::{Keycert, RustlsConfig};

#[handler]
async fn hello() -> &'static str { "Hello HTTPS" }

#[tokio::main]
async fn main() {
    let router = Router::new().get(hello);

    let config = RustlsConfig::new(
        Keycert::new()
            .cert_from_path("certs/cert.pem").unwrap()
            .key_from_path("certs/key.pem").unwrap()
    );

    let acceptor = TcpListener::new("0.0.0.0:443").rustls(config).bind().await;
    Server::new(acceptor).serve(router).await;
}
```

Loading from memory:

```rust
let cert = include_bytes!("../certs/cert.pem");
let key = include_bytes!("../certs/key.pem");
let config = RustlsConfig::new(Keycert::new().cert(cert.as_slice()).key(key.as_slice()));
```

HTTP/2 is enabled automatically with Rustls.

## ACME (Let's Encrypt)

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["acme"] }
```

ACME is an extension trait on `TcpListener` — there is NO `AcmeListener::builder()` constructor. Use `.acme()` on the listener and chain config.

### HTTP-01 challenge

HTTP-01 needs port 80 open for challenge responses. `http01_challenge(&mut router)` mounts the challenge handler into your router automatically.

```rust
use salvo::prelude::*;

#[handler]
async fn hello() -> &'static str { "Hello Let's Encrypt" }

#[tokio::main]
async fn main() {
    let mut router = Router::new().get(hello);

    let listener = TcpListener::new("0.0.0.0:443")
        .acme()
        .cache_path("/tmp/letsencrypt")
        .add_domain("example.com")
        .http01_challenge(&mut router);

    // Bind 443 (TLS) + 80 (challenges) together.
    let acceptor = listener.join(TcpListener::new("0.0.0.0:80")).bind().await;
    Server::new(acceptor).serve(router).await;
}
```

### TLS-ALPN-01 challenge

No port 80 required; challenges run over the TLS handshake on 443.

```rust
let acceptor = TcpListener::new("0.0.0.0:443")
    .acme()
    .cache_path("/tmp/letsencrypt")
    .add_domain("example.com")
    // .tls_alpn01_challenge() is the default; call explicitly to be clear
    .bind()
    .await;
```

### Staging directory (testing)

Let's Encrypt staging avoids production rate limits while testing:

```rust
use salvo::conn::acme::{AcmeListener, LETS_ENCRYPT_STAGING};

let listener = TcpListener::new("0.0.0.0:443")
    .acme()
    .directory("letsencrypt-staging", LETS_ENCRYPT_STAGING)
    .cache_path("/tmp/letsencrypt-staging")
    .add_domain("example.com")
    .http01_challenge(&mut router);
```

Note: use `.directory(name, url)` — there is no `.directory_url(...)` method. Use `.http01_challenge(&mut router)` / `.tls_alpn01_challenge()` / `.dns01_challenge(solver)` — there is no `.challenge_type(...)` method.

### Multiple domains and contacts

```rust
.add_domain("example.com")
.add_domain("www.example.com")
.add_contact("mailto:admin@example.com")
```

Or pass a `Vec`:

```rust
.domains(vec!["example.com".into(), "www.example.com".into()])
.contacts(vec!["mailto:admin@example.com".into()])
```

### Key types

```rust
use salvo::conn::acme::KeyType;
.key_type(KeyType::EcdsaP256)  // default; also Rsa2048/4096, Ed25519, ...
```

## Force HTTP-to-HTTPS redirect

Use the built-in `ForceHttps` middleware (needs `force-https` feature) instead of writing a custom handler:

```toml
salvo = { version = "0.89.3", features = ["rustls", "force-https"] }
```

```rust
use salvo::prelude::*;

let service = Service::new(router).hoop(ForceHttps::new().https_port(443));

let acceptor = TcpListener::new("0.0.0.0:443")
    .rustls(tls_config)
    .join(TcpListener::new("0.0.0.0:80"))
    .bind()
    .await;
Server::new(acceptor).serve(service).await;
```

## HTTP/3 (QUIC)

```toml
salvo = { version = "0.89.3", features = ["quinn"] }
```

`QuinnListener` takes a quinn `ServerConfig` (built from `RustlsConfig`) and an address — NOT a `cert_path`/`key_path` builder.

```rust
use salvo::prelude::*;
use salvo::conn::rustls::{Keycert, RustlsConfig};

let config = RustlsConfig::new(
    Keycert::new()
        .cert(include_bytes!("../certs/cert.pem").as_slice())
        .key(include_bytes!("../certs/key.pem").as_slice())
);

let listener = TcpListener::new(("0.0.0.0", 443)).rustls(config.clone());
let acceptor = QuinnListener::new(config.build_quinn_config().unwrap(), ("0.0.0.0", 443))
    .join(listener)
    .bind()
    .await;
Server::new(acceptor).serve(router).await;
```

## Security headers (HSTS)

```rust
#[handler]
async fn hsts(_req: &mut Request, _depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    res.headers_mut().insert(
        "strict-transport-security",
        "max-age=31536000; includeSubDomains; preload".parse().unwrap(),
    );
    ctrl.call_next(_req, _depot, res).await;
}
```

## Related Skills

- **salvo-graceful-shutdown**: graceful shutdown for HTTPS servers
- **salvo-cors**: CORS and other security headers
