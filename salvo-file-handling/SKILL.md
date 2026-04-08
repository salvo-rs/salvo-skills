---
name: salvo-file-handling
description: Handle file uploads (single/multiple), downloads, and multipart forms. Use for file management, image uploads, and content delivery.
version: 0.89.3
tags: [data, file-upload, multipart, download]
---

# Salvo File Handling

## Setup

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["size-limiter"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread", "fs"] }
```

`req.file()`/`req.files()` return `Option<&FilePart>` / `Option<&Vec<FilePart>>`. Salvo writes multipart parts to a temp dir, so `file.path()` points at an on-disk file — copy or rename it to the final location; don't rely on it sticking around after the handler returns.

## Single File Upload

```rust
use std::path::Path;
use salvo::prelude::*;

#[handler]
async fn upload(req: &mut Request, res: &mut Response) {
    let Some(file) = req.file("file").await else {
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(Text::Plain("No file in request"));
        return;
    };

    let dest = format!("temp/{}", file.name().unwrap_or("file"));
    match tokio::fs::copy(file.path(), Path::new(&dest)).await {
        Ok(_) => res.render(Text::Plain(format!("Uploaded to {dest}"))),
        Err(e) => {
            res.status_code(StatusCode::INTERNAL_SERVER_ERROR);
            res.render(Text::Plain(format!("Upload failed: {e}")));
        }
    }
}

#[tokio::main]
async fn main() {
    tokio::fs::create_dir_all("temp").await.unwrap();
    let router = Router::new().post(upload);
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

HTML form: `<form method="post" enctype="multipart/form-data"><input type="file" name="file"/></form>`.

## Multiple Files

```rust
#[handler]
async fn upload_files(req: &mut Request, res: &mut Response) {
    let Some(files) = req.files("files").await else {
        res.status_code(StatusCode::BAD_REQUEST);
        return;
    };

    let mut uploaded = Vec::with_capacity(files.len());
    for file in files {
        let dest = format!("temp/{}", file.name().unwrap_or("file"));
        if tokio::fs::copy(file.path(), &dest).await.is_ok() {
            uploaded.push(dest);
        }
    }
    res.render(Json(uploaded));
}
```

For any-field upload use `req.all_files().await -> Vec<&FilePart>`; for single unknown-key, `req.first_file().await`.

## Validation

`FilePart::content_type()` returns `Option<mime::Mime>`, not a `&str`. Compare with `.type_()`/`.subtype()` or `.essence_str()`.

```rust
use salvo::prelude::*;

#[handler]
async fn upload_image(req: &mut Request, res: &mut Response) {
    let Some(file) = req.file("image").await else {
        res.status_code(StatusCode::BAD_REQUEST);
        res.render("No file");
        return;
    };

    let ct = file.content_type().map(|m| m.essence_str().to_owned()).unwrap_or_default();
    const ALLOWED: &[&str] = &["image/jpeg", "image/png", "image/gif", "image/webp"];
    if !ALLOWED.contains(&ct.as_str()) {
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(format!("Bad type: {ct}"));
        return;
    }

    if file.size() > 5 * 1024 * 1024 {
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(format!("Too large: {} bytes", file.size()));
        return;
    }

    let filename = file.name().unwrap_or("unnamed");
    let ext = filename.rsplit('.').next().unwrap_or("").to_lowercase();
    let unique = format!("{}.{ext}", uuid::Uuid::new_v4());
    let dest = format!("uploads/{unique}");

    if tokio::fs::copy(file.path(), &dest).await.is_err() {
        res.status_code(StatusCode::INTERNAL_SERVER_ERROR);
        return;
    }

    res.render(Json(serde_json::json!({
        "filename": unique,
        "original": filename,
        "size": file.size(),
        "content_type": ct,
    })));
}
```

## Size Limiting

Requires `size-limiter` feature. Apply as middleware to limit request body size.

```rust
use salvo::prelude::*;
use salvo::size_limiter::max_size;

let router = Router::with_path("upload")
    .hoop(max_size(10 * 1024 * 1024))  // 10 MiB
    .post(upload);
```

## File Downloads with NamedFile

`NamedFile::builder(path)` returns a `NamedFileBuilder`. Key methods: `attached_name(name)` (sets `Content-Disposition: attachment`), `content_type(mime)` (takes `mime::Mime`, not a string), `use_etag(bool)`, `disposition_type(ty)`, `send(headers, res)`.

```rust
use salvo::fs::NamedFile;
use salvo::prelude::*;

#[handler]
async fn download(req: &mut Request, res: &mut Response) {
    let filename: String = req.param("filename").unwrap();
    let path = format!("files/{filename}");

    if NamedFile::builder(&path)
        .attached_name(&filename)
        .send(req.headers(), res)
        .await
        .is_err()
    {
        res.status_code(StatusCode::NOT_FOUND);
        res.render("File not found");
    }
}
```

Inline view (no `attached_name`) with an explicit MIME:

```rust
use salvo::fs::NamedFile;
use salvo::prelude::*;

#[handler]
async fn view_pdf(req: &mut Request, res: &mut Response) {
    let _ = NamedFile::builder("docs/report.pdf")
        .content_type(mime::APPLICATION_PDF)
        .send(req.headers(), res)
        .await;
}
```

`NamedFile::send` handles `If-None-Match`, `If-Modified-Since`, and `Range` headers automatically.

## Protected Downloads

```rust
#[handler]
async fn protected_download(
    req: &mut Request,
    depot: &mut Depot,
    res: &mut Response,
) {
    let Ok(_user) = depot.obtain::<User>() else {
        res.status_code(StatusCode::UNAUTHORIZED);
        return;
    };

    let filename: String = req.param("filename").unwrap();
    if filename.contains("..") || filename.contains('/') || filename.contains('\\') {
        res.status_code(StatusCode::BAD_REQUEST);
        return;
    }

    let path = format!("private/{filename}");
    if NamedFile::builder(&path)
        .attached_name(&filename)
        .send(req.headers(), res)
        .await
        .is_err()
    {
        res.status_code(StatusCode::NOT_FOUND);
    }
}
```

## Form with File + Text Fields

```rust
#[handler]
async fn update_profile(req: &mut Request, res: &mut Response) {
    let name = req.form::<String>("name").await.unwrap_or_default();
    let bio = req.form::<String>("bio").await.unwrap_or_default();

    let avatar = if let Some(file) = req.file("avatar").await {
        let filename = format!("{}.jpg", uuid::Uuid::new_v4());
        let _ = tokio::fs::copy(file.path(), format!("uploads/{filename}")).await;
        Some(filename)
    } else {
        None
    };

    res.render(Json(serde_json::json!({ "name": name, "bio": bio, "avatar": avatar })));
}
```

Note: `form_data` and `file`/`files`/`form` all parse multipart lazily and cache. Interleave calls freely.

## Raw Body Upload (non-multipart)

```rust
use tokio::io::AsyncWriteExt;
use salvo::prelude::*;

#[handler]
async fn raw_upload(req: &mut Request, res: &mut Response) -> Result<(), StatusError> {
    let filename: String = req.query("filename").unwrap_or_else(|| "upload".into());
    let body = req.payload().await.map_err(|_| StatusError::bad_request())?;
    let mut f = tokio::fs::File::create(format!("uploads/{filename}")).await
        .map_err(|_| StatusError::internal_server_error())?;
    f.write_all(body).await.map_err(|_| StatusError::internal_server_error())?;
    res.render(format!("{} bytes", body.len()));
    Ok(())
}
```

`payload()` honors the request's `max_size`; use `payload_with_max_size(n)` for a custom cap.

## OpenAPI File Upload

```rust
use salvo::oapi::extract::*;
use salvo::prelude::*;
use serde::Serialize;

#[derive(Serialize, ToSchema)]
struct UploadResult { filename: String, size: u64 }

#[endpoint(
    tags("files"),
    request_body(content = "multipart/form-data"),
)]
async fn upload(req: &mut Request) -> Result<Json<UploadResult>, StatusError> {
    let file = req.file("file").await
        .ok_or_else(|| StatusError::bad_request().brief("No file"))?;
    let filename = file.name().unwrap_or("unnamed").to_string();
    let size = file.size();
    tokio::fs::copy(file.path(), format!("uploads/{filename}")).await
        .map_err(|_| StatusError::internal_server_error())?;
    Ok(Json(UploadResult { filename, size }))
}
```

## Salvo-Specific Notes

- Temp multipart files live only for the request lifetime — copy them before returning.
- Always validate `filename` against `..`, `/`, `\` before using it as a path segment.
- `content_type()` returns `Option<Mime>`; use `.essence_str()` for a comparable string.
- `NamedFile::content_type()` takes `mime::Mime`, not `&str`.

## Related Skills

- **salvo-static-files**: Serve static files and directories
- **salvo-data-extraction**: Form fields alongside files
- **salvo-openapi**: Document file upload endpoints
