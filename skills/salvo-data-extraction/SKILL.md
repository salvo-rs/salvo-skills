---
name: salvo-data-extraction
description: Extract and validate data from requests including JSON, forms, query parameters, and path parameters. Use for handling user input and API payloads.
---

# Salvo Data Extraction

This skill helps extract and validate data from HTTP requests in Salvo applications.

## Extractible Trait

The `Extractible` trait enables automatic data extraction from requests.

### Basic Usage

```rust
use salvo::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Extractible, Deserialize, Debug)]
#[salvo(extract(default_source(from = "body")))]
struct CreateUser {
    name: String,
    email: String,
}

#[handler]
async fn create_user(user: CreateUser) -> String {
    format!("Created user: {:?}", user)
}
```

## Data Sources

### JSON Body
```rust
#[derive(Extractible, Deserialize)]
#[salvo(extract(default_source(from = "body")))]
struct UserData {
    name: String,
    email: String,
}

// Or use JsonBody directly
#[handler]
async fn handler(body: JsonBody<UserData>) -> StatusCode {
    let user = body.into_inner();
    StatusCode::CREATED
}
```

### Query Parameters
```rust
#[derive(Extractible, Deserialize)]
#[salvo(extract(default_source(from = "query")))]
struct Pagination {
    page: u32,
    per_page: u32,
}

#[handler]
async fn list_items(query: Pagination) -> String {
    format!("Page {} with {} items", query.page, query.per_page)
}
```

### Path Parameters
```rust
#[derive(Extractible, Deserialize)]
#[salvo(extract(default_source(from = "param")))]
struct UserId {
    id: i64,
}

#[handler]
async fn show_user(params: UserId) -> String {
    format!("User ID: {}", params.id)
}
```

### Form Data
```rust
#[derive(Extractible, Deserialize)]
#[salvo(extract(default_source(from = "body"), format = "form"))]
struct LoginForm {
    username: String,
    password: String,
}
```

## Mixed Sources

Extract from multiple sources:

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
    // data.id from path, name and email from body
    StatusCode::OK
}
```

## Manual Extraction

For simple cases, extract directly from Request:

```rust
#[handler]
async fn handler(req: &mut Request) -> String {
    // Query parameter
    let name = req.query::<String>("name").unwrap_or_default();

    // Path parameter
    let id = req.param::<i64>("id").unwrap();

    // Header
    let token = req.header::<String>("Authorization");

    // Parse JSON body
    let body: UserData = req.parse_json().await.unwrap();

    format!("Processed request")
}
```

## Validation

Use validator crate for validation:

```rust
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
    user.validate()?;
    Ok(StatusCode::CREATED)
}
```

## Nested Structures

```rust
#[derive(Extractible, Deserialize)]
#[salvo(extract(default_source(from = "body")))]
struct CreatePost {
    title: String,
    content: String,

    #[salvo(extract(flatten))]
    author: Author,
}

#[derive(Deserialize)]
struct Author {
    name: String,
    email: String,
}
```

## Best Practices

1. Use `Extractible` for complex data structures
2. Use `JsonBody`, `FormBody` for simple cases
3. Specify data sources explicitly for clarity
4. Validate input data at boundaries
5. Use typed path parameters
6. Handle extraction errors gracefully
7. Use `into_inner()` to unwrap extracted data
