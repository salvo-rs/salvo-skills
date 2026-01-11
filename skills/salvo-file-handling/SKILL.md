---
name: salvo-file-handling
description: Handle file uploads, downloads, and serve static files. Use for file management, image uploads, and static asset serving.
---

# Salvo File Handling

This skill helps handle file uploads, downloads, and static file serving in Salvo applications.

## File Upload

### Single File Upload

```rust
use salvo::prelude::*;

#[handler]
async fn upload_file(req: &mut Request) -> Result<String, StatusError> {
    let file = req.file("file").await
        .ok_or_else(|| StatusError::bad_request())?;

    let filename = file.name().unwrap_or("unnamed");
    let dest = format!("uploads/{}", filename);

    tokio::fs::copy(file.path(), &dest).await
        .map_err(|_| StatusError::internal_server_error())?;

    Ok(format!("File uploaded: {}", filename))
}
```

### Multiple File Upload

```rust
#[handler]
async fn upload_files(req: &mut Request) -> Result<Json<Vec<String>>, StatusError> {
    let files = req.files("files").await
        .ok_or_else(|| StatusError::bad_request())?;

    let mut uploaded = Vec::new();

    for file in files {
        let filename = file.name().unwrap_or("unnamed");
        let dest = format!("uploads/{}", filename);

        tokio::fs::copy(file.path(), &dest).await
            .map_err(|_| StatusError::internal_server_error())?;

        uploaded.push(filename.to_string());
    }

    Ok(Json(uploaded))
}
```

### With Validation

```rust
#[handler]
async fn upload_image(req: &mut Request) -> Result<String, StatusError> {
    let file = req.file("image").await
        .ok_or_else(|| StatusError::bad_request())?;

    // Validate file type
    let content_type = file.content_type()
        .ok_or_else(|| StatusError::bad_request())?;

    if !content_type.starts_with("image/") {
        return Err(StatusError::bad_request().with_detail("Only images allowed"));
    }

    // Validate file size (5MB max)
    if file.size() > 5 * 1024 * 1024 {
        return Err(StatusError::bad_request().with_detail("File too large"));
    }

    let filename = file.name().unwrap_or("unnamed");
    let dest = format!("uploads/{}", filename);

    tokio::fs::copy(file.path(), &dest).await
        .map_err(|_| StatusError::internal_server_error())?;

    Ok(format!("Image uploaded: {}", filename))
}
```

## File Download

### Send File

```rust
use salvo::fs::NamedFile;

#[handler]
async fn download_file(req: &mut Request) -> Result<NamedFile, StatusError> {
    let filename = req.param::<String>("filename")
        .ok_or_else(|| StatusError::bad_request())?;

    let path = format!("uploads/{}", filename);

    NamedFile::open(path).await
        .map_err(|_| StatusError::not_found())
}
```

### With Custom Headers

```rust
#[handler]
async fn download_file(req: &mut Request, res: &mut Response) -> Result<(), StatusError> {
    let filename = req.param::<String>("filename")
        .ok_or_else(|| StatusError::bad_request())?;

    let path = format!("uploads/{}", filename);

    let file = NamedFile::open(&path).await
        .map_err(|_| StatusError::not_found())?;

    res.headers_mut().insert(
        "Content-Disposition",
        format!("attachment; filename=\"{}\"", filename).parse().unwrap(),
    );

    res.render(file);
    Ok(())
}
```

## Static File Serving

### Serve Directory

```rust
use salvo::serve_static::StaticDir;

#[tokio::main]
async fn main() {
    let router = Router::new()
        .push(
            Router::with_path("static/<**path>")
                .get(StaticDir::new("public"))
        );

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

### With Directory Listing

```rust
let router = Router::new()
    .push(
        Router::with_path("files/<**path>")
            .get(StaticDir::new("uploads").listing(true))
    );
```

### Serve Embedded Files

```rust
use rust_embed::RustEmbed;
use salvo::serve_static::static_embed;

#[derive(RustEmbed)]
#[folder = "public"]
struct Assets;

#[tokio::main]
async fn main() {
    let router = Router::new()
        .push(
            Router::with_path("assets/<**path>")
                .get(static_embed::<Assets>())
        );

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

## Streaming Files

### Stream Large Files

```rust
use tokio::fs::File;
use tokio::io::AsyncReadExt;

#[handler]
async fn stream_file(req: &mut Request, res: &mut Response) -> Result<(), StatusError> {
    let filename = req.param::<String>("filename")
        .ok_or_else(|| StatusError::bad_request())?;

    let path = format!("large_files/{}", filename);
    let mut file = File::open(&path).await
        .map_err(|_| StatusError::not_found())?;

    let mut buffer = Vec::new();
    file.read_to_end(&mut buffer).await
        .map_err(|_| StatusError::internal_server_error())?;

    res.render(buffer);
    Ok(())
}
```

## File Upload with OpenAPI

```rust
use salvo::oapi::extract::*;

#[derive(ToSchema)]
struct UploadResponse {
    filename: String,
    size: u64,
}

#[endpoint(
    tags("files"),
    request_body(content = "multipart/form-data")
)]
async fn upload_file(req: &mut Request) -> Result<Json<UploadResponse>, StatusError> {
    let file = req.file("file").await
        .ok_or_else(|| StatusError::bad_request())?;

    let filename = file.name().unwrap_or("unnamed").to_string();
    let size = file.size();
    let dest = format!("uploads/{}", filename);

    tokio::fs::copy(file.path(), &dest).await
        .map_err(|_| StatusError::internal_server_error())?;

    Ok(Json(UploadResponse { filename, size }))
}
```

## Size Limiting

```rust
use salvo::size_limiter::max_size;

let router = Router::new()
    .push(
        Router::with_path("upload")
            .hoop(max_size(5 * 1024 * 1024))  // 5MB limit
            .post(upload_file)
    );
```

## Best Practices

1. Validate file types and sizes
2. Use unique filenames to prevent overwrites
3. Store files outside web root for security
4. Implement virus scanning for uploads
5. Use streaming for large files
6. Set appropriate Content-Type headers
7. Implement access control for downloads
8. Clean up temporary files
9. Use CDN for static assets in production
10. Implement rate limiting for uploads
