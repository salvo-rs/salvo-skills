---
name: salvo-data-extraction
description: Extract and validate data from requests including JSON, forms, query parameters, and path parameters. Use for handling user input and API payloads.
version: 0.89.3
tags: [data, extraction, json, form, query]
---

# Salvo Data Extraction

## Manual Extraction

```rust
use salvo::prelude::*;

#[handler]
async fn handler(req: &mut Request) -> String {
    let name = req.query::<String>("name").unwrap_or_default();
    let id = req.param::<i64>("id").unwrap();          // route: /users/{id}
    let auth = req.header::<String>("Authorization");

    let body: UserData = req.parse_json().await.unwrap();
    let form: LoginForm = req.parse_form().await.unwrap();
    let pagination: Pagination = req.parse_queries().unwrap();

    String::new()
}
```

`param`, `query`, `header` return `Option<T>`. `parse_*` return `ParseResult<T>`. Use `try_param`/`try_query`/`try_header` for error details.

## JsonBody Extractor

```rust
use salvo::prelude::*;
use serde::Deserialize;

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[handler]
async fn create_user(body: JsonBody<CreateUser>) -> StatusCode {
    let user = body.into_inner();
    StatusCode::CREATED
}
```

Also available: `FormBody<T>`, `QueryParam<T, const REQUIRED: bool>`, `PathParam<T>`, `HeaderParam<T, const REQUIRED: bool>`, `CookieParam<T, const REQUIRED: bool>`.

## Extractible Derive

Auto-extracts a struct from one or more request sources.

Valid sources: `body`, `query`, `param`, `header`, `cookie`, `depot`.

```rust
use salvo::prelude::*;
use serde::Deserialize;

#[derive(Extractible, Deserialize, Debug)]
#[salvo(extract(default_source(from = "body")))]
struct CreateUser {
    name: String,
    email: String,
}

#[handler]
async fn create_user(user: CreateUser) -> String {
    format!("{user:?}")
}
```

Query source:

```rust
#[derive(Extractible, Deserialize)]
#[salvo(extract(default_source(from = "query")))]
struct Pagination {
    page: Option<u32>,
    per_page: Option<u32>,
}
```

Form body (URL-encoded or multipart):

```rust
#[derive(Extractible, Deserialize)]
#[salvo(extract(default_source(from = "body"), default_format = "form"))]
struct LoginForm {
    username: String,
    password: String,
}
```

## Mixed Sources

Per-field source overrides default:

```rust
#[derive(Extractible, Deserialize)]
struct UpdateUser {
    #[salvo(extract(source(from = "param")))]
    id: i64,
    #[salvo(extract(source(from = "body")))]
    name: String,
    #[salvo(extract(source(from = "body")))]
    email: String,
}

#[handler]
async fn update_user(data: UpdateUser) -> StatusCode {
    StatusCode::OK
}
```

## Depot Extraction

Read middleware-injected state as typed struct fields.

```rust
use salvo::prelude::*;
use serde::{Deserialize, Serialize};

#[handler]
async fn inject_user(depot: &mut Depot) {
    depot.insert("user_id", 123i64);
    depot.insert("username", "alice".to_string());
    depot.insert("is_admin", true);
}

#[derive(Serialize, Deserialize, Extractible, Debug)]
#[salvo(extract(default_source(from = "depot")))]
struct UserContext {
    user_id: i64,
    username: String,
    is_admin: bool,
}

#[handler]
async fn protected(user: UserContext) -> String {
    format!("Hello {}, id {}", user.username, user.user_id)
}

let router = Router::new()
    .hoop(inject_user)
    .push(Router::with_path("protected").get(protected));
```

Depot-extraction supports: `String`, `&'static str`, all signed/unsigned integer primitives, `f32`, `f64`, `bool`. For richer types, fetch manually with `depot.obtain::<T>()`.

Combining depot with other sources:

```rust
#[derive(Serialize, Deserialize, Extractible, Debug)]
struct RequestData {
    #[salvo(extract(source(from = "depot")))]
    user_id: i64,
    #[salvo(extract(source(from = "query")))]
    page: i64,
    #[salvo(extract(source(from = "body")))]
    content: String,
}
```

## Validation with `validator`

```rust
use salvo::prelude::*;
use serde::Deserialize;
use validator::Validate;

#[derive(Extractible, Deserialize, Validate)]
#[salvo(extract(default_source(from = "body")))]
struct CreateUser {
    #[validate(length(min = 1, max = 100))]
    name: String,
    #[validate(email)]
    email: String,
    #[validate(range(min = 18, max = 120))]
    age: u8,
}

#[handler]
async fn create_user(user: CreateUser) -> Result<StatusCode, StatusError> {
    user.validate()
        .map_err(|e| StatusError::bad_request().brief(e.to_string()))?;
    Ok(StatusCode::CREATED)
}
```

Custom validator:

```rust
use validator::{Validate, ValidationError};

fn validate_username(username: &str) -> Result<(), ValidationError> {
    if username.contains("admin") {
        return Err(ValidationError::new("forbidden_username"));
    }
    Ok(())
}

#[derive(Deserialize, Validate)]
struct User {
    #[validate(custom(function = "validate_username"))]
    username: String,
}
```

## Nested Structures

```rust
#[derive(Deserialize)]
struct Address { street: String, city: String, country: String }

#[derive(Extractible, Deserialize)]
#[salvo(extract(default_source(from = "body")))]
struct CreateUserWithAddress {
    name: String,
    email: String,
    address: Address,
}
```

## Error Handling on Manual Parse

```rust
#[handler]
async fn create_user(req: &mut Request, res: &mut Response) {
    match req.parse_json::<CreateUser>().await {
        Ok(user) => res.render(Json(serde_json::json!({ "ok": true }))),
        Err(e) => {
            res.status_code(StatusCode::BAD_REQUEST);
            res.render(Json(serde_json::json!({ "error": e.to_string() })));
        }
    }
}
```

## Related Skills

- **salvo-openapi**: Auto-generate docs for extracted parameters
- **salvo-file-handling**: Multipart uploads and file fields
- **salvo-testing**: Test extractors with TestClient
