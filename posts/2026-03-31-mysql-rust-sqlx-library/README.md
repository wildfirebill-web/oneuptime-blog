# How to Use MySQL with Rust's sqlx Library

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Rust, SQLx, Async, Query

Description: Learn how to use the sqlx library for type-safe, compile-time verified MySQL queries in async Rust applications.

---

## Why sqlx?

`sqlx` is an async, compile-time checked SQL library for Rust. Its key advantage is that it verifies your SQL queries and maps result columns to Rust types at compile time using `query!` macros - catching schema mismatches before your code ships to production.

## Setup

```toml
[dependencies]
sqlx  = { version = "0.8", features = ["mysql", "runtime-tokio", "macros", "chrono", "uuid"] }
tokio = { version = "1", features = ["full"] }
```

Set the environment variable for compile-time checking:

```bash
export DATABASE_URL="mysql://app_user:secret@localhost/shop"
```

## Creating the Connection Pool

```rust
use sqlx::MySqlPool;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let pool = MySqlPool::connect(&std::env::var("DATABASE_URL").unwrap()).await?;
    run_queries(&pool).await?;
    Ok(())
}
```

For fine-grained pool configuration:

```rust
use sqlx::mysql::MySqlPoolOptions;
use std::time::Duration;

let pool = MySqlPoolOptions::new()
    .max_connections(20)
    .min_connections(2)
    .acquire_timeout(Duration::from_secs(5))
    .idle_timeout(Duration::from_secs(600))
    .connect(&database_url)
    .await?;
```

## Compile-Time Verified Queries

```rust
#[derive(Debug, sqlx::FromRow)]
struct User {
    id:    i32,
    email: String,
    name:  String,
}

// query_as! verifies SQL and maps columns at compile time
let users: Vec<User> = sqlx::query_as!(
    User,
    "SELECT id, email, name FROM users WHERE created_at > ?",
    chrono::Utc::now() - chrono::Duration::days(7)
)
.fetch_all(&pool)
.await?;
```

## Dynamic Queries with query() and query_as()

When you cannot use macros (e.g., dynamic column selection), use the non-macro variants:

```rust
let rows = sqlx::query("SELECT id, name FROM products WHERE category = ?")
    .bind("electronics")
    .fetch_all(&pool)
    .await?;

for row in &rows {
    use sqlx::Row;
    println!("{}: {}", row.get::<i32, _>("id"), row.get::<String, _>("name"));
}
```

## Transactions with sqlx

```rust
async fn transfer(pool: &MySqlPool, from: i32, to: i32, amount: f64)
    -> Result<(), sqlx::Error>
{
    let mut tx = pool.begin().await?;

    sqlx::query!("UPDATE accounts SET balance = balance - ? WHERE id = ?", amount, from)
        .execute(&mut *tx)
        .await?;

    sqlx::query!("UPDATE accounts SET balance = balance + ? WHERE id = ?", amount, to)
        .execute(&mut *tx)
        .await?;

    tx.commit().await?;
    Ok(())
}
```

## Running Migrations

```bash
cargo install sqlx-cli
sqlx migrate add create_users_table
sqlx migrate run
```

Migrations are plain SQL files in a `migrations/` directory.

## Handling NULL Values

```rust
#[derive(sqlx::FromRow)]
struct Order {
    id:          i32,
    shipped_at:  Option<chrono::DateTime<chrono::Utc>>,
}

let order = sqlx::query_as!(Order, "SELECT id, shipped_at FROM orders WHERE id = ?", 1)
    .fetch_optional(&pool)
    .await?;
```

## Summary

`sqlx` brings compile-time SQL safety to Rust, catching query errors and type mismatches before deployment. Use `query!` macros for static queries with maximum safety, `query()` for dynamic queries, and `MySqlPoolOptions` for configuring the connection pool. The built-in migration runner (`sqlx-cli`) rounds out a complete database workflow.
