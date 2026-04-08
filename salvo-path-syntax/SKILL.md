---
name: salvo-path-syntax
description: Path parameter syntax guide for Salvo routing. Explains the `{}` syntax (v0.76+) vs deprecated `<>` syntax, with migration examples.
version: 0.89.3
tags: [core, routing, path-syntax, migration]
---

# Salvo Path Parameter Syntax

Salvo 0.76+ uses curly-brace syntax `{...}` for path parameters. The old angle-bracket form `<...>` (pre-0.76) is removed and will not match.

## Current syntax (0.76+)

```rust
Router::with_path("users/{id}")                 // basic
Router::with_path("users/{id:num}")             // typed (num|i32|i64|u32|u64)
Router::with_path(r"users/{id|\d+}")            // regex
Router::with_path("files/{*}")                  // single-segment wildcard
Router::with_path("files/{*name}")              // named single-segment wildcard
Router::with_path("static/{**path}")            // rest-of-path wildcard
```

## Migration from <> to {}

| Before 0.76 | 0.76+ |
|---|---|
| `<id>` | `{id}` |
| `<id:num>` | `{id:num}` |
| `<id\|\d+>` | `{id\|\d+}` |
| `<*>` / `<*name>` | `{*}` / `{*name}` |
| `<**>` / `<**path>` | `{**}` / `{**path}` |

Mechanical substitution: `<` to `{`, `>` to `}` in every `with_path(...)` / `path(...)` argument. Update any tests asserting on path strings.

BAD (pre-0.76, will not match in current Salvo):
```rust
Router::with_path("users/<id>/posts/<**rest>")
```

GOOD:
```rust
Router::with_path("users/{id}/posts/{**rest}")
```

## Parameter types

```rust
Router::with_path("users/{id:num}")   // any integer
Router::with_path("users/{id:i32}")   // signed 32-bit
Router::with_path("users/{id:i64}")
Router::with_path("users/{id:u32}")
Router::with_path("users/{id:u64}")

// Regex: characters between `|` and `}` form the pattern
Router::with_path(r"posts/{slug|[a-z0-9-]+}")
Router::with_path(r"users/{id|[0-9a-f-]{36}}")  // UUID
```

Typed and regex parameters reject non-matching values at the routing layer (returns 404), so handlers can `unwrap`.

## Accessing params

```rust
#[handler]
async fn show(req: &mut Request) -> String {
    let id: i64 = req.param("id").unwrap();
    format!("id = {id}")
}

#[handler]
async fn serve(req: &mut Request) -> String {
    let rest: String = req.param("path").unwrap();  // from {**path}
    format!("serving {rest}")
}
```

`req.param::<T>(name)` returns `Option<T>` and deserializes via serde.

## Gotchas

- Do not mix syntaxes: `"users/{id}/posts/<pid>"` will not compile-error but the `<pid>` segment is treated as a literal.
- Wildcards `{**}` are greedy and capture trailing slashes — name them for clarity: `{**path}`.
- `{*}` matches exactly one segment; `{**}` matches zero or more. Use `{*name}` when you need a single non-empty segment.

## Related Skills

- **salvo-routing**: Full routing configuration guide
- **salvo-basic-app**: Basic application setup
