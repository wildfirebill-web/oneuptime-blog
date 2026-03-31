# How to Use MySQL with Go's database/sql Package

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Go, Golang, database/sql, Database Integration

Description: Learn how to connect to MySQL from Go using the database/sql package and the go-sql-driver/mysql driver with practical CRUD and transaction examples.

---

## What is database/sql

Go's `database/sql` package is the standard library interface for SQL databases. It provides connection pooling, prepared statements, transactions, and query scanning. For MySQL, you need the `go-sql-driver/mysql` driver.

## Installing the Driver

```bash
go get github.com/go-sql-driver/mysql
```

## Connecting to MySQL

```go
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/go-sql-driver/mysql"
)

func main() {
    dsn := "user:password@tcp(localhost:3306)/your_database?parseTime=true&charset=utf8mb4"
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    if err := db.Ping(); err != nil {
        log.Fatal("Cannot connect:", err)
    }
    fmt.Println("Connected to MySQL")
}
```

## Configuring the Connection Pool

```go
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(3 * time.Minute)
```

## Creating a Table

```go
_, err := db.Exec(`
    CREATE TABLE IF NOT EXISTS employees (
        id   INT AUTO_INCREMENT PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        dept VARCHAR(100),
        salary DECIMAL(10, 2)
    )
`)
if err != nil {
    log.Fatal(err)
}
```

## Inserting Data

```go
result, err := db.Exec(
    "INSERT INTO employees (name, dept, salary) VALUES (?, ?, ?)",
    "Alice Smith", "Engineering", 95000.00,
)
if err != nil {
    log.Fatal(err)
}

id, err := result.LastInsertId()
if err != nil {
    log.Fatal(err)
}
fmt.Println("Inserted ID:", id)
```

## Querying Multiple Rows

```go
rows, err := db.Query(
    "SELECT id, name, dept, salary FROM employees WHERE dept = ?",
    "Engineering",
)
if err != nil {
    log.Fatal(err)
}
defer rows.Close()

for rows.Next() {
    var id int
    var name, dept string
    var salary float64

    if err := rows.Scan(&id, &name, &dept, &salary); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("ID: %d, Name: %s, Salary: %.2f\n", id, name, salary)
}
if err := rows.Err(); err != nil {
    log.Fatal(err)
}
```

## Querying a Single Row

```go
var name string
var salary float64

err := db.QueryRow(
    "SELECT name, salary FROM employees WHERE id = ?", 1,
).Scan(&name, &salary)

if err == sql.ErrNoRows {
    fmt.Println("Not found")
} else if err != nil {
    log.Fatal(err)
} else {
    fmt.Printf("Name: %s, Salary: %.2f\n", name, salary)
}
```

## Updating Records

```go
result, err := db.Exec(
    "UPDATE employees SET salary = ? WHERE id = ?",
    105000.00, 1,
)
if err != nil {
    log.Fatal(err)
}

rows, _ := result.RowsAffected()
fmt.Println("Rows updated:", rows)
```

## Deleting Records

```go
result, err := db.Exec("DELETE FROM employees WHERE id = ?", 5)
if err != nil {
    log.Fatal(err)
}
rows, _ := result.RowsAffected()
fmt.Println("Rows deleted:", rows)
```

## Using Transactions

```go
tx, err := db.Begin()
if err != nil {
    log.Fatal(err)
}

_, err = tx.Exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", 500.00, 1)
if err != nil {
    tx.Rollback()
    log.Fatal(err)
}

_, err = tx.Exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", 500.00, 2)
if err != nil {
    tx.Rollback()
    log.Fatal(err)
}

if err := tx.Commit(); err != nil {
    log.Fatal(err)
}
fmt.Println("Transfer successful")
```

## Using Prepared Statements

```go
stmt, err := db.Prepare("SELECT * FROM employees WHERE dept = ?")
if err != nil {
    log.Fatal(err)
}
defer stmt.Close()

rows, err := stmt.Query("Engineering")
// process rows...
```

## Summary

Go's `database/sql` with `go-sql-driver/mysql` provides a robust MySQL client with built-in connection pooling and prepared statements. Use `?` placeholders in all queries, always `defer rows.Close()` and check `rows.Err()` after iteration, configure pool limits with `SetMaxOpenConns()`, and use `tx.Rollback()` in error paths within transactions.
