# How to Set Up a Rust Development Environment with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Rust, Development Environment, Docker, cargo-watch, Debugging

Description: Learn how to set up a Rust development environment with automatic recompilation using cargo-watch in a Docker container managed by Portainer.

---

Rust development in Docker via Portainer isolates your Rust toolchain and ensures consistent compiler versions. Caching the target and cargo registry directories is critical for acceptable compile times.

## Dev Environment Compose Stack

```yaml
version: "3.8"

services:
  rust-dev:
    image: rust:1.77-alpine
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      RUST_LOG: debug
      RUST_BACKTRACE: "1"
    volumes:
      - ./app:/app
      # Cache compiled dependencies - critical for fast rebuilds
      - cargo_registry:/usr/local/cargo/registry
      - cargo_git:/usr/local/cargo/git
      - target_cache:/app/target
    working_dir: /app
    command: >
      sh -c "
        apk add --no-cache musl-dev pkgconfig &&
        cargo install cargo-watch &&
        cargo watch -x run
      "

volumes:
  cargo_registry:
  cargo_git:
  target_cache:
```

## Simple Axum Web Server

```rust
// app/src/main.rs
use axum::{routing::get, Router, Json};
use serde_json::{json, Value};

#[tokio::main]
async fn main() {
    // Initialize logging
    tracing_subscriber::fmt::init();

    let app = Router::new()
        .route("/", get(root))
        .route("/health", get(health));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080")
        .await
        .unwrap();

    tracing::info!("Server listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn root() -> &'static str {
    // Edit and save — cargo watch recompiles and restarts
    "Rust development environment ready"
}

async fn health() -> Json<Value> {
    Json(json!({ "status": "ok" }))
}
```

## Cargo.toml

```toml
[package]
name = "rust-dev"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = "0.3"
```

## Reducing Build Times

First-time compilation in Docker takes several minutes. Subsequent builds with cached volumes are fast. Key optimizations:

```toml
# Cargo.toml: use faster linker in dev
[profile.dev]
opt-level = 0    # No optimization for dev builds (faster compile)
debug = true     # Include debug symbols
```

Use the `sccache` distributed compiler cache for even faster builds across container restarts.
