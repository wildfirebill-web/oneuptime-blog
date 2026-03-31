# How to Use Redis with Axum in Rust

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Axum, Rust

Description: Learn how to integrate Redis with Axum in Rust for async caching and counters using deadpool-redis, with extractors and shared application state.

---

Axum is a modern async web framework for Rust built on Tokio and Tower. Integrating Redis with Axum using `deadpool-redis` provides a non-blocking connection pool that fits naturally with Axum's extractor-based handler model.

## Add Dependencies

```toml
# Cargo.toml
[dependencies]
axum = "0.7"
deadpool-redis = { version = "0.15", features = ["rt_tokio_1"] }
redis = { version = "0.25", features = ["tokio-comp", "aio"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1", features = ["full"] }
tower = "0.4"
```

## Set Up App State

```rust
// src/main.rs
use axum::{routing::{get, post, delete}, Router};
use deadpool_redis::{Config, Pool, Runtime};
use std::sync::Arc;

#[derive(Clone)]
pub struct AppState {
    pub redis: Pool,
}

#[tokio::main]
async fn main() {
    let redis_cfg = Config::from_url("redis://127.0.0.1:6379/");
    let redis_pool = redis_cfg
        .create_pool(Some(Runtime::Tokio1))
        .expect("Failed to create Redis pool");

    let state = Arc::new(AppState { redis: redis_pool });

    let app = Router::new()
        .route("/products/:id", get(get_product))
        .route("/products/:id/cache", delete(evict_product))
        .route("/counter/:name/increment", post(increment_counter))
        .route("/counter/:name", get(get_counter))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

## Cache Handler with Axum Extractors

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::{IntoResponse, Json},
};
use redis::AsyncCommands;
use serde::{Deserialize, Serialize};
use std::sync::Arc;

#[derive(Serialize, Deserialize)]
pub struct Product {
    pub id: String,
    pub name: String,
}

pub async fn get_product(
    Path(id): Path<String>,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    let cache_key = format!("product:{}", id);

    let mut conn = match state.redis.get().await {
        Ok(c) => c,
        Err(_) => return (StatusCode::SERVICE_UNAVAILABLE, "Pool error").into_response(),
    };

    // Try cache first
    if let Ok(cached) = conn.get::<_, String>(&cache_key).await {
        if let Ok(product) = serde_json::from_str::<Product>(&cached) {
            return Json(product).into_response();
        }
    }

    // Simulate DB fetch
    let product = Product {
        id: id.clone(),
        name: format!("Widget {}", id),
    };

    let serialized = serde_json::to_string(&product).unwrap();
    let _: () = conn.set_ex(&cache_key, &serialized, 60u64).await.unwrap_or(());

    Json(product).into_response()
}

pub async fn evict_product(
    Path(id): Path<String>,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    let mut conn = state.redis.get().await.unwrap();
    let deleted: i64 = conn.del(format!("product:{}", id)).await.unwrap_or(0);
    format!("Deleted {} key(s)", deleted)
}
```

## Atomic Counter Endpoints

```rust
pub async fn increment_counter(
    Path(name): Path<String>,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    let mut conn = state.redis.get().await.unwrap();
    let count: i64 = conn
        .incr(format!("counter:{}", name), 1)
        .await
        .unwrap_or(0);
    count.to_string()
}

pub async fn get_counter(
    Path(name): Path<String>,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    let mut conn = state.redis.get().await.unwrap();
    let count: Option<i64> = conn
        .get(format!("counter:{}", name))
        .await
        .unwrap_or(None);
    count.unwrap_or(0).to_string()
}
```

## Tower Middleware for Rate Limiting

```rust
use axum::middleware::{self, Next};
use axum::http::Request;
use axum::body::Body;

pub async fn rate_limit_middleware(
    State(state): State<Arc<AppState>>,
    req: Request<Body>,
    next: Next,
) -> impl IntoResponse {
    let ip = req
        .headers()
        .get("x-forwarded-for")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string();

    if let Ok(mut conn) = state.redis.get().await {
        let key = format!("rate:{}", ip);
        let count: i64 = conn.incr(&key, 1).await.unwrap_or(1);
        if count == 1 {
            let _: () = conn.expire(&key, 60).await.unwrap_or(());
        }
        if count > 100 {
            return (StatusCode::TOO_MANY_REQUESTS, "Rate limit exceeded").into_response();
        }
    }

    next.run(req).await
}
```

## Summary

Axum integrates with Redis via `deadpool-redis` and `AsyncCommands` using Axum's `State` extractor to inject the connection pool into handlers. The `Arc<AppState>` wrapper allows the pool to be cloned cheaply across handler threads. This pattern supports high-concurrency async workloads with full type safety, making it ideal for production caching and rate limiting in Axum-based APIs.
