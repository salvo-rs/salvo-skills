---
name: salvo-middleware
description: Implement middleware for authentication, logging, CORS, and request processing. Use for cross-cutting concerns and request/response modification.
---

# Salvo Middleware

This skill helps implement middleware in Salvo applications. In Salvo, middleware is just a handler with flow control.

## Basic Middleware Pattern

Middleware uses `FlowCtrl` to control execution flow:

```rust
use salvo::prelude::*;

#[handler]
async fn logger(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    println!("Request: {} {}", req.method(), req.uri().path());

    // Continue to next handler
    ctrl.call_next(req, depot, res).await;

    println!("Response status: {}", res.status_code().unwrap_or(StatusCode::OK));
}
```

## Middleware Attachment

Use `hoop()` to attach middleware:

```rust
let router = Router::new()
    .hoop(logger)
    .hoop(auth_check)
    .get(handler);
```

Middleware applies to all child routes:

```rust
let router = Router::new()
    .push(
        Router::with_path("api")
            .hoop(auth_check)  // Only applies to /api routes
            .get(protected_handler)
    )
    .get(public_handler);  // No auth check
```

## Common Middleware Patterns

### Authentication
```rust
#[handler]
async fn auth_check(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let token = req.header::<String>("Authorization");

    match token {
        Some(token) if validate_token(&token) => {
            depot.insert("user_id", extract_user_id(&token));
            ctrl.call_next(req, depot, res).await;
        }
        _ => {
            res.status_code(StatusCode::UNAUTHORIZED);
            res.render("Unauthorized");
            ctrl.skip_rest();
        }
    }
}
```

### Request Logging
```rust
#[handler]
async fn request_logger(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    let start = std::time::Instant::now();
    let method = req.method().clone();
    let path = req.uri().path().to_string();

    ctrl.call_next(req, depot, res).await;

    let duration = start.elapsed();
    let status = res.status_code().unwrap_or(StatusCode::OK);
    println!("{} {} - {} ({:?})", method, path, status, duration);
}
```

### CORS
```rust
use salvo::cors::Cors;

let cors = Cors::new()
    .allow_origin("https://example.com")
    .allow_methods(vec!["GET", "POST", "PUT", "DELETE"])
    .allow_headers(vec!["Content-Type", "Authorization"])
    .into_handler();

let router = Router::new().hoop(cors);
```

### Rate Limiting
```rust
use salvo::rate_limiter::{RateLimiter, FixedGuard, RemoteIpIssuer};

let limiter = RateLimiter::new(
    FixedGuard::new(),
    RemoteIpIssuer,
);

let router = Router::new().hoop(limiter);
```

## Using Depot for Data Sharing

Store data in middleware for use in handlers:

```rust
#[handler]
async fn auth_middleware(depot: &mut Depot, ctrl: &mut FlowCtrl, req: &mut Request, res: &mut Response) {
    let user = authenticate(req).await;
    depot.insert("user", user);
    ctrl.call_next(req, depot, res).await;
}

#[handler]
async fn protected_handler(depot: &mut Depot) -> String {
    let user = depot.get::<User>("user").unwrap();
    format!("Hello, {}", user.name)
}
```

## Early Response

Stop execution and return response:

```rust
#[handler]
async fn validate_input(req: &mut Request, depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    if !is_valid_request(req) {
        res.status_code(StatusCode::BAD_REQUEST);
        res.render("Invalid request");
        ctrl.skip_rest();  // Stop here
        return;
    }
    ctrl.call_next(req, depot, res).await;
}
```

## Built-in Middleware

Salvo provides many built-in middleware:

```rust
use salvo::compression::Compression;
use salvo::cors::Cors;
use salvo::csrf::CsrfHandler;
use salvo::logging::Logger;
use salvo::timeout::Timeout;

let router = Router::new()
    .hoop(Logger::new())
    .hoop(Compression::new())
    .hoop(Cors::permissive())
    .hoop(Timeout::new(Duration::from_secs(30)));
```

## Middleware Order

Middleware executes in onion model:

```rust
Router::new()
    .hoop(middleware_a)  // Runs first (outer)
    .hoop(middleware_b)  // Runs second
    .hoop(middleware_c)  // Runs third (inner)
    .get(handler);       // Runs last

// Execution order:
// middleware_a (before) -> middleware_b (before) -> middleware_c (before)
// -> handler
// -> middleware_c (after) -> middleware_b (after) -> middleware_a (after)
```

## Best Practices

1. Use `ctrl.call_next()` to continue execution
2. Use `ctrl.skip_rest()` to stop early
3. Store shared data in `Depot`
4. Apply middleware at appropriate router level
5. Order middleware by dependency (auth before authorization)
6. Use built-in middleware when available
7. Keep middleware focused on single concern
