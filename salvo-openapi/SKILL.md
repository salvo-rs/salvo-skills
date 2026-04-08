---
name: salvo-openapi
description: Generate OpenAPI documentation automatically from Salvo handlers. Use for API documentation, Swagger UI, and API client generation.
version: 0.89.3
tags: [advanced, openapi, swagger, documentation]
---

# Salvo OpenAPI Integration

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["oapi"] }
serde = { version = "1", features = ["derive"] }
```

## `#[endpoint]` vs `#[handler]`

`#[handler]` is the plain Salvo handler. `#[endpoint]` registers the same handler
AND generates OpenAPI operation metadata. Use `#[endpoint]` on any route you want
documented.

## Auto-documentation rules

Documentation is only generated for things the macro can see at compile time:

- **Parameters**: use extractors (`PathParam<T>`, `QueryParam<T, REQUIRED>`, `JsonBody<T>`,
  `FormBody<T>`, or a struct deriving `ToParameters`). `req.param()` / `req.query()` are
  invisible to the generator.
- **Responses**: the return type must implement `EndpointOutRegister`. `Json<T>` (with
  `T: ToSchema`), `StatusCode`, `&'static str`, `String`, `StatusError`, and
  `Result<T, E>` where both impl `EndpointOutRegister` all work.

BAD (nothing documented):
```rust
#[endpoint]
async fn get_user(req: &mut Request) -> Json<User> {
    let id = req.param::<i64>("id").unwrap();
    let page = req.query::<i32>("page");
    Json(User { /* ... */ })
}
```

GOOD (params + success + error all documented):
```rust
#[endpoint]
async fn get_user(
    id: PathParam<i64>,
    page: QueryParam<i32, false>,
) -> Result<Json<User>, StatusError> { /* ... */ }
```

## Basic setup

```rust
use salvo::oapi::extract::*;
use salvo::prelude::*;

#[endpoint]
async fn hello(name: QueryParam<String, false>) -> String {
    format!("Hello, {}!", name.as_deref().unwrap_or("World"))
}

#[tokio::main]
async fn main() {
    let router = Router::new().push(Router::with_path("hello").get(hello));

    let doc = OpenApi::new("My API", "1.0.0").merge_router(&router);

    let router = router
        .unshift(doc.into_router("/api-doc/openapi.json"))
        .unshift(SwaggerUi::new("/api-doc/openapi.json").into_router("/swagger-ui"));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Extractors

`PathParam<T>` is always required. `QueryParam<T, REQUIRED>` takes a const bool
(defaults to `true`). Call `.into_inner()` (or `0` for `PathParam`) to get the value.

```rust
use salvo::oapi::extract::{PathParam, QueryParam, JsonBody};

#[endpoint]
async fn get_user(id: PathParam<i64>) -> String {
    format!("User ID: {}", id.into_inner())
}

#[endpoint]
async fn search(q: QueryParam<String, false>) -> String {
    format!("Search: {}", q.as_deref().unwrap_or(""))
}

#[endpoint]
async fn create_user(user: JsonBody<CreateUser>) -> StatusCode {
    let _u = user.into_inner();
    StatusCode::CREATED
}
```

## Schemas with `ToSchema`

Doc comments on fields become descriptions. Validation and examples use the
`#[salvo(schema(...))]` attribute.

```rust
use salvo::oapi::ToSchema;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, ToSchema)]
#[salvo(schema(example = json!({"id": 1, "name": "John", "email": "john@example.com"})))]
struct User {
    id: i64,
    /// User's full name
    #[salvo(schema(pattern = "^[a-zA-Z ]+$"))]
    name: String,
    #[salvo(schema(format = "email"))]
    email: String,
    #[salvo(schema(minimum = 1, maximum = 150))]
    age: Option<u8>,
}
```

## Query structs with `ToParameters`

```rust
use salvo::oapi::ToParameters;
use serde::Deserialize;

#[derive(Deserialize, ToParameters)]
struct Pagination {
    /// Page number
    #[salvo(parameter(default = 1, minimum = 1))]
    page: Option<u32>,
    /// Items per page
    #[salvo(parameter(default = 20, minimum = 1, maximum = 100))]
    per_page: Option<u32>,
}

#[endpoint]
async fn list_users(pagination: Pagination) -> Json<Vec<User>> {
    Json(vec![])
}
```

## Endpoint attributes

Supported inside `#[endpoint(...)]`: `operation_id`, `tags`, `summary`, `description`,
`request_body`, `responses`, `status_codes`, `parameters`, `security`, `deprecated`.

```rust
#[derive(Serialize, ToSchema)]
struct ErrorResponse { message: String }

#[endpoint(
    tags("users"),
    summary = "Get user by ID",
    description = "Returns a single user.",
    status_codes(200, 404),
    responses(
        (status_code = 200, description = "User found", body = User),
        (status_code = 404, description = "Not found", body = ErrorResponse),
    )
)]
async fn get_user(id: PathParam<i64>) -> Result<Json<User>, StatusError> {
    Ok(Json(User { /* ... */ }))
}
```

`status_codes(...)` filters the auto-generated responses to just the listed codes.
`responses(...)` adds explicit entries with custom schemas.

## OpenApi metadata

`OpenApi` itself only exposes `info()`, `servers()`, `security()`,
`add_security_scheme()`, `tags()`, and `merge_router()`. There is **no**
`description()` / `contact_name()` / `license_name()` on `OpenApi` — set them on
`Info`/`Contact`/`License` and pass via `.info(...)`.

```rust
use salvo::oapi::{OpenApi, Info, Contact, License};

let doc = OpenApi::new("My API", "1.0.0")
    .info(
        Info::new("My API", "1.0.0")
            .description("A comprehensive user management API")
            .contact(Contact::new().name("API Support").email("support@example.com"))
            .license(License::new("MIT")),
    )
    .merge_router(&router);
```

## Swagger UI

```rust
let router = router
    .unshift(doc.into_router("/api-doc/openapi.json"))
    .unshift(SwaggerUi::new("/api-doc/openapi.json").into_router("/swagger-ui"));
```

## Security schemes

```rust
use salvo::oapi::security::{Http, HttpAuthScheme, SecurityScheme};

let doc = OpenApi::new("My API", "1.0.0")
    .info(Info::new("My API", "1.0.0").description("API with authentication"))
    .add_security_scheme(
        "bearer_auth",
        SecurityScheme::Http(Http::new(HttpAuthScheme::Bearer)),
    )
    .merge_router(&router);

#[endpoint(security(("bearer_auth" = [])))]
async fn get_profile() -> &'static str { "Protected profile" }
```

## File upload

```rust
#[derive(Serialize, ToSchema)]
struct UploadResponse { filename: String, size: u64 }

#[endpoint(tags("files"), request_body(content = "multipart/form-data"))]
async fn upload_file(req: &mut Request) -> Result<Json<UploadResponse>, StatusError> {
    let file = req.file("file").await.ok_or_else(StatusError::bad_request)?;
    Ok(Json(UploadResponse {
        filename: file.name().unwrap_or("unnamed").to_string(),
        size: file.size(),
    }))
}
```

Because the body is raw multipart, the handler takes `&mut Request` directly. The
`request_body` attribute supplies the missing OpenAPI shape.

## Custom error types for rich docs

Implement `Writer` + `EndpointOutRegister` to document a custom error variant:

```rust
use salvo::oapi::{Components, EndpointOutRegister, Operation, ToSchema};
use salvo::prelude::*;
use serde::Serialize;

#[derive(Debug, Serialize, ToSchema)]
struct ApiError { code: i32, message: String }

impl Writer for ApiError {
    async fn write(self, _req: &mut Request, _depot: &mut Depot, res: &mut Response) {
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(Json(self));
    }
}

impl EndpointOutRegister for ApiError {
    fn register(components: &mut Components, operation: &mut Operation) {
        operation.responses.insert(
            "400".into(),
            salvo::oapi::Response::new("Bad Request")
                .add_content("application/json", Self::to_schema(components)),
        );
    }
}

#[endpoint]
async fn create_user(body: JsonBody<CreateUser>) -> Result<Json<User>, ApiError> {
    /* ... */
}
```

## Related Skills

- **salvo-data-extraction**: Extract parameters for documentation
- **salvo-error-handling**: Document error responses
- **salvo-auth**: Document security schemes
