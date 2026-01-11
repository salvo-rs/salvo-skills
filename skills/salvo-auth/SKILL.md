---
name: salvo-auth
description: Implement authentication and authorization using JWT, Basic Auth, or custom schemes. Use for securing API endpoints and user management.
---

# Salvo Authentication

This skill helps implement authentication and authorization in Salvo applications.

## JWT Authentication

### Setup

```toml
[dependencies]
salvo = { version = "0.76", features = ["jwt-auth"] }
jsonwebtoken = "9"
```

### JWT Middleware

```rust
use salvo::jwt_auth::{JwtAuth, JwtClaims};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    exp: i64,
}

#[tokio::main]
async fn main() {
    let auth_handler = JwtAuth::new("secret_key")
        .finders(vec![
            Box::new(HeaderFinder::new()),
            Box::new(QueryFinder::new("token")),
        ]);

    let router = Router::new()
        .push(Router::with_path("login").post(login))
        .push(
            Router::with_path("protected")
                .hoop(auth_handler)
                .get(protected_handler)
        );

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

### Login Handler

```rust
use jsonwebtoken::{encode, EncodingKey, Header};

#[derive(Deserialize)]
struct LoginRequest {
    username: String,
    password: String,
}

#[derive(Serialize)]
struct LoginResponse {
    token: String,
}

#[handler]
async fn login(body: JsonBody<LoginRequest>) -> Result<Json<LoginResponse>, StatusError> {
    let req = body.into_inner();

    // Validate credentials
    if !validate_credentials(&req.username, &req.password) {
        return Err(StatusError::unauthorized());
    }

    // Create JWT
    let claims = Claims {
        sub: req.username,
        exp: (chrono::Utc::now() + chrono::Duration::hours(24)).timestamp(),
    };

    let token = encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret("secret_key".as_ref()),
    )
    .map_err(|_| StatusError::internal_server_error())?;

    Ok(Json(LoginResponse { token }))
}
```

### Protected Handler

```rust
#[handler]
async fn protected_handler(depot: &mut Depot) -> String {
    let claims = depot.get::<JwtClaims<Claims>>("jwt_claims").unwrap();
    format!("Hello, {}!", claims.claims.sub)
}
```

## Basic Authentication

### Setup

```toml
[dependencies]
salvo = { version = "0.76", features = ["basic-auth"] }
```

### Basic Auth Middleware

```rust
use salvo::basic_auth::{BasicAuth, BasicAuthValidator};

struct MyValidator;

#[async_trait]
impl BasicAuthValidator for MyValidator {
    async fn validate(&self, username: &str, password: &str, _depot: &mut Depot) -> bool {
        username == "admin" && password == "password"
    }
}

#[tokio::main]
async fn main() {
    let auth_handler = BasicAuth::new(MyValidator);

    let router = Router::new()
        .push(
            Router::with_path("admin")
                .hoop(auth_handler)
                .get(admin_handler)
        );

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Custom Authentication Middleware

```rust
#[handler]
async fn auth_middleware(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let token = req.header::<String>("Authorization")
        .and_then(|h| h.strip_prefix("Bearer ").map(String::from));

    match token {
        Some(token) => {
            match validate_token(&token) {
                Ok(user_id) => {
                    depot.insert("user_id", user_id);
                    ctrl.call_next(req, depot, res).await;
                }
                Err(_) => {
                    res.status_code(StatusCode::UNAUTHORIZED);
                    res.render(Json(json!({"error": "Invalid token"})));
                    ctrl.skip_rest();
                }
            }
        }
        None => {
            res.status_code(StatusCode::UNAUTHORIZED);
            res.render(Json(json!({"error": "Missing token"})));
            ctrl.skip_rest();
        }
    }
}
```

## Session-Based Authentication

```rust
use salvo::session::{SessionHandler, SessionStore};

#[tokio::main]
async fn main() {
    let session_handler = SessionHandler::builder(
        CookieStore::new(),
        b"secret_key_must_be_at_least_64_bytes_long_for_security_reasons",
    )
    .build()
    .unwrap();

    let router = Router::new()
        .hoop(session_handler)
        .post("login", login)
        .get("profile", profile);

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}

#[handler]
async fn login(req: &mut Request, depot: &mut Depot) -> StatusCode {
    let session = depot.session_mut().unwrap();
    session.insert("user_id", 123).unwrap();
    StatusCode::OK
}

#[handler]
async fn profile(depot: &mut Depot) -> Result<String, StatusError> {
    let session = depot.session().unwrap();
    let user_id: i64 = session.get("user_id")
        .ok_or_else(|| StatusError::unauthorized())?;

    Ok(format!("User ID: {}", user_id))
}
```

## Role-Based Authorization

```rust
#[derive(Clone)]
enum Role {
    Admin,
    User,
}

#[handler]
fn require_role(required: Role) -> impl Handler {
    move |depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl| async move {
        let user_role = depot.get::<Role>("user_role");

        match user_role {
            Some(role) if matches!((role, &required), (Role::Admin, _) | (Role::User, Role::User)) => {
                ctrl.call_next(req, depot, res).await;
            }
            _ => {
                res.status_code(StatusCode::FORBIDDEN);
                res.render("Forbidden");
                ctrl.skip_rest();
            }
        }
    }
}

let router = Router::new()
    .push(
        Router::with_path("admin")
            .hoop(auth_middleware)
            .hoop(require_role(Role::Admin))
            .get(admin_handler)
    );
```

## API Key Authentication

```rust
#[handler]
async fn api_key_auth(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let api_key = req.header::<String>("X-API-Key");

    match api_key {
        Some(key) if is_valid_api_key(&key) => {
            depot.insert("api_key", key);
            ctrl.call_next(req, depot, res).await;
        }
        _ => {
            res.status_code(StatusCode::UNAUTHORIZED);
            res.render("Invalid API key");
            ctrl.skip_rest();
        }
    }
}
```

## Best Practices

1. Never store passwords in plain text
2. Use strong secret keys (at least 64 bytes)
3. Set appropriate token expiration times
4. Use HTTPS in production
5. Implement rate limiting for login endpoints
6. Store sensitive data in environment variables
7. Validate tokens on every protected request
8. Use refresh tokens for long-lived sessions
9. Implement proper logout functionality
10. Log authentication failures for security monitoring
