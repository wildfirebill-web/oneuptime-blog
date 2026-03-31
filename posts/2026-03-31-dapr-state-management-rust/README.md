# How to Use Dapr State Management with Rust

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Rust, State Management, Redis, Persistence

Description: Learn how to save, retrieve, delete, and run transactions on state using the Dapr Rust SDK, with examples for optimistic concurrency and bulk operations.

---

## Introduction

Dapr State Management gives Rust applications a consistent, backend-agnostic API for persisting key-value data. The Rust SDK's async client provides typed state operations with support for ETags, concurrency modes, and bulk reads. This guide covers all essential state management patterns.

## Setup

```toml
# Cargo.toml
[dependencies]
dapr = "0.13"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

## Connecting to Dapr

```rust
use dapr::Client;
type DaprClient = Client<dapr::client::TonicClient>;

async fn make_client() -> Result<DaprClient, Box<dyn std::error::Error>> {
    let client = DaprClient::connect("https://127.0.0.1".to_string()).await?;
    Ok(client)
}
```

## Saving State

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct UserProfile {
    username: String,
    email: String,
    plan: String,
}

async fn save_profile(client: &mut DaprClient) -> Result<(), Box<dyn std::error::Error>> {
    let profile = UserProfile {
        username: "alice".to_string(),
        email: "alice@example.com".to_string(),
        plan: "pro".to_string(),
    };

    client.save_state("statestore", "user-alice", &profile).await?;
    println!("Profile saved for alice");
    Ok(())
}
```

## Getting State

```rust
async fn get_profile(client: &mut DaprClient) -> Result<(), Box<dyn std::error::Error>> {
    let profile: Option<UserProfile> = client
        .get_state("statestore", "user-alice", None)
        .await?;

    match profile {
        Some(p) => println!("User: {} ({})", p.username, p.plan),
        None    => println!("Profile not found"),
    }
    Ok(())
}
```

## Deleting State

```rust
async fn delete_profile(client: &mut DaprClient) -> Result<(), Box<dyn std::error::Error>> {
    client.delete_state("statestore", "user-alice", None).await?;
    println!("Profile deleted");
    Ok(())
}
```

## Bulk State Reads

```rust
async fn bulk_get(client: &mut DaprClient) -> Result<(), Box<dyn std::error::Error>> {
    let keys = vec![
        "user-alice".to_string(),
        "user-bob".to_string(),
        "user-carol".to_string(),
    ];

    let result = client.get_bulk_state("statestore", keys, None).await?;

    for item in result.items {
        if !item.data.is_empty() {
            println!("Key: {}", item.key);
        } else {
            println!("Key: {} - not found", item.key);
        }
    }
    Ok(())
}
```

## Transactional State Updates

```rust
use dapr::dapr::dapr::proto::runtime::v1 as dapr_v1;

async fn transactional_update(client: &mut DaprClient) -> Result<(), Box<dyn std::error::Error>> {
    let order = serde_json::json!({"status": "shipped"});
    let inventory = serde_json::json!({"stock": 95});

    let operations = vec![
        dapr_v1::TransactionalStateOperation {
            operationtype: "upsert".to_string(),
            request: Some(dapr_v1::StateItem {
                key: "order-001".to_string(),
                value: serde_json::to_vec(&order)?,
                ..Default::default()
            }),
        },
        dapr_v1::TransactionalStateOperation {
            operationtype: "upsert".to_string(),
            request: Some(dapr_v1::StateItem {
                key: "inventory-widget".to_string(),
                value: serde_json::to_vec(&inventory)?,
                ..Default::default()
            }),
        },
    ];

    client.execute_state_transaction("statestore", operations, None).await?;
    println!("Transaction committed");
    Ok(())
}
```

## Running with Dapr

```bash
dapr run \
  --app-id rust-state-demo \
  --dapr-grpc-port 50001 \
  -- cargo run
```

## Summary

The Dapr Rust SDK provides a fully async state management API covering single saves, bulk reads, deletes, and transactions. All operations use serde for serialization, making it easy to work with typed Rust structs. The state backend can be swapped from Redis to PostgreSQL or Cosmos DB by updating the component YAML without changing any Rust code.
