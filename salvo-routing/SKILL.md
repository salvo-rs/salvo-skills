---
name: salvo-routing
description: Configure Salvo routers with path parameters, nested routes, and filters. Use for complex routing structures and RESTful APIs.
version: 0.89.3
tags: [core, routing, path-params, filters]
---

# Salvo Routing

Salvo treats path, method, and custom conditions uniformly as filters. Routers are composable (`push`), reusable across multiple mount points, and share middleware with handlers via `hoop()`.

## Path parameters

Salvo uses `{name}` syntax (since v0.76; the older `<name>` form is removed).

```rust
Router::with_path("users").get(list_users)            // static
Router::with_path("users/{id}").get(show_user)        // basic param
Router::with_path("users/{id:num}").get(show_user)    // typed: num, i32, i64, u32, u64
Router::with_path(r"users/{id|\d+}").get(show_user)   // regex
Router::with_path("files/{*rest}").get(serve_file)    // single segment wildcard
Router::with_path("static/{**path}").get(serve_static) // multi-segment wildcard
```

Access in a handler via `req.param::<T>("name")` which returns `Option<T>`:

```rust
#[handler]
async fn show_user(req: &mut Request) -> String {
    let id: i64 = req.param("id").unwrap();
    format!("User ID: {id}")
}
```

## Wildcards

- `{*}` / `{*name}` — matches a single path segment
- `{**}` / `{**name}` — matches all remaining segments (including slashes)

## HTTP methods

```rust
Router::new()
    .get(h).post(h).put(h).patch(h).delete(h).head(h).options(h);
```

Each method call creates a child router filtered to that method. `goal(h)` sets the handler without a method filter.

## Nesting with push()

```rust
let router = Router::with_path("api/v1")
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
    .push(Router::with_path("posts").get(list_posts).post(create_post));
```

## Route composition

Routers are values — extract reusable subtrees into functions and mount them under different prefixes:

```rust
fn user_routes() -> Router {
    Router::with_path("users")
        .get(list_users)
        .post(create_user)
        .push(Router::with_path("{id}").get(get_user).delete(delete_user))
}

let router = Router::new()
    .push(Router::with_path("api/v1").push(user_routes()))
    .push(Router::with_path("api/v2").push(user_routes()));
```

## Middleware via hoop()

`hoop()` attaches middleware. It applies to the current router and all descendants, only when the filter chain matches:

```rust
let router = Router::new()
    .hoop(logger)                             // global
    .push(
        Router::with_path("api")
            .hoop(auth_check)                 // only /api/**
            .push(Router::with_path("users").get(list_users))
    );
```

Use `hoop_when(handler, |req, depot| bool)` for conditional middleware.

## Custom filters

For conditional matching that does not consume a path segment, use `filter_fn`:

```rust
Router::with_path("admin")
    .filter_fn(|req, _state| req.header::<String>("x-admin-key").is_some())
    .get(admin_handler);
```

For a fully custom `Filter`, implement the async trait (`salvo::routing::filters::Filter`). The `filter` method takes `&mut PathState`; to match and consume a segment you must also advance `state.cursor` and insert into `state.params`. Prefer typed/regex path params (e.g. `{id:num}`, `{id|\d+}`) for common cases — they handle all of this correctly.

Built-in filter helpers live in `salvo::routing::filters`: `path()`, `get()`, `post()`, `host()`, `port()`, `scheme()`. Combine with `.and(...)` / `.or(...)`.

## Redirects

```rust
use salvo::prelude::*;

#[handler]
async fn perm(res: &mut Response) { res.render(Redirect::permanent("/new")); }
#[handler]
async fn temp(res: &mut Response) { res.render(Redirect::found("/tmp")); }
#[handler]
async fn other(res: &mut Response) { res.render(Redirect::see_other("/page")); }
```

## Gotchas

- Path filters match segment-by-segment; nested `with_path("api/v1")` then `with_path("users")` matches `api/v1/users`, not `users` alone.
- Param names must be unique within a single path. `{id}` and `{id}` in the same path will clash.
- `Router::new().path("api")` mutates in place; `Router::with_path("api")` constructs fresh. Use the latter to start a branch.

## Related Skills

- **salvo-basic-app**: Basic application setup and handler patterns
- **salvo-path-syntax**: Path parameter syntax guide and migration
- **salvo-middleware**: Attach middleware to routes with hoop()
