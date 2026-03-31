# How to Use ClickHouse with Go (Fiber and Gin)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Go, Fiber, Gin, Database, Analytics

Description: Build analytics APIs in Go using ClickHouse with the official clickhouse-go driver, covering connection setup, parameterized queries, batch inserts for Fiber and Gin.

---

Go's performance and concurrency model pairs naturally with ClickHouse. The official `clickhouse-go` driver provides a native protocol client with full support for batch inserts, async inserts, parameterized queries, and context cancellation. This guide covers both the Fiber and Gin frameworks with the same underlying driver.

## Installation

```bash
go mod init analytics
go get github.com/ClickHouse/clickhouse-go/v2
go get github.com/gofiber/fiber/v2          # for Fiber
go get github.com/gin-gonic/gin             # for Gin
```

## ClickHouse Schema

```sql
CREATE DATABASE IF NOT EXISTS analytics;

CREATE TABLE analytics.events
(
    event_id   UUID        DEFAULT generateUUIDv4(),
    user_id    UInt64,
    session_id String,
    event_type LowCardinality(String),
    page       String,
    properties String      DEFAULT '{}',
    ts         DateTime64(3)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (event_type, user_id, ts);
```

## ClickHouse Connection

```go
// internal/db/clickhouse.go
package db

import (
    "context"
    "crypto/tls"
    "fmt"
    "time"

    "github.com/ClickHouse/clickhouse-go/v2"
    "github.com/ClickHouse/clickhouse-go/v2/lib/driver"
)

var DB driver.Conn

func Connect(host string, port int, database, user, password string, secure bool) error {
    opts := &clickhouse.Options{
        Addr: []string{fmt.Sprintf("%s:%d", host, port)},
        Auth: clickhouse.Auth{
            Database: database,
            Username: user,
            Password: password,
        },
        Debug:            false,
        DialTimeout:      10 * time.Second,
        MaxOpenConns:     20,
        MaxIdleConns:     5,
        ConnMaxLifetime:  10 * time.Minute,
        ConnOpenStrategy: clickhouse.ConnOpenInOrder,
        Settings: clickhouse.Settings{
            "max_execution_time":    30,
            "max_memory_usage":      4_000_000_000,
        },
    }

    if secure {
        opts.TLS = &tls.Config{InsecureSkipVerify: false}
    }

    conn, err := clickhouse.Open(opts)
    if err != nil {
        return fmt.Errorf("clickhouse.Open: %w", err)
    }

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := conn.Ping(ctx); err != nil {
        return fmt.Errorf("clickhouse ping: %w", err)
    }

    DB = conn
    return nil
}
```

## Domain Types

```go
// internal/model/event.go
package model

import "time"

type Event struct {
    EventID   string    `json:"event_id"`
    UserID    uint64    `json:"user_id"`
    SessionID string    `json:"session_id"`
    EventType string    `json:"event_type"`
    Page      string    `json:"page"`
    Properties string   `json:"properties"`
    Ts        time.Time `json:"ts"`
}

type EventInput struct {
    UserID     uint64                 `json:"user_id"`
    SessionID  string                 `json:"session_id"`
    EventType  string                 `json:"event_type"`
    Page       string                 `json:"page"`
    Properties map[string]interface{} `json:"properties"`
}

type AggregateSummary struct {
    EventType   string `json:"event_type"`
    Total       uint64 `json:"total"`
    UniqueUsers uint64 `json:"unique_users"`
}
```

## Repository

```go
// internal/repository/event.go
package repository

import (
    "context"
    "encoding/json"
    "time"

    "github.com/ClickHouse/clickhouse-go/v2/lib/driver"
    "analytics/internal/model"
)

type EventRepository struct {
    db driver.Conn
}

func NewEventRepository(db driver.Conn) *EventRepository {
    return &EventRepository{db: db}
}

func (r *EventRepository) Insert(ctx context.Context, e model.EventInput) error {
    props, _ := json.Marshal(e.Properties)
    return r.db.Exec(ctx,
        `INSERT INTO analytics.events (user_id, session_id, event_type, page, properties, ts)
         VALUES (?, ?, ?, ?, ?, ?)`,
        e.UserID, e.SessionID, e.EventType, e.Page, string(props), time.Now(),
    )
}

func (r *EventRepository) InsertBatch(ctx context.Context, events []model.EventInput) error {
    batch, err := r.db.PrepareBatch(ctx,
        `INSERT INTO analytics.events (user_id, session_id, event_type, page, properties, ts)`,
    )
    if err != nil {
        return err
    }

    for _, e := range events {
        props, _ := json.Marshal(e.Properties)
        if err := batch.Append(
            e.UserID, e.SessionID, e.EventType, e.Page, string(props), time.Now(),
        ); err != nil {
            return err
        }
    }

    return batch.Send()
}

func (r *EventRepository) Summarize(ctx context.Context, days int) ([]model.AggregateSummary, error) {
    rows, err := r.db.Query(ctx,
        `SELECT event_type, count() AS total, uniq(user_id) AS unique_users
         FROM analytics.events
         WHERE ts >= now() - INTERVAL ? DAY
         GROUP BY event_type
         ORDER BY total DESC`,
        days,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var results []model.AggregateSummary
    for rows.Next() {
        var s model.AggregateSummary
        if err := rows.Scan(&s.EventType, &s.Total, &s.UniqueUsers); err != nil {
            return nil, err
        }
        results = append(results, s)
    }
    return results, rows.Err()
}

func (r *EventRepository) TimeSeries(ctx context.Context, eventType string, hours int) ([]map[string]interface{}, error) {
    rows, err := r.db.Query(ctx,
        `SELECT toStartOfHour(ts) AS bucket, count() AS count
         FROM analytics.events
         WHERE event_type = ? AND ts >= now() - INTERVAL ? HOUR
         GROUP BY bucket ORDER BY bucket`,
        eventType, hours,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var results []map[string]interface{}
    for rows.Next() {
        var bucket time.Time
        var count uint64
        if err := rows.Scan(&bucket, &count); err != nil {
            return nil, err
        }
        results = append(results, map[string]interface{}{
            "bucket": bucket.Format(time.RFC3339),
            "count":  count,
        })
    }
    return results, rows.Err()
}
```

## Fiber Application

```go
// cmd/fiber/main.go
package main

import (
    "log"
    "os"
    "strconv"

    "github.com/gofiber/fiber/v2"
    "analytics/internal/db"
    "analytics/internal/model"
    "analytics/internal/repository"
)

func main() {
    host     := getEnv("CLICKHOUSE_HOST", "localhost")
    port, _  := strconv.Atoi(getEnv("CLICKHOUSE_PORT", "9000"))
    database := getEnv("CLICKHOUSE_DATABASE", "analytics")
    user     := getEnv("CLICKHOUSE_USER", "default")
    password := getEnv("CLICKHOUSE_PASSWORD", "")

    if err := db.Connect(host, port, database, user, password, false); err != nil {
        log.Fatalf("cannot connect to ClickHouse: %v", err)
    }

    repo := repository.NewEventRepository(db.DB)
    app  := fiber.New()

    app.Get("/health", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{"status": "ok"})
    })

    app.Post("/events", func(c *fiber.Ctx) error {
        var input model.EventInput
        if err := c.BodyParser(&input); err != nil {
            return c.Status(400).JSON(fiber.Map{"error": err.Error()})
        }
        if err := repo.Insert(c.Context(), input); err != nil {
            return c.Status(500).JSON(fiber.Map{"error": err.Error()})
        }
        return c.Status(201).JSON(fiber.Map{"status": "accepted"})
    })

    app.Post("/events/batch", func(c *fiber.Ctx) error {
        var events []model.EventInput
        if err := c.BodyParser(&events); err != nil {
            return c.Status(400).JSON(fiber.Map{"error": err.Error()})
        }
        if err := repo.InsertBatch(c.Context(), events); err != nil {
            return c.Status(500).JSON(fiber.Map{"error": err.Error()})
        }
        return c.Status(201).JSON(fiber.Map{"status": "accepted", "count": len(events)})
    })

    app.Get("/analytics/summary", func(c *fiber.Ctx) error {
        days, _ := strconv.Atoi(c.Query("days", "7"))
        data, err := repo.Summarize(c.Context(), days)
        if err != nil {
            return c.Status(500).JSON(fiber.Map{"error": err.Error()})
        }
        return c.JSON(data)
    })

    app.Get("/analytics/timeseries", func(c *fiber.Ctx) error {
        eventType := c.Query("event_type", "page_view")
        hours, _  := strconv.Atoi(c.Query("hours", "24"))
        data, err := repo.TimeSeries(c.Context(), eventType, hours)
        if err != nil {
            return c.Status(500).JSON(fiber.Map{"error": err.Error()})
        }
        return c.JSON(data)
    })

    log.Fatal(app.Listen(":3000"))
}

func getEnv(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}
```

## Gin Application

```go
// cmd/gin/main.go
package main

import (
    "log"
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
    "analytics/internal/db"
    "analytics/internal/model"
    "analytics/internal/repository"
)

func main() {
    if err := db.Connect("localhost", 9000, "analytics", "default", "", false); err != nil {
        log.Fatalf("cannot connect to ClickHouse: %v", err)
    }

    repo   := repository.NewEventRepository(db.DB)
    router := gin.Default()

    router.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    router.POST("/events", func(c *gin.Context) {
        var input model.EventInput
        if err := c.ShouldBindJSON(&input); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        if err := repo.Insert(c.Request.Context(), input); err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
            return
        }
        c.JSON(http.StatusCreated, gin.H{"status": "accepted"})
    })

    router.POST("/events/batch", func(c *gin.Context) {
        var events []model.EventInput
        if err := c.ShouldBindJSON(&events); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        if err := repo.InsertBatch(c.Request.Context(), events); err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
            return
        }
        c.JSON(http.StatusCreated, gin.H{"status": "accepted", "count": len(events)})
    })

    router.GET("/analytics/summary", func(c *gin.Context) {
        days, _ := strconv.Atoi(c.DefaultQuery("days", "7"))
        data, err := repo.Summarize(c.Request.Context(), days)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
            return
        }
        c.JSON(http.StatusOK, data)
    })

    router.GET("/analytics/timeseries", func(c *gin.Context) {
        eventType := c.DefaultQuery("event_type", "page_view")
        hours, _  := strconv.Atoi(c.DefaultQuery("hours", "24"))
        data, err := repo.TimeSeries(c.Request.Context(), eventType, hours)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
            return
        }
        c.JSON(http.StatusOK, data)
    })

    log.Fatal(router.Run(":3000"))
}
```

## Running the Applications

```bash
# Fiber
go run cmd/fiber/main.go

# Gin
go run cmd/gin/main.go
```

## Async Inserts

For very high-throughput ingestion, enable ClickHouse async inserts server-side:

```go
func (r *EventRepository) InsertAsync(ctx context.Context, e model.EventInput) error {
    props, _ := json.Marshal(e.Properties)
    return r.db.AsyncInsert(ctx,
        `INSERT INTO analytics.events (user_id, session_id, event_type, page, properties, ts)
         VALUES (?, ?, ?, ?, ?, ?)`,
        false, // wait=false: fire and forget
        e.UserID, e.SessionID, e.EventType, e.Page, string(props), time.Now(),
    )
}
```

Enable on the server side:

```sql
SET async_insert = 1;
SET wait_for_async_insert = 0;
```

## Summary

The `clickhouse-go` driver integrates with both Fiber and Gin through a shared repository layer. The critical production patterns are: use `PrepareBatch` for bulk inserts, pass `context.Context` through every call for timeout and cancellation support, and prefer the native port (9000) over HTTP for better throughput. The repository pattern isolates ClickHouse from your handler code, making it easy to test and swap implementations.
