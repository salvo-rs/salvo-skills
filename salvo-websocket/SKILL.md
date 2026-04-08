---
name: salvo-websocket
description: Implement WebSocket connections for real-time bidirectional communication. Use for chat, live updates, gaming, and collaborative features.
version: 0.89.3
tags: [realtime, websocket, bidirectional, chat]
---

# Salvo WebSocket

`WebSocketUpgrade` from `salvo-extra` turns a GET handler into a WebSocket endpoint. `WebSocket` implements `Stream<Item = Result<Message, Error>>` + `Sink<Message, Error = Error>`, so `StreamExt::split()`, `recv()`, and `send()` all work.

## Setup

```toml
[dependencies]
salvo = { version = "0.89.3", features = ["websocket"] }
futures-util = "0.3"
tokio = { version = "1", features = ["full"] }
tokio-stream = "0.1"
```

## Echo server

```rust
use salvo::prelude::*;
use salvo::websocket::WebSocketUpgrade;

#[handler]
async fn ws_handler(req: &mut Request, res: &mut Response) -> Result<(), StatusError> {
    WebSocketUpgrade::new()
        .upgrade(req, res, |mut ws| async move {
            while let Some(Ok(msg)) = ws.recv().await {
                if ws.send(msg).await.is_err() {
                    return;
                }
            }
        })
        .await
}

#[tokio::main]
async fn main() {
    let router = Router::new().push(Router::with_path("ws").goal(ws_handler));
    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

Client: `new WebSocket('ws://host/ws')` — `<!-- minimal JS client omitted -->`.

## `WebSocketUpgrade` configuration

```rust
WebSocketUpgrade::new()
    .protocols(&["graphql-ws", "graphql-transport-ws"]) // subprotocol allowlist
    .accept_any_protocol()           // OR: echo client's first offered protocol (don't use for auth tokens)
    .max_message_size(1 << 20)       // default 64 MiB
    .max_frame_size(256 * 1024)      // default 16 MiB
    .write_buffer_size(128 * 1024)   // default 128 KiB
    .max_write_buffer_size(usize::MAX)
    .accept_unmasked_frames(false)
    .upgrade(req, res, |ws| async move { /* ... */ })
    .await
```

Note: `accept_any_protocol()` takes precedence over `protocols()`.

## Query params & pre-upgrade state

Parse request data **before** `.upgrade()` — `req` isn't available inside the closure.

```rust
#[derive(serde::Deserialize, Debug, Clone)]
struct ConnectParams { user_id: usize, name: String }

#[handler]
async fn connect(req: &mut Request, res: &mut Response) -> Result<(), StatusError> {
    let params = req.parse_queries::<ConnectParams>().map_err(|_| StatusError::bad_request())?;
    WebSocketUpgrade::new()
        .upgrade(req, res, move |mut ws| async move {
            tracing::info!(?params, "connected");
            while let Some(Ok(msg)) = ws.recv().await {
                if ws.send(msg).await.is_err() { break; }
            }
        })
        .await
}
```

## Authenticated upgrade

Authenticate on the route **before** upgrade (e.g. via a JWT hoop). The handler then reads claims from `Depot`:

```rust
#[handler]
async fn ws_auth(req: &mut Request, depot: &mut Depot, res: &mut Response)
    -> Result<(), StatusError>
{
    let user_id = depot.jwt_auth_data::<Claims>()
        .ok_or_else(StatusError::unauthorized)?
        .claims.user_id;
    WebSocketUpgrade::new()
        .upgrade(req, res, move |mut ws| async move {
            while let Some(Ok(msg)) = ws.recv().await {
                if ws.send(msg).await.is_err() { break; }
            }
            let _ = user_id;
        })
        .await
}
```

## Message helpers

`Message` supports `text`, `binary`, `ping`, `pong`, `close`, `close_with(code, reason)`. Pings are auto-ponged; handle `is_close()` explicitly.

```rust
while let Some(Ok(msg)) = ws.recv().await {
    if msg.is_text() {
        let text = msg.as_str().unwrap_or_default();
        ws.send(Message::text(format!("echo: {text}"))).await.ok();
    } else if msg.is_binary() {
        ws.send(Message::binary(msg.as_bytes().to_vec())).await.ok();
    } else if msg.is_close() {
        break;
    }
}
```

`as_str()` returns `Err` for non-text messages — use `is_text()` or match on the type first.

## Broadcasting pattern (channel + map of senders)

Use an mpsc channel per client. The receiver is forwarded into the socket sink, so broadcasters never hold a lock while sending:

```rust
use std::collections::HashMap;
use std::sync::LazyLock;
use std::sync::atomic::{AtomicUsize, Ordering};
use futures_util::{FutureExt, StreamExt};
use salvo::prelude::*;
use salvo::websocket::{Message, WebSocket, WebSocketUpgrade};
use tokio::sync::{RwLock, mpsc};
use tokio_stream::wrappers::UnboundedReceiverStream;

type Tx = mpsc::UnboundedSender<Result<Message, salvo::Error>>;
static NEXT_ID: AtomicUsize = AtomicUsize::new(1);
static USERS: LazyLock<RwLock<HashMap<usize, Tx>>> = LazyLock::new(Default::default);

#[handler]
async fn chat(req: &mut Request, res: &mut Response) -> Result<(), StatusError> {
    WebSocketUpgrade::new().upgrade(req, res, handle_socket).await
}

async fn handle_socket(ws: WebSocket) {
    let my_id = NEXT_ID.fetch_add(1, Ordering::Relaxed);
    let (ws_tx, mut ws_rx) = ws.split();

    let (tx, rx) = mpsc::unbounded_channel();
    tokio::spawn(UnboundedReceiverStream::new(rx).forward(ws_tx).map(|_| ()));

    USERS.write().await.insert(my_id, tx);

    while let Some(Ok(msg)) = ws_rx.next().await {
        if let Ok(text) = msg.as_str() {
            let out = format!("<User#{my_id}>: {text}");
            for (&uid, peer_tx) in USERS.read().await.iter() {
                if uid != my_id {
                    let _ = peer_tx.send(Ok(Message::text(out.clone())));
                }
            }
        }
    }

    USERS.write().await.remove(&my_id);
}
```

Gotchas:
- `ws.split()` comes from `StreamExt::split`; items flowing into the sink must be `Result<Message, salvo::Error>`, so the channel element type matches.
- Don't hold the `RwLock` write guard across `.await` on the send side — keep the read guard short-lived.

## Rooms

Wrap a `HashMap<String, HashMap<usize, Tx>>` in `Arc<RwLock<...>>` and share via `Depot` or a `LazyLock`. The insert/remove/broadcast pattern mirrors the flat map above.

## Heartbeat via `tokio::select!`

Client pings are auto-ponged; use server-side pings only if you need to detect silent peers.

```rust
use std::time::Duration;
use futures_util::{SinkExt, StreamExt};
use tokio::time::interval;

async fn with_heartbeat(ws: salvo::websocket::WebSocket) {
    use salvo::websocket::Message;
    let (mut tx, mut rx) = ws.split();
    let hb = async move {
        let mut ticker = interval(Duration::from_secs(30));
        loop {
            ticker.tick().await;
            if tx.send(Message::ping(Vec::new())).await.is_err() { break; }
        }
    };
    let recv = async move {
        while let Some(Ok(msg)) = rx.next().await {
            if msg.is_close() { break; }
        }
    };
    tokio::select! { _ = hb => {}, _ = recv => {} }
}
```

## JSON messages

```rust
#[derive(serde::Serialize, serde::Deserialize)]
#[serde(tag = "type")]
enum WsMsg {
    Chat { content: String },
    Join { room: String },
}

// Decode:
if let Ok(text) = msg.as_str() {
    if let Ok(parsed) = serde_json::from_str::<WsMsg>(text) { /* ... */ }
}
// Encode:
let json = serde_json::to_string(&WsMsg::Chat { content: "hi".into() }).unwrap();
ws.send(salvo::websocket::Message::text(json)).await.ok();
```

## Gotchas

- `recv()` returns `None` once the stream ends — always exit the loop.
- `upgrade()` spawns the callback as a separate task; anything captured must be `Send + 'static`.
- `Sec-WebSocket-Protocol` is echoed to clients — never put secrets in it.
- If the client omits `Sec-WebSocket-Version: 13` or the `Upgrade`/`Connection` headers, `upgrade()` returns `StatusError::bad_request`.

## Related Skills

- **salvo-sse**: Server-Sent Events for unidirectional updates
- **salvo-realtime**: Overview of real-time communication options
- **salvo-auth**: Authenticate WebSocket connections
