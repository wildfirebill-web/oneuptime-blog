# How to Implement Connection Pooling for MySQL in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection Pool, Go, database/sql, Performance

Description: Learn how to configure MySQL connection pooling in Go using the standard database/sql package with the go-sql-driver/mysql driver for efficient concurrent access.

---

## Introduction

Go's `database/sql` package includes built-in connection pooling. When you open a database handle with `sql.Open`, Go manages a pool of connections automatically. The pool configuration is critical for production MySQL applications - too few connections cause latency spikes, too many exhaust MySQL's connection limit.

## Installation

```bash
go get github.com/go-sql-driver/mysql
```

## Opening and Configuring the Pool

```go
package main

import (
    "database/sql"
    "log"
    "time"

    _ "github.com/go-sql-driver/mysql"
)

func NewDB() (*sql.DB, error) {
    dsn := "root:password@tcp(localhost:3306)/mydb?charset=utf8mb4&parseTime=True&loc=Local"
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }

    // Pool configuration
    db.SetMaxOpenConns(25)          // max simultaneous connections
    db.SetMaxIdleConns(10)          // connections kept idle in pool
    db.SetConnMaxLifetime(5 * time.Minute)   // max connection age
    db.SetConnMaxIdleTime(2 * time.Minute)   // evict idle connections after this

    // Verify connectivity
    if err := db.Ping(); err != nil {
        return nil, err
    }

    log.Println("MySQL connection pool initialized")
    return db, nil
}
```

## Performing Queries

```go
type Product struct {
    ID    int
    Name  string
    Price float64
    Stock int
}

func GetAffordableProducts(db *sql.DB, maxPrice float64) ([]Product, error) {
    rows, err := db.Query(
        "SELECT id, name, price, stock FROM products WHERE price <= ? AND stock > 0 ORDER BY price",
        maxPrice,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var products []Product
    for rows.Next() {
        var p Product
        if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock); err != nil {
            return nil, err
        }
        products = append(products, p)
    }
    return products, rows.Err()
}
```

## Using Prepared Statements

```go
func InsertProduct(db *sql.DB, name string, price float64, stock, categoryID int) (int64, error) {
    stmt, err := db.Prepare(
        "INSERT INTO products (name, price, stock, category_id) VALUES (?, ?, ?, ?)",
    )
    if err != nil {
        return 0, err
    }
    defer stmt.Close()

    result, err := stmt.Exec(name, price, stock, categoryID)
    if err != nil {
        return 0, err
    }
    return result.LastInsertId()
}
```

## Transactions

```go
func TransferStock(db *sql.DB, fromID, toID, qty int) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    if _, err := tx.Exec("UPDATE products SET stock = stock - ? WHERE id = ?", qty, fromID); err != nil {
        return err
    }
    if _, err := tx.Exec("UPDATE products SET stock = stock + ? WHERE id = ?", qty, toID); err != nil {
        return err
    }

    return tx.Commit()
}
```

## Monitoring Pool Statistics

```go
func PrintPoolStats(db *sql.DB) {
    stats := db.Stats()
    log.Printf(
        "Pool stats - Open: %d, InUse: %d, Idle: %d, WaitCount: %d, WaitDuration: %s",
        stats.OpenConnections,
        stats.InUse,
        stats.Idle,
        stats.WaitCount,
        stats.WaitDuration,
    )
}
```

Call `PrintPoolStats` periodically or expose it as a metrics endpoint to track connection utilization.

## Sizing the Pool

A practical formula:

```text
MaxOpenConns = (number_of_cpu_cores * 2) + number_of_disk_spindles
```

For a 4-core machine with SSD:

```go
db.SetMaxOpenConns(9)
db.SetMaxIdleConns(4)
```

## Summary

Go's `database/sql` provides production-ready connection pooling out of the box. Set `SetMaxOpenConns`, `SetMaxIdleConns`, and `SetConnMaxLifetime` based on your server's CPU cores and MySQL's `max_connections`. Monitor `db.Stats()` in production to detect connection pool exhaustion (high `WaitCount`) or inefficiency (low `Idle`). Always use `rows.Close()` and `stmt.Close()` with `defer` to return connections to the pool promptly.
