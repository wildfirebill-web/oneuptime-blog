# How to Use ClickHouse with Go Context for Timeouts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Go, Context, Timeout, Cancellation

Description: Learn how to use Go's context.Context with ClickHouse queries to enforce timeouts, propagate cancellation from HTTP requests, and set query-level settings via context.

---

## Why Context Matters with ClickHouse

Long-running ClickHouse queries can tie up connections and server resources. Go's `context.Context` lets you set deadlines and cancel queries when the calling code no longer needs the result - for example, when an HTTP client disconnects.

## Basic Timeout Context

```go
import (
    "context"
    "time"
)

func queryWithTimeout(conn clickhouse.Conn) (uint64, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    var count uint64
    row := conn.QueryRow(ctx, "SELECT count() FROM events WHERE event_date = today()")
    if err := row.Scan(&count); err != nil {
        return 0, fmt.Errorf("query failed or timed out: %w", err)
    }
    return count, nil
}
```

When the context deadline is exceeded, the ClickHouse client sends a cancel signal to the server, terminating the query immediately.

## Propagating HTTP Request Context

In a web handler, propagate the request's context so the query is cancelled if the HTTP client disconnects:

```go
func dashboardHandler(w http.ResponseWriter, r *http.Request) {
    // r.Context() is cancelled when the client disconnects
    ctx := r.Context()

    rows, err := conn.Query(ctx,
        "SELECT event_type, count() FROM events GROUP BY event_type LIMIT 20")
    if err != nil {
        if errors.Is(err, context.Canceled) {
            log.Println("Client disconnected before query completed")
            return
        }
        http.Error(w, "Query failed", 500)
        return
    }
    defer rows.Close()

    // ... encode and write results
}
```

## Setting Query-Level ClickHouse Settings via Context

The Go client supports injecting ClickHouse settings through the context:

```go
ctx := clickhouse.Context(context.Background(), clickhouse.WithSettings(clickhouse.Settings{
    "max_execution_time":    60,
    "max_memory_usage":      4294967296,  // 4 GB
    "max_threads":           4,
}))

rows, err := conn.Query(ctx, "SELECT user_id, sum(value) FROM events GROUP BY user_id")
```

This sets per-query settings without changing global or user-level defaults.

## Setting Query ID for Tracing

```go
ctx := clickhouse.Context(context.Background(),
    clickhouse.WithQueryID("dashboard-dau-20240101"))

rows, err := conn.Query(ctx, "SELECT uniq(user_id) FROM events WHERE event_date = today()")
```

Find the query later in `system.query_log`:

```sql
SELECT query_duration_ms, read_rows
FROM system.query_log
WHERE query_id = 'dashboard-dau-20240101' AND type = 'QueryFinish';
```

## Deadline vs. Timeout

```go
// Timeout: relative duration from now
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)

// Deadline: absolute time
deadline := time.Now().Add(30 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
```

Use deadline when you have a fixed wall-clock time by which results must arrive (e.g., SLA enforcement).

## Detecting Context Errors

```go
if err != nil {
    switch {
    case errors.Is(err, context.DeadlineExceeded):
        log.Println("Query timed out")
    case errors.Is(err, context.Canceled):
        log.Println("Query was cancelled")
    default:
        log.Printf("Query error: %v", err)
    }
}
```

## Summary

Go context integration with ClickHouse enables timeout enforcement, client-disconnect cancellation, per-query settings injection, and query ID tracing. Always pass `context.Context` from the caller down to ClickHouse queries so deadlines and cancellations propagate correctly through the entire call stack.
