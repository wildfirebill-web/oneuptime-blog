# How to Connect to MySQL from Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Go, Driver, database/sql, Connection

Description: Learn how to connect to a MySQL database from Go using the go-sql-driver/mysql package and the standard database/sql interface.

---

## Overview

Go uses the `database/sql` package as a standard interface for SQL databases. For MySQL, you need the `go-sql-driver/mysql` driver, which is the most widely used driver and fully implements the `database/sql` interface.

## Adding the Driver

```bash
go get -u github.com/go-sql-driver/mysql
```

## Opening a Connection Pool

`database/sql` manages a connection pool automatically. `sql.Open()` does not immediately connect - it just validates the DSN.

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"

    _ "github.com/go-sql-driver/mysql"
)

func main() {
    dsn := fmt.Sprintf("%s:%s@tcp(%s:3306)/%s?charset=utf8mb4&parseTime=True&loc=UTC",
        os.Getenv("DB_USER"),
        os.Getenv("DB_PASSWORD"),
        os.Getenv("DB_HOST"),
        os.Getenv("DB_NAME"),
    )

    db, err := sql.Open("mysql", dsn)
    if err != nil {
        log.Fatalf("open: %v", err)
    }
    defer db.Close()

    // Configure pool
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)

    // Verify connectivity
    if err := db.Ping(); err != nil {
        log.Fatalf("ping: %v", err)
    }
    fmt.Println("Connected to MySQL")
}
```

## Querying Rows

```go
rows, err := db.Query(
    "SELECT id, name, price FROM products WHERE price < ?", 50.00)
if err != nil {
    log.Fatal(err)
}
defer rows.Close()

for rows.Next() {
    var id   int
    var name string
    var price float64
    if err := rows.Scan(&id, &name, &price); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%d  %s  %.2f\n", id, name, price)
}

if err := rows.Err(); err != nil {
    log.Fatal(err)
}
```

## Querying a Single Row

```go
var email string
err := db.QueryRow(
    "SELECT email FROM users WHERE id = ?", 42,
).Scan(&email)
if err == sql.ErrNoRows {
    fmt.Println("user not found")
} else if err != nil {
    log.Fatal(err)
}
```

## Inserting Data

```go
result, err := db.Exec(
    "INSERT INTO orders (customer_id, total) VALUES (?, ?)", 42, 99.99)
if err != nil {
    log.Fatal(err)
}
id, _ := result.LastInsertId()
fmt.Println("Inserted order ID:", id)
```

## Using Transactions

```go
tx, err := db.Begin()
if err != nil {
    log.Fatal(err)
}

_, err = tx.Exec("UPDATE stock SET qty = qty - 1 WHERE id = ?", 7)
if err != nil {
    tx.Rollback()
    log.Fatal(err)
}

_, err = tx.Exec("INSERT INTO order_items (item_id) VALUES (?)", 7)
if err != nil {
    tx.Rollback()
    log.Fatal(err)
}

tx.Commit()
```

## Summary

Go MySQL connectivity is built on the `database/sql` package with the `go-sql-driver/mysql` driver. Always call `db.Ping()` after opening to verify connectivity, configure pool limits with `SetMaxOpenConns` and `SetMaxIdleConns`, and use `?` placeholders in queries to avoid SQL injection. Include `parseTime=True` in the DSN to automatically scan MySQL `DATE` and `DATETIME` values into Go `time.Time`.
