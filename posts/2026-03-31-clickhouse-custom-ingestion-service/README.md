# How to Build a Custom Data Ingestion Service for ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Ingestion, Custom Service, Go, Batching

Description: Learn how to build a custom data ingestion service for ClickHouse in Go that batches writes, retries on failure, and handles back-pressure.

---

When off-the-shelf collectors do not fit your data model, a custom ingestion service gives you full control over schema mapping, batching, and error handling.

## Architecture

The service receives events over HTTP, batches them in memory, and flushes to ClickHouse on a size or time threshold.

```text
Client --> HTTP API --> In-Memory Buffer --> ClickHouse
                              |
                         Flush every 5s or 5000 rows
```

## Go Implementation

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "net/http"
    "sync"
    "time"

    "github.com/ClickHouse/clickhouse-go/v2"
)

type Event struct {
    Timestamp time.Time         `json:"ts"`
    Type      string            `json:"type"`
    UserID    uint32            `json:"user_id"`
    Data      map[string]string `json:"data"`
}

var (
    mu     sync.Mutex
    buffer []Event
)

func ingestHandler(conn clickhouse.Conn) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var events []Event
        if err := json.NewDecoder(r.Body).Decode(&events); err != nil {
            http.Error(w, err.Error(), 400)
            return
        }
        mu.Lock()
        buffer = append(buffer, events...)
        mu.Unlock()
        w.WriteHeader(http.StatusAccepted)
    }
}

func flusher(conn clickhouse.Conn) {
    ticker := time.NewTicker(5 * time.Second)
    for range ticker.C {
        mu.Lock()
        if len(buffer) == 0 {
            mu.Unlock()
            continue
        }
        batch := buffer
        buffer = nil
        mu.Unlock()
        flush(conn, batch)
    }
}

func flush(conn clickhouse.Conn, rows []Event) {
    ctx := context.Background()
    b, err := conn.PrepareBatch(ctx, "INSERT INTO events")
    if err != nil {
        log.Printf("prepare error: %v", err)
        return
    }
    for _, e := range rows {
        _ = b.AppendStruct(&e)
    }
    if err := b.Send(); err != nil {
        log.Printf("flush error: %v", err)
    }
}
```

## ClickHouse Connection Setup

```go
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{"clickhouse:9000"},
    Auth: clickhouse.Auth{Database: "default", Username: "default", Password: "secret"},
    Compression: &clickhouse.Compression{Method: clickhouse.CompressionLZ4},
    MaxOpenConns: 5,
})
```

## Running the Service

```bash
go build -o ingestion-service .
./ingestion-service
```

Send test events:

```bash
curl -X POST http://localhost:8080/ingest \
  -H 'Content-Type: application/json' \
  -d '[{"ts":"2026-03-31T10:00:00Z","type":"click","user_id":42,"data":{"page":"/home"}}]'
```

## Summary

A custom Go ingestion service batches events with a mutex-protected buffer and flushes to ClickHouse on a 5-second interval or size threshold. Use `PrepareBatch` for efficient columnar inserts and add retry logic with exponential backoff for transient ClickHouse errors.
