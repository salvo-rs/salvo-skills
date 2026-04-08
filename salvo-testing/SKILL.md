---
name: salvo-testing
description: Write unit and integration tests for Salvo applications using TestClient. Use for testing handlers, middleware, and API endpoints.
version: 0.89.3
tags: [advanced, testing, test-client, integration]
---

# Salvo Testing

```toml
[dev-dependencies]
salvo = { version = "0.89.3", features = ["test"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

## TestClient basics

`TestClient` builds a `RequestBuilder` for each HTTP method, then `send(target)` runs it against a `Service`, `Router`, or `Handler` — no TCP bind needed. The URL scheme/host is a placeholder; only the path and query are routed.

```rust
use salvo::prelude::*;
use salvo::test::{ResponseExt, TestClient};

#[handler]
async fn hello() -> &'static str { "Hello World" }

#[tokio::test]
async fn test_hello() {
    let router = Router::new().get(hello);
    let service = Service::new(router);

    let content = TestClient::get("http://127.0.0.1:8080/")
        .send(&service).await
        .take_string().await.unwrap();

    assert_eq!(content, "Hello World");
}
```

Available methods: `get`, `post`, `put`, `delete`, `patch`, `head`, `options`, `trace`.

## Builder methods

| Method | Purpose |
| --- | --- |
| `.query(k, v)` / `.queries(pairs)` | Append query params |
| `.add_header(name, value, overwrite)` | Set a header |
| `.basic_auth(user, Some(pass))` | HTTP Basic auth |
| `.bearer_auth(token)` | `Authorization: Bearer ...` |
| `.json(&value)` | Serialize + set JSON body |
| `.raw_json(string)` | Pre-serialized JSON body |
| `.form(&value)` | Serialize + set urlencoded form |
| `.raw_form(string)` | Pre-serialized form body |
| `.text(s)` / `.bytes(vec)` / `.body(body)` | Raw bodies |
| `.send(target)` | Run and return `Response` |

`target` can be `&Service`, `Router`, `Arc<Router>`, or any `Handler`.

## Response helpers

`ResponseExt` adds async consumers that take `&mut Response`:

- `take_string()` — decode body as string (honors charset)
- `take_json::<T>()` — deserialize body
- `take_bytes(content_type)` — raw bytes

Status: `res.status_code` (an `Option<StatusCode>` field, not a method).

## Path params

```rust
#[handler]
async fn show_user(req: &mut Request) -> String {
    let id = req.param::<i64>("id").unwrap();
    format!("User ID: {id}")
}

#[tokio::test]
async fn test_show_user() {
    let router = Router::new().push(Router::with_path("users/{id}").get(show_user));
    let content = TestClient::get("http://127.0.0.1:8080/users/123")
        .send(&Service::new(router)).await
        .take_string().await.unwrap();
    assert_eq!(content, "User ID: 123");
}
```

Path syntax uses `{name}`.

## JSON request/response

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, PartialEq, Debug)]
struct User { id: i64, name: String }

#[handler]
async fn create(req: &mut Request) -> Result<Json<User>, StatusError> {
    let user: User = req.parse_json().await
        .map_err(|_| StatusError::bad_request())?;
    Ok(Json(user))
}

#[tokio::test]
async fn test_create() {
    let router = Router::new().post(create);
    let user = TestClient::post("http://127.0.0.1:8080/")
        .json(&serde_json::json!({ "id": 1, "name": "Alice" }))
        .send(&Service::new(router)).await
        .take_json::<User>().await.unwrap();
    assert_eq!(user.name, "Alice");
}
```

## Headers and auth

```rust
#[handler]
async fn protected(req: &mut Request) -> Result<&'static str, StatusError> {
    match req.header::<String>("authorization").as_deref() {
        Some("Bearer valid") => Ok("ok"),
        _ => Err(StatusError::unauthorized()),
    }
}

#[tokio::test]
async fn test_auth() {
    let router = Router::new().get(protected);
    let service = Service::new(router);

    let res = TestClient::get("http://127.0.0.1:8080/")
        .bearer_auth("valid")
        .send(&service).await;
    assert_eq!(res.status_code, Some(StatusCode::OK));

    let res = TestClient::get("http://127.0.0.1:8080/")
        .send(&service).await;
    assert_eq!(res.status_code, Some(StatusCode::UNAUTHORIZED));
}
```

## Middleware and Depot

```rust
#[handler]
async fn setup(depot: &mut Depot) { depot.insert("flag", true); }

#[handler]
async fn read(depot: &mut Depot) -> String {
    let flag = *depot.get::<bool>("flag").unwrap();
    format!("flag={flag}")
}

#[tokio::test]
async fn test_depot() {
    let router = Router::new().hoop(setup).get(read);
    let content = TestClient::get("http://127.0.0.1:8080/")
        .send(&Service::new(router)).await
        .take_string().await.unwrap();
    assert_eq!(content, "flag=true");
}
```

Note: `depot.get::<T>(key)` returns `Result<&T, _>`, not `Option`. Unwrap and deref.

## Form data

```rust
let res = TestClient::post("http://127.0.0.1:8080/")
    .form(&[("name", "Alice"), ("email", "alice@example.com")])
    .send(&service).await;
```

## Full CRUD flow

```rust
#[tokio::test]
async fn test_crud() {
    let service = Service::new(create_router());

    let res = TestClient::post("http://127.0.0.1:8080/users")
        .json(&serde_json::json!({"name": "Alice"}))
        .send(&service).await;
    assert_eq!(res.status_code, Some(StatusCode::CREATED));

    let user = TestClient::get("http://127.0.0.1:8080/users/1")
        .send(&service).await
        .take_json::<User>().await.unwrap();
    assert_eq!(user.name, "Alice");

    let res = TestClient::delete("http://127.0.0.1:8080/users/1")
        .send(&service).await;
    assert_eq!(res.status_code, Some(StatusCode::NO_CONTENT));
}
```

## Gotchas

- `send()` takes the target by value for `Router` / `Handler`, but by `&Service` for a service. Build a `Service` once if you reuse it across requests.
- `res.status_code` is a field (`Option<StatusCode>`), not a method call.
- The URL host/port in `TestClient::get(url)` is ignored by routing — only the path and query matter.
- `take_string`/`take_json` consume the body; a second call returns empty.

## Related Skills

- **salvo-error-handling**: Assert `StatusError` responses map to the expected codes.
- **salvo-auth**: Use `.bearer_auth()` / `.basic_auth()` for protected endpoints.
- **salvo-database**: Integration-test real repositories by passing a test pool via `affix_state`.
