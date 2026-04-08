---
name: salvo-auth
description: Implement authentication and authorization using JWT, Basic Auth, or custom schemes. Use for securing API endpoints and user management.
version: 0.89.3
tags: [security, authentication, jwt, basic-auth]
---

# Salvo Authentication

## JWT Authentication

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["jwt-auth"] }
jsonwebtoken = "9"
serde = { version = "1", features = ["derive"] }
chrono = "0.4"
```

`JwtAuth::new` takes a **decoder** (not a raw secret string). Use `ConstDecoder::from_secret` for HMAC or `ConstDecoder::from_rsa_pem` / `from_ec_pem` for asymmetric keys.

```rust
use salvo::prelude::*;
use salvo::jwt_auth::{ConstDecoder, HeaderFinder, JwtAuth, JwtAuthDepotExt, QueryFinder};
use jsonwebtoken::{encode, EncodingKey, Header};
use serde::{Deserialize, Serialize};

const SECRET_KEY: &str = "your-secret-key-at-least-32-bytes";

#[derive(Debug, Serialize, Deserialize)]
struct JwtClaims {
    sub: String,
    exp: i64,
    role: String,
}

#[derive(Deserialize)]
struct LoginRequest { username: String, password: String }

#[derive(Serialize)]
struct LoginResponse { token: String }

#[handler]
async fn login(body: JsonBody<LoginRequest>) -> Result<Json<LoginResponse>, StatusError> {
    let req = body.into_inner();
    if req.username != "admin" || req.password != "password" {
        return Err(StatusError::unauthorized());
    }
    let claims = JwtClaims {
        sub: req.username,
        exp: (chrono::Utc::now() + chrono::Duration::hours(24)).timestamp(),
        role: "user".to_string(),
    };
    let token = encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(SECRET_KEY.as_bytes()),
    )
    .map_err(|_| StatusError::internal_server_error())?;
    Ok(Json(LoginResponse { token }))
}

#[handler]
async fn protected(depot: &mut Depot) -> Result<String, StatusError> {
    let data = depot.jwt_auth_data::<JwtClaims>()
        .ok_or_else(StatusError::unauthorized)?;
    Ok(format!("Hello, {}! Role: {}", data.claims.sub, data.claims.role))
}

#[tokio::main]
async fn main() {
    let auth: JwtAuth<JwtClaims, _> = JwtAuth::new(ConstDecoder::from_secret(SECRET_KEY.as_bytes()))
        .finders(vec![
            Box::new(HeaderFinder::new()),
            Box::new(QueryFinder::new("token")),
        ]);

    let router = Router::new()
        .push(Router::with_path("login").post(login))
        .push(Router::with_path("protected").hoop(auth).get(protected));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

Available finders: `HeaderFinder`, `QueryFinder`, `FormFinder`, `CookieFinder`. Available decoders include `ConstDecoder` and (with `oidc` feature) `OidcDecoder`.

### Depot extension

`JwtAuthDepotExt` provides:
- `depot.jwt_auth_token() -> Option<&str>`
- `depot.jwt_auth_data::<C>() -> Option<&TokenData<C>>`
- `depot.jwt_auth_state() -> JwtAuthState` (`Authorized` / `Unauthorized` / `Forbidden`)

## Basic Authentication

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["basic-auth"] }
```

```rust
use salvo::prelude::*;
use salvo::basic_auth::{BasicAuth, BasicAuthValidator};

struct MyValidator;

impl BasicAuthValidator for MyValidator {
    async fn validate(&self, username: &str, password: &str, depot: &mut Depot) -> bool {
        if username == "admin" && password == "password" {
            depot.insert("user_role", "admin");
            true
        } else {
            false
        }
    }
}

#[tokio::main]
async fn main() {
    let auth = BasicAuth::new(MyValidator);
    let router = Router::with_path("admin").hoop(auth).get(admin_handler);
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

`BasicAuth::new(v).set_realm("myapp")` customizes the realm. Retrieve the authenticated user via `depot.basic_auth_username()` (`BasicAuthDepotExt`).

## Custom Middleware (Bearer / API Key)

```rust
use salvo::prelude::*;

#[handler]
async fn auth_middleware(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let token = req.header::<String>("Authorization")
        .and_then(|h| h.strip_prefix("Bearer ").map(String::from));

    match token.and_then(|t| validate_token(&t).ok()) {
        Some(user_id) => {
            depot.insert("user_id", user_id);
        }
        None => {
            res.status_code(StatusCode::UNAUTHORIZED);
            res.render(Json(serde_json::json!({"error": "Unauthorized"})));
            ctrl.skip_rest();
        }
    }
}

fn validate_token(token: &str) -> Result<i64, ()> {
    if token == "valid_token" { Ok(123) } else { Err(()) }
}
```

For an API key scheme, read `req.header::<String>("X-API-Key")` and follow the same pattern.

## Role-Based Access Control

```rust
#[derive(Clone, PartialEq)]
enum Role { Admin, User, Guest }

fn require_role(required: Role) -> impl Handler {
    #[handler]
    async fn check(depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
        // Pull required role from a closure-captured value in real code.
    }
    check
}
```

In practice, store the role in `Depot` during authentication, then build a small `#[handler]` that reads it and calls `ctrl.skip_rest()` with 403 on mismatch. Chain it after the auth middleware:

```rust
Router::with_path("admin")
    .hoop(auth)
    .hoop(require_admin)
    .get(admin_handler);
```

## Refresh Tokens

Issue a short-lived access token (e.g. 15 min) and a long-lived refresh token stored server-side (DB/Redis). On `/refresh`, validate the refresh token, look up the user, mint a new access token, and optionally rotate the refresh token.

```rust
#[handler]
async fn refresh(body: JsonBody<RefreshRequest>) -> Result<Json<TokenResponse>, StatusError> {
    let user_id = validate_refresh_token(&body.into_inner().refresh_token)
        .map_err(|_| StatusError::unauthorized())?;
    let claims = JwtClaims {
        sub: user_id.to_string(),
        exp: (chrono::Utc::now() + chrono::Duration::minutes(15)).timestamp(),
        role: "user".into(),
    };
    let access_token = encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(SECRET_KEY.as_bytes()),
    ).map_err(|_| StatusError::internal_server_error())?;
    Ok(Json(TokenResponse {
        access_token,
        refresh_token: generate_refresh_token(),
        expires_in: 900,
    }))
}
```

## Salvo-specific Notes

- `JwtAuth::new` requires a `JwtAuthDecoder` impl, not a `&str`. Passing a string will not compile.
- `JwtAuth` has a type parameter for the claims type; annotate it (`JwtAuth<JwtClaims, _>`) when the compiler cannot infer it.
- `.force_passed(true)` lets the request continue even when unauthorized so downstream handlers can decide; default is `false`.
- `BasicAuthValidator::validate` uses native `async fn` in traits (no `async_trait` crate required).

## Related Skills

- **salvo-session**: Session-based authentication
- **salvo-cors**: Configure CORS for authenticated APIs
- **salvo-rate-limiter**: Rate limit authentication endpoints
