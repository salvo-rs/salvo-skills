---
name: salvo-database
description: Integrate databases with Salvo using SQLx, Diesel, SeaORM, or other ORMs. Use for persistent data storage and database operations.
---

# Salvo Database Integration

This skill helps integrate databases with Salvo applications.

## SQLx (Async, Compile-time Checked)

### Setup

```toml
[dependencies]
sqlx = { version = "0.7", features = ["runtime-tokio", "postgres", "macros"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

### Connection Pool

```rust
use sqlx::PgPool;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let pool = PgPool::connect("postgres://user:pass@localhost/db")
        .await
        .expect("Failed to connect to database");

    let router = Router::new()
        .hoop(add_pool(pool))
        .get(list_users);

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}

#[handler]
fn add_pool(pool: PgPool) -> impl Handler {
    move |depot: &mut Depot| {
        depot.inject(pool.clone());
    }
}
```

### Query Handlers

```rust
use sqlx::FromRow;
use serde::Serialize;

#[derive(FromRow, Serialize)]
struct User {
    id: i64,
    name: String,
    email: String,
}

#[handler]
async fn list_users(depot: &mut Depot) -> Json<Vec<User>> {
    let pool = depot.obtain::<PgPool>().unwrap();

    let users = sqlx::query_as::<_, User>("SELECT id, name, email FROM users")
        .fetch_all(pool)
        .await
        .unwrap();

    Json(users)
}

#[handler]
async fn create_user(body: JsonBody<CreateUser>, depot: &mut Depot) -> StatusCode {
    let pool = depot.obtain::<PgPool>().unwrap();
    let user = body.into_inner();

    sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind(&user.name)
        .bind(&user.email)
        .execute(pool)
        .await
        .unwrap();

    StatusCode::CREATED
}
```

## SeaORM (Async ORM)

### Setup

```toml
[dependencies]
sea-orm = { version = "0.12", features = ["sqlx-postgres", "runtime-tokio-native-tls", "macros"] }
```

### Connection

```rust
use sea_orm::{Database, DatabaseConnection};

#[tokio::main]
async fn main() {
    let db = Database::connect("postgres://user:pass@localhost/db")
        .await
        .expect("Failed to connect");

    let router = Router::new()
        .hoop(add_db(db))
        .get(list_users);

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}

#[handler]
fn add_db(db: DatabaseConnection) -> impl Handler {
    move |depot: &mut Depot| {
        depot.inject(db.clone());
    }
}
```

### Entity Operations

```rust
use sea_orm::*;

// Assuming you have generated entities with sea-orm-cli

#[handler]
async fn list_users(depot: &mut Depot) -> Json<Vec<user::Model>> {
    let db = depot.obtain::<DatabaseConnection>().unwrap();

    let users = user::Entity::find()
        .all(db)
        .await
        .unwrap();

    Json(users)
}

#[handler]
async fn show_user(id: PathParam<i64>, depot: &mut Depot) -> Result<Json<user::Model>, StatusError> {
    let db = depot.obtain::<DatabaseConnection>().unwrap();

    let user = user::Entity::find_by_id(id.into_inner())
        .one(db)
        .await
        .unwrap()
        .ok_or_else(|| StatusError::not_found())?;

    Ok(Json(user))
}

#[handler]
async fn create_user(body: JsonBody<CreateUser>, depot: &mut Depot) -> StatusCode {
    let db = depot.obtain::<DatabaseConnection>().unwrap();
    let data = body.into_inner();

    let user = user::ActiveModel {
        name: Set(data.name),
        email: Set(data.email),
        ..Default::default()
    };

    user.insert(db).await.unwrap();

    StatusCode::CREATED
}
```

## Diesel (Sync ORM)

### Setup

```toml
[dependencies]
diesel = { version = "2.1", features = ["postgres", "r2d2"] }
```

### Connection Pool

```rust
use diesel::r2d2::{self, ConnectionManager};
use diesel::PgConnection;

type DbPool = r2d2::Pool<ConnectionManager<PgConnection>>;

#[tokio::main]
async fn main() {
    let manager = ConnectionManager::<PgConnection>::new("postgres://user:pass@localhost/db");
    let pool = r2d2::Pool::builder()
        .build(manager)
        .expect("Failed to create pool");

    let router = Router::new()
        .hoop(add_pool(pool))
        .get(list_users);

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}

#[handler]
fn add_pool(pool: DbPool) -> impl Handler {
    move |depot: &mut Depot| {
        depot.inject(pool.clone());
    }
}
```

### Query Handlers

```rust
use diesel::prelude::*;

#[handler]
async fn list_users(depot: &mut Depot) -> Json<Vec<User>> {
    let pool = depot.obtain::<DbPool>().unwrap();
    let mut conn = pool.get().unwrap();

    let users = tokio::task::spawn_blocking(move || {
        use crate::schema::users::dsl::*;
        users.load::<User>(&mut conn)
    })
    .await
    .unwrap()
    .unwrap();

    Json(users)
}
```

## Error Handling

```rust
use salvo::http::StatusError;

#[handler]
async fn get_user(id: PathParam<i64>, depot: &mut Depot) -> Result<Json<User>, StatusError> {
    let pool = depot.obtain::<PgPool>().unwrap();

    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
        .bind(id.into_inner())
        .fetch_optional(pool)
        .await
        .map_err(|_| StatusError::internal_server_error())?
        .ok_or_else(|| StatusError::not_found())?;

    Ok(Json(user))
}
```

## Transactions

```rust
#[handler]
async fn transfer_funds(body: JsonBody<Transfer>, depot: &mut Depot) -> Result<StatusCode, StatusError> {
    let pool = depot.obtain::<PgPool>().unwrap();
    let transfer = body.into_inner();

    let mut tx = pool.begin().await
        .map_err(|_| StatusError::internal_server_error())?;

    sqlx::query("UPDATE accounts SET balance = balance - $1 WHERE id = $2")
        .bind(transfer.amount)
        .bind(transfer.from_account)
        .execute(&mut *tx)
        .await
        .map_err(|_| StatusError::internal_server_error())?;

    sqlx::query("UPDATE accounts SET balance = balance + $1 WHERE id = $2")
        .bind(transfer.amount)
        .bind(transfer.to_account)
        .execute(&mut *tx)
        .await
        .map_err(|_| StatusError::internal_server_error())?;

    tx.commit().await
        .map_err(|_| StatusError::internal_server_error())?;

    Ok(StatusCode::OK)
}
```

## Best Practices

1. Use connection pooling for performance
2. Store pool/connection in Depot via middleware
3. Use `spawn_blocking` for sync database operations
4. Handle database errors gracefully
5. Use transactions for multi-step operations
6. Validate input before database operations
7. Use prepared statements to prevent SQL injection
8. Consider using migrations for schema management
