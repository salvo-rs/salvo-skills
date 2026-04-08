---
name: salvo-database
description: Integrate databases with Salvo using SQLx, Diesel, SeaORM, or other ORMs. Use for persistent data storage and database operations.
version: 0.89.3
tags: [data, database, sqlx, seaorm, diesel]
---

# Salvo Database Integration

Share a pool via `affix_state::inject(pool)` and pull it out with `depot.obtain::<T>()`.

`depot.obtain::<T>()` returns `Result<&T, _>` (NOT `Option`). Convert to a Salvo error with `.map_err(|_| StatusError::internal_server_error())` or propagate with `?` after conversion.

Requires the `affix-state` feature on `salvo`:

```toml
salvo = { version = "0.89.3", features = ["affix-state"] }
```

## SQLx

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["affix-state"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "macros"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
serde = { version = "1", features = ["derive"] }
```

```rust
use salvo::prelude::*;
use salvo::affix_state;
use sqlx::{FromRow, PgPool};
use serde::{Deserialize, Serialize};

#[derive(FromRow, Serialize)]
struct User { id: i64, name: String, email: String }

#[derive(Deserialize)]
struct CreateUser { name: String, email: String }

#[handler]
async fn list_users(depot: &mut Depot) -> Result<Json<Vec<User>>, StatusError> {
    let pool = depot.obtain::<PgPool>()
        .map_err(|_| StatusError::internal_server_error())?;
    let users = sqlx::query_as::<_, User>("SELECT id, name, email FROM users")
        .fetch_all(pool)
        .await
        .map_err(|_| StatusError::internal_server_error())?;
    Ok(Json(users))
}

#[handler]
async fn create_user(
    body: JsonBody<CreateUser>,
    depot: &mut Depot,
) -> Result<StatusCode, StatusError> {
    let pool = depot.obtain::<PgPool>()
        .map_err(|_| StatusError::internal_server_error())?;
    let user = body.into_inner();
    sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind(&user.name)
        .bind(&user.email)
        .execute(pool)
        .await
        .map_err(|_| StatusError::internal_server_error())?;
    Ok(StatusCode::CREATED)
}

#[handler]
async fn get_user(req: &mut Request, depot: &mut Depot) -> Result<Json<User>, StatusError> {
    let pool = depot.obtain::<PgPool>()
        .map_err(|_| StatusError::internal_server_error())?;
    let id = req.param::<i64>("id")
        .ok_or_else(StatusError::bad_request)?;

    let user = sqlx::query_as::<_, User>("SELECT id, name, email FROM users WHERE id = $1")
        .bind(id)
        .fetch_optional(pool)
        .await
        .map_err(|_| StatusError::internal_server_error())?
        .ok_or_else(StatusError::not_found)?;
    Ok(Json(user))
}

#[tokio::main]
async fn main() {
    let pool = PgPool::connect("postgres://user:pass@localhost/db")
        .await
        .expect("db connect");

    let router = Router::new()
        .hoop(affix_state::inject(pool))
        .push(
            Router::with_path("users")
                .get(list_users)
                .post(create_user)
                .push(Router::with_path("{id}").get(get_user)),
        );

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

### Transactions

```rust
#[handler]
async fn transfer(
    body: JsonBody<Transfer>,
    depot: &mut Depot,
) -> Result<StatusCode, StatusError> {
    let pool = depot.obtain::<PgPool>()
        .map_err(|_| StatusError::internal_server_error())?;
    let t = body.into_inner();

    let mut tx = pool.begin().await
        .map_err(|_| StatusError::internal_server_error())?;

    sqlx::query("UPDATE accounts SET balance = balance - $1 WHERE id = $2")
        .bind(t.amount).bind(t.from_account)
        .execute(&mut *tx).await
        .map_err(|_| StatusError::internal_server_error())?;

    sqlx::query("UPDATE accounts SET balance = balance + $1 WHERE id = $2")
        .bind(t.amount).bind(t.to_account)
        .execute(&mut *tx).await
        .map_err(|_| StatusError::internal_server_error())?;

    tx.commit().await
        .map_err(|_| StatusError::internal_server_error())?;
    Ok(StatusCode::OK)
}
```

## SeaORM

```toml
sea-orm = { version = "1.0", features = ["sqlx-postgres", "runtime-tokio-native-tls", "macros"] }
```

```rust
use salvo::prelude::*;
use salvo::affix_state;
use sea_orm::*;

#[tokio::main]
async fn main() {
    let db = Database::connect("postgres://user:pass@localhost/db")
        .await.expect("db connect");

    let router = Router::new()
        .hoop(affix_state::inject(db))
        .push(Router::with_path("users").get(list_users)
              .push(Router::with_path("{id}").get(show_user)));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}

#[handler]
async fn list_users(depot: &mut Depot) -> Result<Json<Vec<user::Model>>, StatusError> {
    let db = depot.obtain::<DatabaseConnection>()
        .map_err(|_| StatusError::internal_server_error())?;
    let users = user::Entity::find().all(db).await
        .map_err(|_| StatusError::internal_server_error())?;
    Ok(Json(users))
}

#[handler]
async fn show_user(
    req: &mut Request,
    depot: &mut Depot,
) -> Result<Json<user::Model>, StatusError> {
    let db = depot.obtain::<DatabaseConnection>()
        .map_err(|_| StatusError::internal_server_error())?;
    let id = req.param::<i64>("id").ok_or_else(StatusError::bad_request)?;
    let user = user::Entity::find_by_id(id).one(db).await
        .map_err(|_| StatusError::internal_server_error())?
        .ok_or_else(StatusError::not_found)?;
    Ok(Json(user))
}

#[handler]
async fn create_user(
    body: JsonBody<CreateUser>,
    depot: &mut Depot,
) -> Result<StatusCode, StatusError> {
    let db = depot.obtain::<DatabaseConnection>()
        .map_err(|_| StatusError::internal_server_error())?;
    let data = body.into_inner();
    let m = user::ActiveModel {
        name: Set(data.name),
        email: Set(data.email),
        ..Default::default()
    };
    m.insert(db).await.map_err(|_| StatusError::internal_server_error())?;
    Ok(StatusCode::CREATED)
}
```

## Diesel (sync ORM)

Diesel is blocking, so wrap calls in `tokio::task::spawn_blocking` to avoid starving the async runtime.

```toml
diesel = { version = "2.2", features = ["postgres", "r2d2"] }
```

```rust
use salvo::prelude::*;
use salvo::affix_state;
use diesel::prelude::*;
use diesel::r2d2::{self, ConnectionManager};
use diesel::PgConnection;

type DbPool = r2d2::Pool<ConnectionManager<PgConnection>>;

#[tokio::main]
async fn main() {
    let manager = ConnectionManager::<PgConnection>::new("postgres://user:pass@localhost/db");
    let pool = r2d2::Pool::builder().build(manager).expect("pool");

    let router = Router::new()
        .hoop(affix_state::inject(pool))
        .push(Router::with_path("users").get(list_users));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}

#[handler]
async fn list_users(depot: &mut Depot) -> Result<Json<Vec<User>>, StatusError> {
    let pool = depot.obtain::<DbPool>()
        .map_err(|_| StatusError::internal_server_error())?
        .clone();

    let users = tokio::task::spawn_blocking(move || {
        use crate::schema::users::dsl::*;
        let mut conn = pool.get().map_err(|_| ())?;
        users.load::<User>(&mut conn).map_err(|_| ())
    })
    .await
    .map_err(|_| StatusError::internal_server_error())?
    .map_err(|_| StatusError::internal_server_error())?;

    Ok(Json(users))
}
```

## Salvo-Specific Notes

- Inject the pool, not a connection — pools are `Clone` and share cheaply across requests.
- `depot.obtain::<T>()` returns `Result<&T, _>`; use `.map_err(...)`, not `.ok_or_else(...)`.
- For Diesel (sync), always wrap queries in `spawn_blocking` to avoid blocking the tokio runtime.
- `affix_state::inject()` requires the `affix-state` feature on `salvo`.

## Related Skills

- **salvo-error-handling**: Convert database errors to responses
- **salvo-testing**: Integration testing with a real database
- **salvo-caching**: Cache database query results
