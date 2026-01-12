---
name: salvo-routing
description: Configure Salvo routers with path parameters, nested routes, and filters. Use for complex routing structures and RESTful APIs.
---

# Salvo Routing

This skill helps configure advanced routing patterns in Salvo applications.

## Path Patterns

### Static Paths
```rust
Router::with_path("users").get(list_users)
```

### Path Parameters
```rust
// Basic parameter
Router::with_path("users/{id}").get(show_user)

// Typed parameter (num, i32, i64, etc.)
Router::with_path("users/{id:num}").get(show_user)

// Regex pattern
Router::with_path(r"users/{id|\d+}").get(show_user)

// Wildcard (captures rest of path)
Router::with_path("files/{**path}").get(serve_file)
```

Accessing parameters:
```rust
#[handler]
async fn show_user(req: &mut Request) -> String {
    let id = req.param::<i64>("id").unwrap();
    format!("User ID: {}", id)
}
```

## Nested Routers

### Tree Structure
```rust
let router = Router::new()
    .push(
        Router::with_path("api/v1")
            .push(
                Router::with_path("users")
                    .get(list_users)
                    .post(create_user)
                    .push(
                        Router::with_path("{id}")
                            .get(show_user)
                            .patch(update_user)
                            .delete(delete_user)
                    )
            )
            .push(
                Router::with_path("posts")
                    .get(list_posts)
                    .post(create_post)
            )
    );
```

### Flat Structure
```rust
let router = Router::new()
    .push(Router::with_path("api/v1/users").get(list_users).post(create_user))
    .push(Router::with_path("api/v1/users/{id}").get(show_user).patch(update_user).delete(delete_user));
```

## HTTP Methods

```rust
Router::new()
    .get(handler)      // GET
    .post(handler)     // POST
    .put(handler)      // PUT
    .patch(handler)    // PATCH
    .delete(handler)   // DELETE
    .head(handler)     // HEAD
    .options(handler); // OPTIONS
```

## Filters

Routers use filters for matching:

```rust
use salvo::routing::filters;

// Path filter
Router::with_filter(filters::path("users"))

// Method filter
Router::with_filter(filters::get())

// Combined filters
Router::with_filter(filters::path("users").and(filters::get()))
```

## Router Groups

Share common path prefix:

```rust
let api_v1 = Router::with_path("api/v1");
let users = Router::with_path("users")
    .get(list_users)
    .post(create_user);
let posts = Router::with_path("posts")
    .get(list_posts);

let router = Router::new()
    .push(api_v1.push(users).push(posts));
```

## Best Practices

1. Use tree structure for complex APIs
2. Use flat structure for simple routes
3. Type path parameters when possible (`<id:num>`)
4. Group related routes under common paths
5. Use descriptive parameter names
6. Prefer nested routers over long path strings
