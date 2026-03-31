# How to Build a REST API with MySQL and Go Gin

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Go, Gin, REST API, Backend

Description: Build a fast and efficient REST API using Go, Gin, and MySQL with connection pooling, prepared statements, and structured error handling.

---

## Project Setup

Go with the Gin framework and the `go-sql-driver/mysql` package provides a high-performance stack for MySQL-backed REST APIs. Go's built-in `database/sql` package includes connection pooling, making it straightforward to build production-ready APIs.

```bash
mkdir go-mysql-api && cd go-mysql-api
go mod init github.com/yourorg/go-mysql-api
go get github.com/gin-gonic/gin
go get github.com/go-sql-driver/mysql
go get github.com/joho/godotenv
```

## Database Connection

```go
// internal/database/db.go
package database

import (
    "database/sql"
    "fmt"
    "os"
    "time"

    _ "github.com/go-sql-driver/mysql"
)

func New() (*sql.DB, error) {
    dsn := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?parseTime=true&loc=UTC",
        os.Getenv("DB_USER"),
        os.Getenv("DB_PASSWORD"),
        os.Getenv("DB_HOST"),
        os.Getenv("DB_PORT"),
        os.Getenv("DB_NAME"),
    )

    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }

    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)
    db.SetConnMaxIdleTime(10 * time.Minute)

    if err := db.Ping(); err != nil {
        return nil, err
    }
    return db, nil
}
```

## Order Model and Repository

```go
// internal/orders/model.go
package orders

import "time"

type Order struct {
    ID        int64     `json:"id"`
    UserID    int64     `json:"user_id"`
    Total     float64   `json:"total"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"created_at"`
}

type CreateOrderRequest struct {
    UserID int64   `json:"user_id" binding:"required"`
    Total  float64 `json:"total"   binding:"required,gt=0"`
}
```

```go
// internal/orders/repository.go
package orders

import (
    "context"
    "database/sql"
)

type Repository struct {
    db *sql.DB
}

func NewRepository(db *sql.DB) *Repository {
    return &Repository{db: db}
}

func (r *Repository) List(ctx context.Context) ([]Order, error) {
    rows, err := r.db.QueryContext(ctx,
        `SELECT id, user_id, total, status, created_at
         FROM orders ORDER BY created_at DESC LIMIT 50`)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var orders []Order
    for rows.Next() {
        var o Order
        if err := rows.Scan(&o.ID, &o.UserID, &o.Total, &o.Status, &o.CreatedAt); err != nil {
            return nil, err
        }
        orders = append(orders, o)
    }
    return orders, rows.Err()
}

func (r *Repository) GetByID(ctx context.Context, id int64) (*Order, error) {
    var o Order
    err := r.db.QueryRowContext(ctx,
        `SELECT id, user_id, total, status, created_at FROM orders WHERE id = ?`, id,
    ).Scan(&o.ID, &o.UserID, &o.Total, &o.Status, &o.CreatedAt)
    if err == sql.ErrNoRows {
        return nil, nil
    }
    return &o, err
}

func (r *Repository) Create(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    result, err := r.db.ExecContext(ctx,
        `INSERT INTO orders (user_id, total, status) VALUES (?, ?, 'pending')`,
        req.UserID, req.Total,
    )
    if err != nil {
        return nil, err
    }
    id, _ := result.LastInsertId()
    return r.GetByID(ctx, id)
}
```

## Gin Handler

```go
// internal/orders/handler.go
package orders

import (
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
)

type Handler struct {
    repo *Repository
}

func NewHandler(repo *Repository) *Handler {
    return &Handler{repo: repo}
}

func (h *Handler) Register(r *gin.RouterGroup) {
    r.GET("", h.list)
    r.GET("/:id", h.get)
    r.POST("", h.create)
}

func (h *Handler) list(c *gin.Context) {
    orders, err := h.repo.List(c.Request.Context())
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"data": orders, "count": len(orders)})
}

func (h *Handler) get(c *gin.Context) {
    id, _ := strconv.ParseInt(c.Param("id"), 10, 64)
    order, err := h.repo.GetByID(c.Request.Context(), id)
    if err != nil || order == nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "order not found"})
        return
    }
    c.JSON(http.StatusOK, order)
}

func (h *Handler) create(c *gin.Context) {
    var req CreateOrderRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    order, err := h.repo.Create(c.Request.Context(), req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }
    c.JSON(http.StatusCreated, order)
}
```

## Summary

A Go Gin MySQL REST API uses `database/sql`'s built-in connection pool with `SetMaxOpenConns` and `SetConnMaxLifetime` for reliable connection management. The repository pattern separates database logic from HTTP handlers, and Gin's `ShouldBindJSON` provides automatic request validation. Go's context propagation ensures queries respect request timeouts throughout the call chain.
