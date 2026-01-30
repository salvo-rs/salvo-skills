---
name: salvo-openapi
description: Generate OpenAPI documentation automatically from Salvo handlers. Use for API documentation, Swagger UI, and API client generation.
---

# Salvo OpenAPI Integration

This skill helps generate OpenAPI 3.0 documentation from Salvo applications.

## Key Difference: #[handler] vs #[endpoint]

- `#[handler]` - Basic Salvo handler, no OpenAPI documentation
- `#[endpoint]` - Generates OpenAPI documentation automatically

Use `#[endpoint]` for all handlers that should appear in API documentation.

## Setup

Add dependencies:

```toml
[dependencies]
salvo = { version = "0.89.0", features = ["oapi"] }
serde = { version = "1", features = ["derive"] }
```

## Basic Usage

Use `#[endpoint]` instead of `#[handler]`:

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

    // Create OpenAPI documentation
    let doc = OpenApi::new("My API", "1.0.0").merge_router(&router);

    // Add routes for API documentation
    let router = router
        .unshift(doc.into_router("/api-doc/openapi.json"))
        .unshift(SwaggerUi::new("/api-doc/openapi.json").into_router("/swagger-ui"));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## OpenAPI Extractors

These extractors work with both `#[handler]` and `#[endpoint]`, but only generate documentation with `#[endpoint]`:

### Path Parameters

```rust
use salvo::oapi::extract::PathParam;

#[endpoint]
async fn get_user(id: PathParam<i64>) -> String {
    format!("User ID: {}", id.into_inner())
}
```

### Query Parameters

```rust
use salvo::oapi::extract::QueryParam;

// Optional parameter (false = not required)
#[endpoint]
async fn search(q: QueryParam<String, false>) -> String {
    format!("Search: {}", q.as_deref().unwrap_or(""))
}

// Required parameter (true = required)
#[endpoint]
async fn search_required(q: QueryParam<String, true>) -> String {
    format!("Search: {}", q.into_inner())
}
```

### JSON Body

```rust
use salvo::oapi::extract::JsonBody;

#[endpoint]
async fn create_user(user: JsonBody<CreateUser>) -> StatusCode {
    // user.into_inner() to get the actual value
    StatusCode::CREATED
}
```

## Request Body Documentation

```rust
use salvo::oapi::ToSchema;
use serde::{Deserialize, Serialize};

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
use salvo::oapi::ToSchema;
use serde::Serialize;

#[derive(Serialize, ToSchema)]
struct User {
    id: i64,
    name: String,
    email: String,
}

#[endpoint]
async fn get_user(id: PathParam<i64>) -> Json<User> {
    Json(User {
        id: id.into_inner(),
        name: "John".to_string(),
        email: "john@example.com".to_string(),
    })
}
```

## Query Parameters with ToParameters

```rust
use salvo::oapi::{ToParameters, ToSchema};
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
    let page = pagination.page.unwrap_or(1);
    let per_page = pagination.per_page.unwrap_or(20);
    Json(vec![])
}
```

## Status Codes and Error Responses

```rust
use salvo::oapi::ToSchema;
use serde::Serialize;

#[derive(Serialize, ToSchema)]
struct ErrorResponse {
    message: String,
}

#[endpoint(
    status_codes(200, 404),
    responses(
        (status_code = 200, description = "User found", body = User),
        (status_code = 404, description = "User not found", body = ErrorResponse),
    )
)]
async fn get_user(id: PathParam<i64>) -> Result<Json<User>, StatusError> {
    Ok(Json(User {
        id: id.into_inner(),
        name: "John".to_string(),
        email: "john@example.com".to_string(),
    }))
}
```

## Tags, Summary and Description

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
        .push(Router::with_path("users/{id}").get(show_user));

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
use salvo::oapi::ToSchema;
use serde::Serialize;

#[derive(Serialize, ToSchema)]
#[salvo(schema(example = json!({"id": 1, "name": "John", "email": "john@example.com"})))]
struct User {
    id: i64,

    #[salvo(schema(minimum = 1, maximum = 100))]
    age: Option<u8>,

    #[salvo(schema(pattern = "^[a-zA-Z]+$"))]
    name: String,

    #[salvo(schema(format = "email"))]
    email: String,
}
```

## Security Schemes

```rust
use salvo::oapi::security::{Http, HttpAuthScheme, SecurityScheme};
use salvo::prelude::*;

#[tokio::main]
async fn main() {
    let router = Router::new()
        .push(Router::with_path("users").get(list_users))
        .push(Router::with_path("profile").get(get_profile));

    let doc = OpenApi::new("My API", "1.0.0")
        .description("API with authentication")
        .add_security_scheme(
            "bearer_auth",
            SecurityScheme::Http(Http::new(HttpAuthScheme::Bearer))
        )
        .merge_router(&router);

    let router = router
        .unshift(doc.into_router("/api-doc/openapi.json"))
        .unshift(SwaggerUi::new("/api-doc/openapi.json").into_router("/swagger-ui"));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}

#[endpoint(
    security(("bearer_auth" = []))
)]
async fn get_profile() -> &'static str {
    "Protected profile"
}
```

## File Upload Documentation

```rust
use salvo::oapi::extract::*;

#[endpoint(
    tags("files"),
    request_body(content = "multipart/form-data")
)]
async fn upload_file(req: &mut Request) -> Result<Json<UploadResponse>, StatusError> {
    let file = req.file("file").await
        .ok_or_else(|| StatusError::bad_request())?;

    let filename = file.name().unwrap_or("unnamed").to_string();
    let size = file.size();

    Ok(Json(UploadResponse { filename, size }))
}
```

## Complete OpenAPI Setup Example

```rust
use salvo::oapi::extract::*;
use salvo::oapi::{OpenApi, ToSchema, ToParameters};
use salvo::prelude::*;
use serde::{Deserialize, Serialize};

// Schema definitions
#[derive(Serialize, Deserialize, ToSchema)]
struct User {
    id: i64,
    name: String,
    email: String,
}

#[derive(Deserialize, ToSchema)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Deserialize, ToParameters)]
struct Pagination {
    #[salvo(parameter(default = 1))]
    page: Option<u32>,
    #[salvo(parameter(default = 20))]
    per_page: Option<u32>,
}

// Endpoints
#[endpoint(tags("users"), summary = "List all users")]
async fn list_users(pagination: Pagination) -> Json<Vec<User>> {
    Json(vec![])
}

#[endpoint(tags("users"), summary = "Get user by ID")]
async fn get_user(id: PathParam<i64>) -> Result<Json<User>, StatusError> {
    Ok(Json(User {
        id: id.into_inner(),
        name: "John".to_string(),
        email: "john@example.com".to_string(),
    }))
}

#[endpoint(tags("users"), summary = "Create a new user", status_codes(201))]
async fn create_user(body: JsonBody<CreateUser>) -> StatusCode {
    StatusCode::CREATED
}

#[tokio::main]
async fn main() {
    let router = Router::new()
        .push(
            Router::with_path("users")
                .get(list_users)
                .post(create_user)
                .push(Router::with_path("{id}").get(get_user))
        );

    // Create OpenAPI documentation with metadata
    let doc = OpenApi::new("User API", "1.0.0")
        .description("A comprehensive user management API")
        .contact_name("API Support")
        .contact_email("support@example.com")
        .license_name("MIT")
        .merge_router(&router);

    // Add documentation routes
    let router = router
        .unshift(doc.into_router("/api-doc/openapi.json"))
        .unshift(SwaggerUi::new("/api-doc/openapi.json").into_router("/swagger-ui"));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Generated Client Libraries

With OpenAPI documentation, you can generate client libraries:

- **OpenAPI Generator**: `openapi-generator-cli generate -i openapi.json -g rust`
- **Swagger Codegen**: For various languages
- **Postman**: Import directly for testing

## Best Practices

1. Use `#[endpoint]` for all API handlers that need documentation
2. Add `ToSchema` to all request/response types
3. Document fields with Rust doc comments (`///`)
4. Use `ToParameters` for query parameter structs
5. Specify response status codes explicitly
6. Group endpoints with meaningful tags
7. Provide examples for complex schemas
8. Use security schemes for protected endpoints
9. Serve Swagger UI for interactive documentation
10. Keep documentation in sync by generating from code
