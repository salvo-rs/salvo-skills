---
name: salvo-flash
description: Implement flash messages for one-time notifications across redirects. Use for success/error messages after form submissions.
version: 0.89.3
tags: [advanced, flash-messages, notifications]
---

# Salvo Flash Messages

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["flash"] }
```

`FlashHandler` middleware stores messages between requests. The set
(outgoing) is written by the current handler, the previous request's messages
are loaded into the "incoming" slot before handlers run, and the old entries
are cleared automatically after display.

## Cookie store

```rust
use std::fmt::Write;
use salvo::flash::{CookieStore, FlashDepotExt};
use salvo::prelude::*;

#[handler]
async fn set_flash(depot: &mut Depot, res: &mut Response) {
    let flash = depot.outgoing_flash_mut();
    flash.info("Operation completed successfully!");
    flash.debug("Debug information here");
    res.render(Redirect::other("/show"));
}

#[handler]
async fn show_flash(depot: &mut Depot, res: &mut Response) {
    let mut output = String::new();
    if let Some(flash) = depot.incoming_flash() {
        for msg in flash.iter() {
            writeln!(output, "[{}] {}", msg.level, msg.value).unwrap();
        }
    }
    if output.is_empty() { output.push_str("No flash messages"); }
    res.render(Text::Plain(output));
}

#[tokio::main]
async fn main() {
    let router = Router::new()
        .hoop(CookieStore::new().into_handler())
        .push(Router::with_path("set").get(set_flash))
        .push(Router::with_path("show").get(show_flash));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

Note `incoming_flash` takes `&mut Depot`, so any helper that reads flash must
also take `&mut Depot`.

## Message levels

Five levels on `Flash` (chainable `&mut Self` setters):

```rust
let flash = depot.outgoing_flash_mut();
flash.debug("...").info("...").success("...").warning("...").error("...");
```

`FlashLevel` is an **enum**, not a `String`. Match on the variants (it implements
`Display` / `Debug` as the lowercase name), or call `.to_str()`:

```rust
use salvo::flash::FlashLevel;

for msg in flash.iter() {
    let class = match msg.level {
        FlashLevel::Success => "alert-success",
        FlashLevel::Error   => "alert-error",
        FlashLevel::Warning => "alert-warning",
        FlashLevel::Info    => "alert-info",
        FlashLevel::Debug   => "alert-debug",
    };
    println!("<div class=\"{class}\">{}</div>", msg.value);
}
```

`msg.level.as_str()` does NOT exist — use `to_str()` or the `Display` impl.

## `CookieStore` configuration

Builders on `CookieStore` (all `#[must_use]`): `name`, `max_age`, `same_site`,
`http_only`, `path`. Defaults: name `"salvo.flash"`, `max_age = 60s`,
`SameSite::Lax`, `http_only = true`, `path = "/"`.

```rust
use salvo::http::cookie::{time::Duration, SameSite};

let store = CookieStore::new()
    .name("app.flash")
    .max_age(Duration::seconds(300))
    .same_site(SameSite::Strict)
    .path("/app");

let router = Router::new().hoop(store.into_handler());
```

## Session store

When messages are too large for a cookie, use `SessionStore` (requires a
`SessionHandler` in front of it).

```rust
use salvo::flash::{FlashDepotExt, SessionStore};
use salvo::session::{CookieStore as SessionCookieStore, SessionHandler};
use salvo::prelude::*;

#[tokio::main]
async fn main() {
    let session = SessionHandler::builder(
        SessionCookieStore::new(),
        b"secretabsecretabsecretabsecretabsecretabsecretabsecretabsecretab",
    )
    .build()
    .unwrap();

    let router = Router::new()
        .hoop(session)                         // session first
        .hoop(SessionStore::new().into_handler())  // then flash
        .push(Router::with_path("set").get(set_flash))
        .push(Router::with_path("show").get(show_flash));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Filtering by minimum level

`FlashHandler::minimum_level(level)` drops outgoing messages below the given
level (useful in production to suppress debug noise):

```rust
use salvo::flash::{CookieStore, FlashHandler, FlashLevel};

let mut handler = FlashHandler::new(CookieStore::new());
handler.minimum_level(FlashLevel::Warning);
let router = Router::new().hoop(handler);
```

## Form submission (Post-Redirect-Get)

```rust
use salvo::flash::{CookieStore, FlashDepotExt, FlashLevel};
use salvo::prelude::*;
use serde::Deserialize;

#[derive(Deserialize)]
struct ContactForm { name: String, email: String, message: String }

#[handler]
async fn show_form(depot: &mut Depot, res: &mut Response) {
    let mut alerts = String::new();
    if let Some(flash) = depot.incoming_flash() {
        for msg in flash.iter() {
            let class = match msg.level {
                FlashLevel::Success => "alert-success",
                FlashLevel::Error   => "alert-error",
                FlashLevel::Warning => "alert-warning",
                _ => "alert-info",
            };
            alerts.push_str(&format!(r#"<div class="{class}">{}</div>"#, msg.value));
        }
    }
    res.render(Text::Html(format!(r#"
        <!DOCTYPE html><html><body>
        {alerts}
        <form method="post" action="/contact">
            <input name="name" required />
            <input type="email" name="email" required />
            <textarea name="message" required></textarea>
            <button type="submit">Send</button>
        </form>
        </body></html>
    "#)));
}

#[handler]
async fn handle_form(req: &mut Request, depot: &mut Depot, res: &mut Response) {
    match req.parse_form::<ContactForm>().await {
        Ok(form) => {
            // ... process form ...
            let _ = form;
            depot.outgoing_flash_mut().success("Thank you! Your message has been sent.");
        }
        Err(e) => {
            depot.outgoing_flash_mut().error(format!("Error: {e}"));
        }
    }
    res.render(Redirect::other("/contact"));
}

#[tokio::main]
async fn main() {
    let router = Router::new()
        .hoop(CookieStore::new().into_handler())
        .push(Router::with_path("contact").get(show_form).post(handle_form));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## JSON rendering

```rust
fn flash_to_json(depot: &mut Depot) -> serde_json::Value {
    let messages: Vec<_> = depot
        .incoming_flash()
        .map(|flash| {
            flash
                .iter()
                .map(|m| serde_json::json!({ "level": m.level.to_str(), "message": m.value }))
                .collect()
        })
        .unwrap_or_default();
    serde_json::json!({ "flash": messages })
}
```

## Related Skills

- **salvo-session**: Required backend for `SessionStore`
