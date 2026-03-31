# How to Build a ClickHouse Connection Retry Strategy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Retry, Resilience, Connection, Best Practice

Description: Learn how to implement a robust connection retry strategy for ClickHouse clients to handle transient failures and node restarts gracefully.

---

## Why You Need a Retry Strategy

ClickHouse nodes can be temporarily unavailable during restarts, merges, or network hiccups. Without a retry strategy, your application will surface errors to users for failures that would have resolved in seconds. A well-designed retry strategy absorbs these transient errors transparently.

## Exponential Backoff with Jitter

The standard pattern is exponential backoff with random jitter to avoid thundering herd effects when many clients retry simultaneously.

```python
import time
import random
from clickhouse_driver import Client
from clickhouse_driver.errors import NetworkError, SocketTimeoutError

def connect_with_retry(host, max_retries=5, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            client = Client(host=host, connect_timeout=5, send_receive_timeout=30)
            client.execute('SELECT 1')
            return client
        except (NetworkError, SocketTimeoutError) as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay:.1f}s")
            time.sleep(delay)
```

## Retry on Query Failure

Wrap query execution with retry logic separately from connection setup:

```python
def execute_with_retry(client, query, params=None, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.execute(query, params or {})
        except (NetworkError, SocketTimeoutError) as e:
            if attempt == max_retries - 1:
                raise
            delay = 0.5 * (2 ** attempt) + random.uniform(0, 0.5)
            time.sleep(delay)
```

## Go Client Example

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "time"
    "github.com/ClickHouse/clickhouse-go/v2"
)

func connectWithRetry(dsn string, maxRetries int) (clickhouse.Conn, error) {
    var conn clickhouse.Conn
    var err error
    for i := 0; i < maxRetries; i++ {
        conn, err = clickhouse.Open(&clickhouse.Options{Addr: []string{dsn}})
        if err == nil {
            if pingErr := conn.Ping(context.Background()); pingErr == nil {
                return conn, nil
            }
        }
        delay := time.Duration(500*(1<<i))*time.Millisecond +
            time.Duration(rand.Intn(500))*time.Millisecond
        fmt.Printf("Retry %d in %v\n", i+1, delay)
        time.Sleep(delay)
    }
    return nil, fmt.Errorf("failed after %d retries: %w", maxRetries, err)
}
```

## Circuit Breaker Pattern

For production systems, combine retries with a circuit breaker to stop retrying when the cluster is genuinely down:

```text
State: CLOSED (normal) -> failures exceed threshold -> OPEN (stop retrying)
OPEN -> timeout passes -> HALF-OPEN (test one request)
HALF-OPEN -> success -> CLOSED
HALF-OPEN -> failure -> OPEN
```

Use a library like `gobreaker` (Go) or `circuitbreaker` (Python) to implement this pattern.

## ClickHouse Client Settings for Resilience

```python
client = Client(
    host='clickhouse-host',
    connect_timeout=5,
    send_receive_timeout=60,
    sync_request_timeout=5,
    compress_block_size=1048576,
    settings={
        'connect_timeout_with_failover_ms': 50,
        'receive_timeout': 60,
        'send_timeout': 60,
    }
)
```

## Summary

A robust ClickHouse retry strategy uses exponential backoff with jitter to handle transient network errors and restarts. Implement retry logic at both the connection and query levels. For production systems, add a circuit breaker to prevent cascading failures when ClickHouse is genuinely unavailable for an extended period.
