# How to Stream Large Result Sets from ClickHouse in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Go, Streaming, Large Result, Memory

Description: Stream large ClickHouse result sets in Go row by row instead of loading everything into memory, and process results as they arrive for memory-efficient pipelines.

---

## The Problem with Large Result Sets

A naive approach loads all query results into memory before processing:

```go
// Risky for large results - loads everything into RAM
var allRows []MyRow
rows, _ := conn.Query(ctx, "SELECT * FROM events WHERE event_date = '2024-01-01'")
for rows.Next() {
    var r MyRow
    rows.Scan(&r.UserID, &r.EventTime, &r.EventType)
    allRows = append(allRows, r)
}
```

For a query returning 100 million rows, this allocates gigabytes of memory. Streaming processes one row at a time.

## Streaming with rows.Next()

The ClickHouse Go client already streams results by default. `rows.Next()` pulls one row at a time:

```go
rows, err := conn.Query(ctx, "SELECT user_id, event_time, event_type FROM events WHERE event_date = today()")
if err != nil {
    log.Fatal(err)
}
defer rows.Close()

var processed int64
for rows.Next() {
    var (
        userID    uint64
        eventTime time.Time
        eventType string
    )
    if err := rows.Scan(&userID, &eventTime, &eventType); err != nil {
        log.Printf("Scan error: %v", err)
        continue
    }
    // Process row immediately - no accumulation
    processEvent(userID, eventTime, eventType)
    processed++
}

if err := rows.Err(); err != nil {
    log.Printf("Row iteration error: %v", err)
}
log.Printf("Processed %d rows", processed)
```

Memory usage stays constant regardless of result set size.

## Writing a Streaming Export to File

```go
func exportToCSV(conn clickhouse.Conn, ctx context.Context, outputPath string) error {
    f, err := os.Create(outputPath)
    if err != nil {
        return err
    }
    defer f.Close()
    w := bufio.NewWriterSize(f, 1<<20)  // 1 MB write buffer
    defer w.Flush()

    rows, err := conn.Query(ctx, "SELECT user_id, event_type, value FROM events")
    if err != nil {
        return err
    }
    defer rows.Close()

    for rows.Next() {
        var userID uint64
        var eventType string
        var value float64
        rows.Scan(&userID, &eventType, &value)
        fmt.Fprintf(w, "%d,%s,%.4f\n", userID, eventType, value)
    }
    return rows.Err()
}
```

## Streaming with a Worker Pool

Process rows faster by feeding them to a worker pool:

```go
jobs := make(chan EventRow, 1000)

// Start workers
var wg sync.WaitGroup
for i := 0; i < 8; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for row := range jobs {
            processEvent(row)
        }
    }()
}

// Stream rows into jobs channel
rows, _ := conn.Query(ctx, "SELECT user_id, event_time, event_type FROM events LIMIT 10000000")
defer rows.Close()
for rows.Next() {
    var row EventRow
    rows.Scan(&row.UserID, &row.EventTime, &row.EventType)
    jobs <- row
}
close(jobs)
wg.Wait()
```

## Controlling Fetch Block Size

ClickHouse sends results in blocks. The block size affects streaming latency vs. throughput:

```go
ctx := clickhouse.Context(context.Background(), clickhouse.WithSettings(clickhouse.Settings{
    "max_block_size": 65536,  // rows per block
}))
```

Smaller blocks reduce first-byte latency; larger blocks improve throughput.

## Summary

ClickHouse Go results are streamed by default via `rows.Next()`. Processing rows immediately rather than accumulating them keeps memory usage constant. Combine streaming with a buffered file writer or worker pool for efficient export and parallel processing pipelines. Tune `max_block_size` to balance latency and throughput.
