---
name: salvo-session
description: Implement session management for user state persistence. Use for login systems, shopping carts, and user preferences.
version: 0.89.3
tags: [security, session, cookie, login]
---

# Salvo Session Management

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["session"] }
```

## Basic Setup

Secret key must be at least 64 bytes; `HandlerBuilder::new` panics otherwise (use `try_new` for a `Result`).

```rust
use salvo::prelude::*;
use salvo::session::{CookieStore, Session, SessionDepotExt, SessionHandler};

#[tokio::main]
async fn main() {
    let session_handler = SessionHandler::builder(
        CookieStore::new(),
        b"secretabsecretabsecretabsecretabsecretabsecretabsecretabsecretab",
    )
    .build()
    .unwrap();

    let router = Router::new()
        .hoop(session_handler)
        .get(home)
        .push(Router::with_path("login").get(login).post(login))
        .push(Router::with_path("logout").get(logout));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Session Operations

Create a new session via `Session::new()` and store it with `depot.set_session(session)`. Mutate an existing session through `depot.session_mut()`.

```rust
#[handler]
async fn login(req: &mut Request, depot: &mut Depot, res: &mut Response) {
    let username = req.form::<String>("username").await.unwrap_or_default();
    let mut session = Session::new();
    session.insert("username", username).unwrap();
    session.insert("logged_in", true).unwrap();
    depot.set_session(session);
    res.render(Redirect::other("/"));
}

#[handler]
async fn home(depot: &mut Depot, res: &mut Response) {
    if let Some(session) = depot.session_mut()
        && let Some(username) = session.get::<String>("username")
    {
        res.render(Text::Plain(format!("Hello, {username}!")));
    } else {
        res.render(Text::Plain("Please login"));
    }
}

#[handler]
async fn logout(depot: &mut Depot, res: &mut Response) {
    if let Some(session) = depot.session_mut() {
        session.remove("username");
        // session.clear(); // remove everything
    }
    res.render(Redirect::other("/"));
}
```

Counter pattern:

```rust
let visits: i32 = session.get("visits").unwrap_or(0);
session.insert("visits", visits + 1).unwrap();
```

## Stores

- `CookieStore::new()` — encrypts session data into a cookie. No external dependencies, limited size.
- `MemoryStore::new()` — in-process, fast, lost on restart. Not suitable for multi-instance deployments.

Both plug into the same builder:

```rust
use salvo::session::MemoryStore;

SessionHandler::builder(MemoryStore::new(), secret).build().unwrap();
```

## Configuration

```rust
use std::time::Duration;
use salvo::http::cookie::SameSite;
use salvo::session::{CookieStore, SessionHandler};

let handler = SessionHandler::builder(CookieStore::new(), secret)
    .session_ttl(Some(Duration::from_secs(3600)))
    .cookie_name("session_id")
    .cookie_path("/")
    .cookie_domain("example.com")
    .same_site_policy(SameSite::Strict)
    .build()
    .unwrap();
```

Available builder methods: `session_ttl`, `cookie_name`, `cookie_path`, `cookie_domain`, `same_site_policy`, `save_unchanged`, `fallback_keys`, `add_fallback_key`.

**Gotcha:** there is no `cookie_http_only` or `cookie_secure` builder. `HttpOnly` is always set to `true`, and `Secure` is toggled on automatically when the request is served over HTTPS. The `SameSite` default is `Lax`.

## Session with Authentication Middleware

```rust
#[handler]
async fn require_login(depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let logged_in = depot.session_mut()
        .and_then(|s| s.get::<bool>("logged_in"))
        .unwrap_or(false);
    if !logged_in {
        res.render(Redirect::other("/login"));
        ctrl.skip_rest();
    }
}

#[handler]
async fn require_admin(depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let is_admin = depot.session_mut()
        .and_then(|s| s.get::<String>("role"))
        .map(|r| r == "admin")
        .unwrap_or(false);
    if !is_admin {
        res.status_code(StatusCode::FORBIDDEN);
        res.render("Admin access required");
        ctrl.skip_rest();
    }
}

let router = Router::new()
    .hoop(session_handler)
    .push(Router::with_path("dashboard").hoop(require_login).get(dashboard))
    .push(Router::with_path("admin").hoop(require_login).hoop(require_admin).get(admin_panel));
```

## Shopping Cart Example

```rust
use serde::{Deserialize, Serialize};

#[derive(Clone, Serialize, Deserialize)]
struct CartItem { product_id: u32, name: String, quantity: u32, price: f64 }

#[handler]
async fn add_to_cart(req: &mut Request, depot: &mut Depot, res: &mut Response) {
    let product_id: u32 = req.param("id").unwrap();
    let session = depot.session_mut().unwrap();
    let mut cart: Vec<CartItem> = session.get("cart").unwrap_or_default();
    if let Some(item) = cart.iter_mut().find(|i| i.product_id == product_id) {
        item.quantity += 1;
    } else {
        cart.push(CartItem { product_id, name: format!("Product {product_id}"), quantity: 1, price: 9.99 });
    }
    session.insert("cart", cart).unwrap();
    res.render(Redirect::other("/cart"));
}

#[handler]
async fn view_cart(depot: &mut Depot, res: &mut Response) {
    let cart: Vec<CartItem> = depot.session_mut()
        .and_then(|s| s.get("cart"))
        .unwrap_or_default();
    let total: f64 = cart.iter().map(|i| i.price * i.quantity as f64).sum();
    res.render(Json(serde_json::json!({ "items": cart, "total": total })));
}
```

## Salvo-specific Notes

- Session value types must be `Serialize + Deserialize`.
- Regenerate on login: create a fresh `Session::new()` and `set_session` to prevent fixation.
- For multi-instance deployments, use a shared store (see `salvo-session` extras / community stores) rather than `MemoryStore`.
- `CookieStore` has a cookie size limit (~4 KB); keep serialized data small or switch stores.

## Related Skills

- **salvo-auth**: Authentication using sessions
- **salvo-csrf**: CSRF protection with session store
- **salvo-flash**: Flash messages using sessions
