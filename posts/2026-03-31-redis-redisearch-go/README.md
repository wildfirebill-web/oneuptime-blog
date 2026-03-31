# How to Use RediSearch with Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Go, RediSearch

Description: Learn how to create RediSearch indexes, index documents, and run full-text and filtered queries from Go using go-redis.

---

RediSearch adds full-text search and secondary indexing to Redis. From Go, you can use go-redis `Do` method to send `FT.CREATE`, `FT.SEARCH`, and `FT.AGGREGATE` commands, giving you a fast search engine without an external service.

## Setup

Run Redis Stack which includes RediSearch:

```bash
docker run -d -p 6379:6379 redis/redis-stack:latest
```

## Create an Index

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/redis/go-redis/v9"
)

func createIndex(ctx context.Context, rdb *redis.Client) {
    err := rdb.Do(ctx,
        "FT.CREATE", "product-idx",
        "ON", "HASH",
        "PREFIX", "1", "product:",
        "SCHEMA",
        "title", "TEXT", "WEIGHT", "5",
        "description", "TEXT",
        "price", "NUMERIC", "SORTABLE",
        "category", "TAG",
    ).Err()

    if err != nil && err.Error() != "Index already exists" {
        log.Fatal(err)
    }
}
```

## Index Documents as Hashes

RediSearch automatically indexes hashes that match the prefix:

```go
func indexProducts(ctx context.Context, rdb *redis.Client) {
    rdb.HSet(ctx, "product:1",
        "title", "Wireless Headphones",
        "description", "Noise-cancelling bluetooth headphones",
        "price", "99.99",
        "category", "electronics",
    )

    rdb.HSet(ctx, "product:2",
        "title", "Trail Running Shoes",
        "description", "Lightweight shoes for mountain running",
        "price", "79.99",
        "category", "sports",
    )

    rdb.HSet(ctx, "product:3",
        "title", "Yoga Mat",
        "description", "Non-slip premium yoga mat",
        "price", "29.99",
        "category", "sports",
    )
}
```

## Full-Text Search

```go
func search(ctx context.Context, rdb *redis.Client, query string) {
    result, err := rdb.Do(ctx, "FT.SEARCH", "product-idx", query).Result()
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Results for '%s': %v\n", query, result)
}

// Examples
search(ctx, rdb, "headphones")
search(ctx, rdb, "running")
```

## Numeric Filter

```go
// Products priced between 50 and 100
result, err := rdb.Do(ctx,
    "FT.SEARCH", "product-idx",
    "@price:[50 100]",
    "RETURN", "2", "title", "price",
).Result()
```

## Tag Filter

```go
// All sports products
result, err := rdb.Do(ctx,
    "FT.SEARCH", "product-idx",
    "@category:{sports}",
    "SORTBY", "price", "ASC",
).Result()
```

## Combined Query

```go
// Sports products under $50
result, err := rdb.Do(ctx,
    "FT.SEARCH", "product-idx",
    "@category:{sports} @price:[0 50]",
    "RETURN", "3", "title", "price", "category",
).Result()
```

## Pagination

```go
result, err := rdb.Do(ctx,
    "FT.SEARCH", "product-idx",
    "*",
    "SORTBY", "price", "ASC",
    "LIMIT", "0", "10", // offset 0, 10 results
).Result()
```

## Aggregation - Group by Category with Count

```go
result, err := rdb.Do(ctx,
    "FT.AGGREGATE", "product-idx", "*",
    "GROUPBY", "1", "@category",
    "REDUCE", "COUNT", "0", "AS", "count",
    "SORTBY", "2", "@count", "DESC",
).Result()
if err != nil {
    log.Fatal(err)
}
fmt.Println(result)
```

## Drop an Index

```go
rdb.Do(ctx, "FT.DROPINDEX", "product-idx")
// Pass "DD" to also delete the documents
rdb.Do(ctx, "FT.DROPINDEX", "product-idx", "DD")
```

## Summary

RediSearch from Go uses the `Do` method to send `FT.CREATE`, `FT.SEARCH`, and `FT.AGGREGATE` commands. Indexes are defined over hash key prefixes and support text, numeric, and tag field types. Queries combine full-text search with filter expressions, sorting, and pagination - all running inside Redis without an external search engine.
