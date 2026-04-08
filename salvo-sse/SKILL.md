---
name: salvo-sse
description: Implement Server-Sent Events for real-time server-to-client updates. Use for live feeds, notifications, and streaming data.
version: 0.89.3
tags: [realtime, sse, server-sent-events, streaming]
---

# Salvo Server-Sent Events (SSE)

`salvo::sse::stream(res, event_stream)` writes a `text/event-stream` body from any `TryStream<Ok = SseEvent>` whose error implements `std::error::Error + Send + Sync + 'static`. `SseKeepAlive` wraps a stream and injects periodic comment frames when idle.

## Setup

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["sse"] }
futures-util = "0.3"
tokio = { version = "1", features = ["full"] }
tokio-stream = "0.1"
async-stream = "0.3"   # only for async_stream::stream! macro
```

## Counter

```rust
use std::convert::Infallible;
use std::time::Duration;
use futures_util::StreamExt;
use salvo::prelude::*;
use salvo::sse::{self, SseEvent};
use tokio::time::interval;
use tokio_stream::wrappers::IntervalStream;

#[handler]
async fn sse_counter(res: &mut Response) {
    let mut counter: u64 = 0;
    let event_stream = IntervalStream::new(interval(Duration::from_secs(1)))
        .map(move |_| {
            counter += 1;
            Ok::<_, Infallible>(SseEvent::default().text(counter.to_string()))
        });
    sse::stream(res, event_stream);
}

#[tokio::main]
async fn main() {
    let router = Router::new().push(Router::with_path("events").get(sse_counter));
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

Client: `new EventSource('/events')` — `<!-- minimal JS client omitted -->`.

## `SseEvent` builder

All setters except `json` return `Self`; `json` returns `Result<Self, serde_json::Error>`.

```rust
use std::time::Duration;
use salvo::sse::SseEvent;

SseEvent::default().text("hello");                              // data:hello
SseEvent::default().name("notification").text("new message");   // event:notification + data
SseEvent::default().name("update").json(&value)?;               // data:<serialized JSON>
SseEvent::default().id("msg-123").text("...");                  // id:msg-123
SseEvent::default().retry(Duration::from_secs(5)).text("...");  // retry:5000
SseEvent::default().comment("keep-alive");                      // :keep-alive (ignored by clients)
```

Text data is split on `\n` into multiple `data:` lines automatically.

## Keep-alive

`SseKeepAlive` emits a comment frame after `max_interval` of inactivity; any real event from the inner stream resets the timer.

```rust
use std::time::Duration;
use salvo::sse::SseKeepAlive;

#[handler]
async fn sse_with_keepalive(res: &mut Response) {
    let stream = create_event_stream();
    SseKeepAlive::new(stream)
        .max_interval(Duration::from_secs(15))  // NOTE: max_interval, not interval
        .comment("ping")                         // NOTE: comment(), not text()
        .stream(res);
}
```

## Broadcast channel → SSE

Most real apps wire a `tokio::sync::broadcast` receiver into an async stream:

```rust
use std::convert::Infallible;
use std::time::Duration;
use salvo::prelude::*;
use salvo::sse::{SseEvent, SseKeepAlive};
use tokio::sync::broadcast;

#[derive(Clone, serde::Serialize)]
struct Notification { id: u64, title: String, body: String }

#[derive(Clone)]
struct Hub { tx: broadcast::Sender<Notification> }

impl Hub {
    fn new() -> Self {
        let (tx, _) = broadcast::channel(128);
        Self { tx }
    }
}

#[handler]
async fn notifications(depot: &mut Depot, res: &mut Response) {
    let hub = depot.obtain::<Hub>().unwrap().clone();
    let mut rx = hub.tx.subscribe();

    let stream = async_stream::stream! {
        while let Ok(n) = rx.recv().await {
            if let Ok(evt) = SseEvent::default()
                .name("notification")
                .id(n.id.to_string())
                .json(&n)
            {
                yield Ok::<_, Infallible>(evt);
            }
        }
    };

    SseKeepAlive::new(stream)
        .max_interval(Duration::from_secs(30))
        .stream(res);
}
```

Each subscriber gets its own `Receiver`; broadcast lag (`RecvError::Lagged`) should be handled if clients can fall behind.

## Chat room (pub/sub via broadcast)

```rust
use std::convert::Infallible;
use futures_util::StreamExt;
use salvo::prelude::*;
use salvo::sse::{self, SseEvent};
use tokio::sync::broadcast;
use tokio_stream::wrappers::BroadcastStream;

#[derive(Clone)]
struct Chat { tx: broadcast::Sender<String> }

#[handler]
async fn sse_subscribe(depot: &mut Depot, res: &mut Response) {
    let chat = depot.obtain::<Chat>().unwrap().clone();
    let stream = BroadcastStream::new(chat.tx.subscribe())
        .filter_map(|item| async move {
            item.ok().map(|text| Ok::<_, Infallible>(SseEvent::default().text(text)))
        });
    sse::stream(res, stream);
}

#[handler]
async fn post_message(depot: &mut Depot, req: &mut Request) {
    let chat = depot.obtain::<Chat>().unwrap().clone();
    let body = req.payload().await.map(|b| String::from_utf8_lossy(b).into_owned()).unwrap_or_default();
    let _ = chat.tx.send(body);
}
```

## Last-Event-ID reconnect

Browsers auto-reconnect and resend the last seen event id in the `Last-Event-ID` header. Parse it and replay from there:

```rust
#[handler]
async fn sse_with_ids(req: &mut Request, res: &mut Response) {
    let last_id: u64 = req.header("Last-Event-ID").unwrap_or(0);
    let stream = async_stream::stream! {
        let mut id = last_id + 1;
        loop {
            tokio::time::sleep(std::time::Duration::from_secs(1)).await;
            yield Ok::<_, std::convert::Infallible>(
                SseEvent::default().id(id.to_string()).text(format!("Event {id}"))
            );
            id += 1;
        }
    };
    salvo::sse::stream(res, stream);
}
```

## Gotchas

- `SseKeepAlive` methods are `max_interval` and `comment` — **not** `interval` / `text`.
- The stream's error type must implement `std::error::Error + Send + Sync + 'static`. Use `Infallible` for infinite streams, or `salvo::Error` / `Box<dyn Error + Send + Sync>`.
- SSE is text-only; encode binary as base64 inside the event data.
- `sse::stream()` sets `Content-Type: text/event-stream` and `Cache-Control: no-cache` — don't overwrite them afterwards.
- Each subscriber holds a connection; pair with `salvo-concurrency-limiter` if client counts can spike.

## Related Skills

- **salvo-websocket**: WebSocket for bidirectional communication
- **salvo-realtime**: Overview of real-time communication options
- **salvo-concurrency-limiter**: Cap simultaneous subscribers
