# How to Use IPv6 with Rust Axum

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rust, IPv6, Axum, HTTP, Web Framework, Tokio

Description: Build IPv6-ready web services with Rust's Axum framework including binding, client IP extraction, extractors, and middleware.

## Binding Axum to IPv6

```toml
# Cargo.toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
```

```rust
use axum::{routing::get, Router};
use std::net::SocketAddr;

async fn hello() -> &'static str {
    "Hello from IPv6 Axum!"
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(hello));

    // Listen on all IPv6 interfaces (dual-stack on Linux)
    let listener = tokio::net::TcpListener::bind("[::]:3000").await.unwrap();
    println!("Listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await.unwrap();
}
```

## Extracting Client IP with ConnectInfo

Axum provides `ConnectInfo` to access the peer's `SocketAddr`:

```rust
use axum::{
    extract::ConnectInfo,
    routing::get,
    Router,
};
use std::net::{IpAddr, SocketAddr};

async fn client_ip(ConnectInfo(addr): ConnectInfo<SocketAddr>) -> String {
    let real_ip = match addr.ip() {
        IpAddr::V6(v6) => {
            // Unwrap IPv4-mapped addresses (::ffff:x.x.x.x) from dual-stack
            v6.to_ipv4().map(IpAddr::V4).unwrap_or(IpAddr::V6(v6))
        }
        v4 => v4,
    };

    format!("Client IP: {}", real_ip)
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/ip", get(client_ip));

    let listener = tokio::net::TcpListener::bind("[::]:3000").await.unwrap();

    // Must use serve_with_incoming_make_service or into_make_service_with_connect_info
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await
    .unwrap();
}
```

## Custom Extractor for IPv6 Validation

```rust
use axum::{
    async_trait,
    extract::{FromRequestParts, Query},
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
};
use serde::Deserialize;
use std::net::Ipv6Addr;

#[derive(Deserialize)]
struct IPv6Query {
    addr: String,
}

pub struct ValidIPv6(Ipv6Addr);

#[async_trait]
impl<S> FromRequestParts<S> for ValidIPv6
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let Query(q) = Query::<IPv6Query>::from_request_parts(parts, state)
            .await
            .map_err(|e| e.into_response())?;

        q.addr
            .parse::<Ipv6Addr>()
            .map(ValidIPv6)
            .map_err(|_| {
                (StatusCode::BAD_REQUEST, "Invalid IPv6 address").into_response()
            })
    }
}

async fn process_addr(ValidIPv6(addr): ValidIPv6) -> String {
    format!("Processing: {} (loopback={})", addr, addr.is_loopback())
}
```

## Tower Middleware for IPv6 Logging

```rust
use axum::{routing::get, Router};
use std::net::SocketAddr;
use tower_http::trace::{DefaultMakeSpan, TraceLayer};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

async fn root() -> &'static str {
    "IPv6 Axum with tracing"
}

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new()
        .route("/", get(root))
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(DefaultMakeSpan::default().include_headers(true)),
        );

    let listener = tokio::net::TcpListener::bind("[::]:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## State Sharing with IPv6 Allow List

```rust
use axum::{
    extract::{ConnectInfo, State},
    http::StatusCode,
    response::IntoResponse,
    routing::get,
    Router,
};
use std::{net::{IpAddr, Ipv6Addr, SocketAddr}, sync::Arc};

#[derive(Clone)]
struct AppState {
    allowed: Arc<Vec<Ipv6Addr>>,
}

async fn protected(
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
    State(state): State<AppState>,
) -> impl IntoResponse {
    let client_v6 = match addr.ip() {
        IpAddr::V6(v6) => v6,
        IpAddr::V4(v4) => v4.to_ipv6_mapped(),
    };

    if state.allowed.contains(&client_v6) {
        (StatusCode::OK, "Access granted")
    } else {
        (StatusCode::FORBIDDEN, "Access denied")
    }
}

#[tokio::main]
async fn main() {
    let state = AppState {
        allowed: Arc::new(vec![
            "::1".parse().unwrap(),
            "2001:db8::100".parse().unwrap(),
        ]),
    };

    let app = Router::new()
        .route("/protected", get(protected))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("[::]:3000").await.unwrap();
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await
    .unwrap();
}
```

## Conclusion

Axum supports IPv6 through Tokio's `TcpListener::bind("[::]:port")`. The `ConnectInfo<SocketAddr>` extractor provides the client's socket address including IPv6 addresses. Custom extractors encode validation logic — including IPv6 checks — into the type system. `into_make_service_with_connect_info` is required to enable `ConnectInfo` extraction. Tower middleware via `TraceLayer` provides structured logging that includes IPv6 peer addresses automatically.
