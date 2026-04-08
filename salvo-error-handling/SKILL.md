---
name: salvo-error-handling
description: Handle errors gracefully with custom error types, status codes, and error pages. Use for building robust APIs with proper error responses.
version: 0.89.3
tags: [core, error-handling, status-code]
---

# Salvo Error Handling

## StatusError

Return a `StatusError` from any handler that is `Result<T, StatusError>`:

```rust
use salvo::prelude::*;

#[handler]
async fn get_user(req: &mut Request) -> Result<Json<User>, StatusError> {
    let id = req.param::<i64>("id")
        .ok_or_else(|| StatusError::bad_request().brief("Missing user ID"))?;

    find_user(id).await
        .ok_or_else(|| StatusError::not_found().brief("User not found"))
        .map(Json)
}
```

Constructors match HTTP status names: `bad_request()` (400), `unauthorized()` (401), `forbidden()` (403), `not_found()` (404), `method_not_allowed()` (405), `conflict()` (409), `unprocessable_entity()` (422), `internal_server_error()` (500), `not_implemented()` (501), `bad_gateway()` (502), `service_unavailable()` (503), plus most other RFC codes.

Chainable setters: `.brief(...)`, `.detail(...)`, `.cause(err)`. Note: no `not_modified()` — set 304 directly via `res.status_code(StatusCode::NOT_MODIFIED)`.

## anyhow / eyre

Enable the feature, then return `anyhow::Error` / `eyre::Report` directly:

```toml
salvo = { version = "0.89.3", features = ["anyhow", "eyre"] }
```

```rust
#[handler]
async fn process() -> anyhow::Result<String> {
    let data = fetch_data().await.context("fetch failed")?;
    Ok(process_data(data)?)
}
```

Both are converted to `500 Internal Server Error` responses.

## Custom errors with Writer

Implement `Writer` for full control over the response. Use `thiserror` for ergonomic definitions:

```rust
use salvo::prelude::*;
use thiserror::Error;

#[derive(Error, Debug)]
enum ApiError {
    #[error("not found: {0}")]
    NotFound(String),
    #[error("validation: {0}")]
    Validation(String),
    #[error(transparent)]
    Database(#[from] sqlx::Error),
}

#[async_trait]
impl Writer for ApiError {
    async fn write(self, _req: &mut Request, _depot: &mut Depot, res: &mut Response) {
        let status = match &self {
            ApiError::NotFound(_) => StatusCode::NOT_FOUND,
            ApiError::Validation(_) => StatusCode::BAD_REQUEST,
            ApiError::Database(_) => StatusCode::INTERNAL_SERVER_ERROR,
        };
        res.status_code(status);
        res.render(Json(serde_json::json!({
            "error": self.to_string(),
            "code": status.as_u16(),
        })));
    }
}

#[handler]
async fn get_user(req: &mut Request) -> Result<Json<User>, ApiError> {
    let id = req.param::<i64>("id")
        .ok_or_else(|| ApiError::Validation("missing id".into()))?;
    let user = find_user(id).await?
        .ok_or_else(|| ApiError::NotFound(format!("user {id}")))?;
    Ok(Json(user))
}
```

## CatchPanic

Convert panics into 500 responses. Register as the FIRST middleware so it wraps everything downstream. Lives in `salvo::catch_panic` (feature `catch-panic`, enabled by default in `full`):

```rust
use salvo::catch_panic::CatchPanic;

let router = Router::new()
    .hoop(CatchPanic::new())
    .get(handler);
```

## Custom error pages with Catcher

When a response has an error status code (4xx/5xx) and empty body, Salvo runs the service's `Catcher`. Add custom handlers via `hoop`; call `ctrl.skip_rest()` to stop the chain. The default `Catcher` performs content negotiation (HTML / JSON / XML / text) from the `Accept` header.

```rust
use salvo::prelude::*;
use salvo::catcher::Catcher;

#[handler]
async fn handle_404(res: &mut Response, ctrl: &mut FlowCtrl) {
    if res.status_code == Some(StatusCode::NOT_FOUND) {
        res.render("Custom 404");
        ctrl.skip_rest();
    }
}

let service = Service::new(router)
    .catcher(Catcher::default().hoop(handle_404));
```

Control error detail exposure via env var `SALVO_STATUS_ERROR`:
- `force_detail,force_cause` — always show (debugging only)
- `debug_detail,debug_cause` — only in debug builds
- `never_detail,never_cause` — never show

## Error logging

Log internal errors but return sanitized messages to clients:

```rust
use tracing::error;

#[handler]
async fn handler(req: &mut Request) -> Result<String, StatusError> {
    process(req).await.map_err(|e| {
        error!(error = %e, path = %req.uri().path(), "request failed");
        StatusError::internal_server_error().brief("Request failed")
    })
}
```

## Gotchas

- `StatusError` has no `not_modified()` / `continue_()` / other 1xx–3xx constructors. Only 4xx and 5xx. Use `res.status_code(...)` for others.
- `Catcher` only runs when the body is empty and status is an error. Writing any body skips it.
- `depot.obtain::<T>()` returns `Result<&T, _>`, not `Option`.

## Related Skills

- **salvo-openapi**: Document error responses in OpenAPI.
- **salvo-logging**: Log and trace errors.
- **salvo-testing**: Test error responses.
