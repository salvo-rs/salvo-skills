---
name: salvo-openapi
description: Generate OpenAPI documentation automatically from Salvo handlers. Use for API documentation, Swagger UI, and API client generation.
---

# Salvo OpenAPI Integration

This skill helps generate OpenAPI 3.0 documentation from Salvo applications.

## Setup

Add dependencies:

```toml
[dependencies]
salvo = { version = "0.76", features = ["oapi"] }
```

## Basic Usage

Use `#[endpoint]` instead of `#[handler]`:

```rust
use salvo::oapi::extract::*;
use salvo::prelude::*;

#[endpoint]
async fn hello() -> &'static str {
    "Hello World"
}
```

## Request Body Documentation

```rust
use serde::{Deserialize, Serialize};
use salvo::oapi::ToSchema;

#[derive(Deserialize, ToSchema)]
struct CreateUser {
    /// User's full name
    name: String,
    /// User's email address
    #[salvo(schema(example = "user@example.com"))]
    email: String,
}

#[endpoint]
async fn create_user(body: JsonBody<CreateUser>) -> StatusCode {
    StatusCode::CREATED
}
```

## Response Documentation

```rust
#[derive(Serialize, ToSchema)]
struct User {
    id: i64,
    name: String,
    email: String,
}

#[endpoint]
async fn get_user() -> Json<User> {
    Json(User {
        id: 1,
        name: "John".to_string(),
        email: "john@example.com".to_string(),
    })
}
```

## Path Parameters

```rust
use salvo::oapi::extract::PathParam;

#[endpoint]
async fn show_user(id: PathParam<i64>) -> String {
    format!("User ID: {}", id.into_inner())
}
```

## Query Parameters

```rust
use salvo::oapi::extract::QueryParam;

#[derive(Deserialize, ToParameters)]
struct Pagination {
    /// Page number
    #[salvo(parameter(default = 1, minimum = 1))]
    page: u32,
    /// Items per page
    #[salvo(parameter(default = 20, minimum = 1, maximum = 100))]
    per_page: u32,
}

#[endpoint]
async fn list_users(query: QueryParam<Pagination>) -> Json<Vec<User>> {
    Json(vec![])
}
```

## Status Codes and Errors

```rust
#[derive(Serialize, ToSchema)]
struct ErrorResponse {
    message: String,
}

#[endpoint(
    responses(
        (status_code = 200, description = "Success", body = User),
        (status_code = 404, description = "User not found", body = ErrorResponse),
    )
)]
async fn get_user(id: PathParam<i64>) -> Result<Json<User>, StatusError> {
    Ok(Json(User { /* ... */ }))
}
```

## Tags and Descriptions

```rust
#[endpoint(
    tags("users"),
    summary = "Create a new user",
    description = "Creates a new user account with the provided information"
)]
async fn create_user(body: JsonBody<CreateUser>) -> StatusCode {
    StatusCode::CREATED
}
```

## OpenAPI Document Generation

```rust
use salvo::oapi::{OpenApi, Info, License};

#[tokio::main]
async fn main() {
    let router = Router::new()
        .push(Router::with_path("users").get(list_users).post(create_user))
        .push(Router::with_path("users/<id>").get(show_user));

    let doc = OpenApi::new("My API", "1.0.0")
        .info(
            Info::new("My API", "1.0.0")
                .description("API description")
                .license(License::new("MIT"))
        )
        .merge_router(&router);

    let router = router
        .push(doc.into_router("/api-doc/openapi.json"))
        .push(SwaggerUi::new("/api-doc/openapi.json").into_router("/swagger-ui"));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Swagger UI

```rust
use salvo::oapi::swagger_ui::SwaggerUi;

let router = router.push(
    SwaggerUi::new("/api-doc/openapi.json")
        .into_router("/swagger-ui")
);
```

## Schema Customization

```rust
#[derive(Serialize, ToSchema)]
#[salvo(schema(example = json!({"id": 1, "name": "John"})))]
struct User {
    id: i64,

    #[salvo(schema(minimum = 1, maximum = 100))]
    age: u8,

    #[salvo(schema(pattern = "^[a-zA-Z]+$"))]
    name: String,

    #[salvo(schema(format = "email"))]
    email: String,
}
```

## Security Schemes

```rust
use salvo::oapi::security::{Http, HttpAuthScheme, SecurityScheme};

let doc = OpenApi::new("My API", "1.0.0")
    .add_security_scheme(
        "bearer_auth",
        SecurityScheme::Http(Http::new(HttpAuthScheme::Bearer))
    );

#[endpoint(
    security(("bearer_auth" = []))
)]
async fn protected_handler() -> &'static str {
    "Protected"
}
```

## Best Practices

1. Use `#[endpoint]` for all API handlers
2. Add `ToSchema` to all request/response types
3. Document fields with doc comments
4. Use `ToParameters` for query parameters
5. Specify response status codes explicitly
6. Group endpoints with tags
7. Provide examples for complex schemas
8. Use security schemes for protected endpoints
9. Serve Swagger UI for interactive documentation
