---
name: salvo-csrf
description: Implement CSRF (Cross-Site Request Forgery) protection using cookie or session storage. Use for protecting forms and state-changing endpoints.
version: 0.89.3
tags: [security, csrf, protection]
---

# Salvo CSRF Protection

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["csrf"] }
```

## Ciphers

| Function prefix | Key type | Notes |
|-----------------|----------|-------|
| `bcrypt_*` | — | No key. Slowest (bcrypt hashing). |
| `hmac_*` | `[u8; 32]` | Fast. |
| `aes_gcm_*` | `[u8; 32]` | Authenticated encryption. |
| `ccp_*` | `[u8; 32]` | ChaCha20Poly1305. |

Each cipher pairs with a store: `*_cookie_csrf(...)` or `*_session_csrf(...)`. Session variants require a `SessionHandler` installed earlier in the chain.

## Basic: Cookie Store + Form Token

```rust
use salvo::csrf::{bcrypt_cookie_csrf, CsrfDepotExt, FormFinder};
use salvo::prelude::*;
use serde::Deserialize;

#[derive(Deserialize)]
struct FormData {
    csrf_token: String,
    message: String,
}

#[handler]
async fn show_form(depot: &mut Depot, res: &mut Response) {
    let token = depot.csrf_token().unwrap_or_default();
    res.render(Text::Html(format!(
        r#"<form method="post">
            <input type="hidden" name="csrf_token" value="{token}" />
            <input type="text" name="message" />
            <button type="submit">Submit</button>
        </form>"#
    )));
}

#[handler]
async fn handle_form(req: &mut Request, res: &mut Response) {
    let data = req.parse_form::<FormData>().await.unwrap();
    res.render(format!("Message received: {}", data.message));
}

#[tokio::main]
async fn main() {
    let csrf = bcrypt_cookie_csrf(FormFinder::new("csrf_token"));
    let router = Router::new().hoop(csrf).get(show_form).post(handle_form);
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Cipher Variants

```rust
use salvo::csrf::{
    bcrypt_cookie_csrf, hmac_cookie_csrf, aes_gcm_cookie_csrf, ccp_cookie_csrf,
    bcrypt_session_csrf, hmac_session_csrf, aes_gcm_session_csrf, ccp_session_csrf,
    FormFinder,
};

let finder = FormFinder::new("csrf_token");
let key = *b"01234567012345670123456701234567"; // 32 bytes

let _ = bcrypt_cookie_csrf(finder.clone());
let _ = hmac_cookie_csrf(key, finder.clone());
let _ = aes_gcm_cookie_csrf(key, finder.clone());
let _ = ccp_cookie_csrf(key, finder.clone());
```

## Session Store

Session-backed CSRF needs a `SessionHandler` hooped before the CSRF handler:

```rust
use salvo::csrf::{bcrypt_session_csrf, FormFinder};
use salvo::session::{CookieStore, SessionHandler};
use salvo::prelude::*;

#[tokio::main]
async fn main() {
    let session = SessionHandler::builder(
        CookieStore::new(),
        b"secretabsecretabsecretabsecretabsecretabsecretabsecretabsecretab",
    ).build().unwrap();

    let csrf = bcrypt_session_csrf(FormFinder::new("csrf_token"));

    let router = Router::new()
        .hoop(session) // must come first
        .hoop(csrf)
        .get(show_form)
        .post(handle_form);

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Token Finders

```rust
use salvo::csrf::{FormFinder, HeaderFinder, JsonFinder};

FormFinder::new("csrf_token");          // application/x-www-form-urlencoded body
HeaderFinder::new("x-csrf-token");      // HTTP header
JsonFinder::new("csrf_token");          // JSON body field
```

**Only** `FormFinder`, `HeaderFinder`, and `JsonFinder` exist in `salvo::csrf`. There is no `QueryFinder`. Use `Csrf::add_finder` to accept the token from multiple locations.

## Retrieving the Token

```rust
use salvo::csrf::CsrfDepotExt;

#[handler]
async fn show_form(depot: &mut Depot, res: &mut Response) {
    let token = depot.csrf_token().unwrap_or_default();
    res.render(Text::Html(format!(
        r#"<form method="post"><input type="hidden" name="csrf_token" value="{token}" /></form>"#
    )));
}
```

A new token is generated on every request; re-render it each time.

## Scoped CSRF Middleware

```rust
let form_finder = FormFinder::new("csrf_token");
let bcrypt_csrf = bcrypt_cookie_csrf(form_finder.clone());
let hmac_csrf = hmac_cookie_csrf(*b"01234567012345670123456701234567", form_finder);

let router = Router::new()
    .push(Router::with_hoop(bcrypt_csrf).path("forms").get(show_form).post(handle_form))
    .push(Router::with_hoop(hmac_csrf).path("api").get(get_token).post(api_handler));
```

## AJAX / Header-based

```rust
use salvo::csrf::{HeaderFinder, hmac_cookie_csrf};

let csrf = hmac_cookie_csrf(
    *b"01234567012345670123456701234567",
    HeaderFinder::new("x-csrf-token"),
);
```

Client:

```js
fetch('/api', {
    method: 'POST',
    headers: { 'X-CSRF-Token': token },
    body: JSON.stringify(data),
});
```

## Salvo-specific Notes

- CSRF middleware only checks "unsafe" methods (POST/PUT/PATCH/DELETE); GET/HEAD/OPTIONS bypass validation and refresh the token.
- Session-backed variants require `SessionHandler` hooped **before** the CSRF handler, otherwise the store cannot locate a session.
- When pairing with `salvo-cors`, CSRF headers (e.g. `x-csrf-token`) must be listed in `allow_headers`.

## Related Skills

- **salvo-cors**: CORS configuration for cross-origin requests
- **salvo-session**: CSRF with session-based storage
