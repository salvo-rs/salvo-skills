---
name: salvo-realtime
description: Implement real-time features using WebSocket and Server-Sent Events (SSE). Use for chat applications, live updates, and bidirectional communication.
---

# Salvo Real-time Communication

This skill helps implement real-time features using WebSocket and SSE in Salvo applications.

## WebSocket

### Basic WebSocket Handler

```rust
use salvo::prelude::*;
use salvo::websocket::{Message, WebSocket};
use futures_util::{SinkExt, StreamExt};

#[handler]
async fn websocket_handler(req: &mut Request, res: &mut Response) -> Result<(), StatusError> {
    WebSocket::new()
        .handle(req, res, |mut ws| async move {
            while let Some(msg) = ws.next().await {
                match msg {
                    Ok(Message::Text(text)) => {
                        println!("Received: {}", text);
                        ws.send(Message::Text(format!("Echo: {}", text))).await.ok();
                    }
                    Ok(Message::Close(_)) => {
                        println!("Connection closed");
                        break;
                    }
                    _ => {}
                }
            }
        })
        .await
}
```

### WebSocket Router Setup

```rust
#[tokio::main]
async fn main() {
    let router = Router::new()
        .push(Router::with_path("ws").get(websocket_handler));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

### WebSocket Chat Application

```rust
use std::sync::Arc;
use tokio::sync::broadcast;

type Tx = broadcast::Sender<String>;

#[handler]
async fn chat_handler(req: &mut Request, res: &mut Response, depot: &mut Depot) -> Result<(), StatusError> {
    let tx = depot.obtain::<Arc<Tx>>().unwrap().clone();
    let mut rx = tx.subscribe();

    WebSocket::new()
        .handle(req, res, move |mut ws| async move {
            let (mut ws_tx, mut ws_rx) = ws.split();

            // Spawn task to receive broadcasts and send to WebSocket
            let mut send_task = tokio::spawn(async move {
                while let Ok(msg) = rx.recv().await {
                    if ws_tx.send(Message::Text(msg)).await.is_err() {
                        break;
                    }
                }
            });

            // Receive from WebSocket and broadcast
            let tx_clone = tx.clone();
            let mut recv_task = tokio::spawn(async move {
                while let Some(Ok(Message::Text(text))) = ws_rx.next().await {
                    tx_clone.send(text).ok();
                }
            });

            tokio::select! {
                _ = &mut send_task => recv_task.abort(),
                _ = &mut recv_task => send_task.abort(),
            }
        })
        .await
}

#[tokio::main]
async fn main() {
    let (tx, _) = broadcast::channel::<String>(100);
    let tx = Arc::new(tx);

    let router = Router::new()
        .hoop(move |depot: &mut Depot| {
            depot.inject(tx.clone());
        })
        .push(Router::with_path("chat").get(chat_handler));

    let acceptor = TcpListener::new("0.0.0.0:8080").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

### WebSocket with Authentication

```rust
#[handler]
async fn authenticated_ws(req: &mut Request, res: &mut Response, depot: &mut Depot) -> Result<(), StatusError> {
    // Check authentication
    let user_id = depot.get::<i64>("user_id")
        .ok_or_else(|| StatusError::unauthorized())?;

    WebSocket::new()
        .handle(req, res, move |mut ws| async move {
            ws.send(Message::Text(format!("Welcome, user {}!", user_id))).await.ok();

            while let Some(msg) = ws.next().await {
                if let Ok(Message::Text(text)) = msg {
                    ws.send(Message::Text(text)).await.ok();
                }
            }
        })
        .await
}
```

## Server-Sent Events (SSE)

### Basic SSE Handler

```rust
use salvo::sse::{SseEvent, SseKeepAlive};
use tokio_stream::StreamExt as _;
use std::time::Duration;

#[handler]
async fn sse_handler(res: &mut Response) {
    let stream = tokio_stream::iter(0..10)
        .throttle(Duration::from_secs(1))
        .map(|n| {
            SseEvent::default()
                .data(format!("Message {}", n))
                .id(n.to_string())
        });

    SseKeepAlive::new(stream).streaming(res);
}
```

### SSE with Custom Events

```rust
#[handler]
async fn sse_events(res: &mut Response) {
    let stream = tokio_stream::iter(vec![
        SseEvent::default()
            .event("user_joined")
            .data(r#"{"user": "Alice"}"#),
        SseEvent::default()
            .event("message")
            .data(r#"{"text": "Hello!"}"#),
        SseEvent::default()
            .event("user_left")
            .data(r#"{"user": "Bob"}"#),
    ]);

    SseKeepAlive::new(stream).streaming(res);
}
```

### SSE Chat Application

```rust
use tokio::sync::broadcast;

#[handler]
async fn sse_chat(depot: &mut Depot, res: &mut Response) {
    let tx = depot.obtain::<Arc<Tx>>().unwrap();
    let mut rx = tx.subscribe();

    let stream = async_stream::stream! {
        while let Ok(msg) = rx.recv().await {
            yield SseEvent::default().data(msg);
        }
    };

    SseKeepAlive::new(stream).streaming(res);
}

#[handler]
async fn send_message(body: JsonBody<Message>, depot: &mut Depot) -> StatusCode {
    let tx = depot.obtain::<Arc<Tx>>().unwrap();
    let msg = body.into_inner();

    tx.send(serde_json::to_string(&msg).unwrap()).ok();

    StatusCode::OK
}
```

### SSE with Heartbeat

```rust
use tokio::time::{interval, Duration};

#[handler]
async fn sse_with_heartbeat(res: &mut Response) {
    let mut ticker = interval(Duration::from_secs(30));

    let stream = async_stream::stream! {
        loop {
            ticker.tick().await;
            yield SseEvent::default()
                .event("heartbeat")
                .data("ping");
        }
    };

    SseKeepAlive::new(stream).streaming(res);
}
```

## Real-time Notifications

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

type Subscribers = Arc<RwLock<Vec<broadcast::Sender<String>>>>;

#[handler]
async fn subscribe_notifications(depot: &mut Depot, res: &mut Response) {
    let (tx, mut rx) = broadcast::channel::<String>(100);

    let subscribers = depot.obtain::<Subscribers>().unwrap();
    subscribers.write().await.push(tx);

    let stream = async_stream::stream! {
        while let Ok(notification) = rx.recv().await {
            yield SseEvent::default()
                .event("notification")
                .data(notification);
        }
    };

    SseKeepAlive::new(stream).streaming(res);
}

#[handler]
async fn send_notification(body: JsonBody<Notification>, depot: &mut Depot) -> StatusCode {
    let subscribers = depot.obtain::<Subscribers>().unwrap();
    let notification = serde_json::to_string(&body.into_inner()).unwrap();

    for tx in subscribers.read().await.iter() {
        tx.send(notification.clone()).ok();
    }

    StatusCode::OK
}
```

## Best Practices

### WebSocket
1. Handle connection errors gracefully
2. Implement ping/pong for connection health
3. Use message queues for high-traffic scenarios
4. Implement authentication before upgrade
5. Clean up resources on disconnect
6. Use binary messages for efficiency when appropriate
7. Implement reconnection logic on client side

### SSE
1. Set appropriate Content-Type header
2. Implement heartbeat to keep connection alive
3. Use event types for different message categories
4. Include event IDs for client-side replay
5. Handle client disconnections
6. Consider using Redis for multi-server scenarios
7. Implement proper error handling

### General
1. Use connection pooling for scalability
2. Implement rate limiting
3. Monitor active connections
4. Use compression for large messages
5. Implement proper logging
6. Test with multiple concurrent connections
