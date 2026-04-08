---
name: salvo-realtime
description: Implement real-time features using WebSocket and Server-Sent Events (SSE). Use for chat applications, live updates, notifications, and bidirectional communication.
version: 0.89.3
tags: [realtime, websocket, sse, overview]
---

# Salvo Real-time Communication

Overview / decision guide for Salvo's real-time transports. For full APIs see **salvo-websocket** and **salvo-sse**.

## Pick a transport

|                       | WebSocket            | SSE                        |
|-----------------------|----------------------|----------------------------|
| Direction             | Full-duplex          | Server → client only       |
| Transport             | Custom over TCP      | Plain HTTP (`text/event-stream`) |
| Reconnection          | Manual (heartbeat)   | Automatic (browser)        |
| Binary payloads       | Yes                  | No (text only)             |
| Proxies / firewalls   | Occasionally blocked | Same as any HTTP request   |
| Salvo feature         | `websocket`          | `sse`                      |

Use **WebSocket** for chat, multiplayer games, collaborative editing, any bidirectional stream. Use **SSE** for notifications, tickers, progress bars, dashboards — anything that only pushes from server.

## WebSocket quick start

```rust
use salvo::prelude::*;
use salvo::websocket::WebSocketUpgrade;

#[handler]
async fn ws(req: &mut Request, res: &mut Response) -> Result<(), StatusError> {
    WebSocketUpgrade::new()
        .upgrade(req, res, |mut ws| async move {
            while let Some(Ok(msg)) = ws.recv().await {
                if ws.send(msg).await.is_err() { return; }
            }
        })
        .await
}

#[tokio::main]
async fn main() {
    let router = Router::new().push(Router::with_path("ws").goal(ws));
    Server::new(TcpListener::new("0.0.0.0:8080").bind().await).serve(router).await;
}
```

## SSE quick start

```rust
use std::{convert::Infallible, time::Duration};
use futures_util::StreamExt;
use salvo::prelude::*;
use salvo::sse::{self, SseEvent};
use tokio::time::interval;
use tokio_stream::wrappers::IntervalStream;

#[handler]
async fn events(res: &mut Response) {
    let mut n: u64 = 0;
    let stream = IntervalStream::new(interval(Duration::from_secs(1))).map(move |_| {
        n += 1;
        Ok::<_, Infallible>(SseEvent::default().text(n.to_string()))
    });
    sse::stream(res, stream);
}
```

## Fan-out patterns

### Per-client channel map (WebSocket broadcast)

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{RwLock, mpsc};
use salvo::websocket::Message;

type Tx = mpsc::UnboundedSender<Result<Message, salvo::Error>>;
type Users = Arc<RwLock<HashMap<usize, Tx>>>;

async fn broadcast(users: &Users, sender_id: usize, text: &str) {
    let out = format!("<#{sender_id}>: {text}");
    for (&uid, tx) in users.read().await.iter() {
        if uid != sender_id { let _ = tx.send(Ok(Message::text(out.clone()))); }
    }
}
```

### Pub/sub via `broadcast` (fits both transports)

```rust
use tokio::sync::broadcast;

#[derive(Clone)]
struct Hub { tx: broadcast::Sender<String> }

impl Hub {
    fn new(cap: usize) -> Self { let (tx, _) = broadcast::channel(cap); Self { tx } }
    fn publish(&self, msg: String) { let _ = self.tx.send(msg); }
    fn subscribe(&self) -> broadcast::Receiver<String> { self.tx.subscribe() }
}
```

Share a `Hub` via `Depot` (typically stashed by an `affix_state` hoop) and wrap `BroadcastStream` into an SSE stream or forward into a WebSocket sink.

### Rooms

```rust
type Rooms = Arc<RwLock<HashMap<String, HashMap<usize, Tx>>>>;
```

Insert on join, remove on disconnect (both while holding the write guard, no `.await` in between).

## Connection lifecycle

- **Auth before upgrade**: validate JWT / session on the route before `WebSocketUpgrade::upgrade`. Query params and headers are only accessible before the closure runs.
- **Track counts**: `AtomicUsize` on connect/disconnect if you want metrics without locking.
- **WebSocket heartbeat**: server-side `Message::ping` every 30 s inside a `tokio::select!` alongside `recv()`.
- **SSE keep-alive**: use `SseKeepAlive::new(stream).max_interval(Duration::from_secs(15))` (note: `max_interval`, not `interval`).
- **Back-pressure**: slow clients will fill unbounded channels — prefer bounded `mpsc` for per-client queues, or drop lagged subscribers on `broadcast::error::RecvError::Lagged`.
- **Cap fanout**: pair with `salvo-concurrency-limiter` to bound total subscribers.

## Mixing both transports

```rust
let router = Router::new()
    .push(Router::with_path("chat").goal(ws_chat_handler))         // bidirectional
    .push(Router::with_path("notifications").get(sse_notifications)) // push-only
    .push(Router::with_path("feed").get(sse_feed));
```

## Related Skills

- **salvo-websocket**: Full WebSocket reference
- **salvo-sse**: Full SSE reference
- **salvo-concurrency-limiter**: Cap concurrent real-time connections
- **salvo-auth**: Authenticate upgrade handshakes
