# How to Use ClickHouse with Go Generics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Go, Generics, Type Safety, Analytics

Description: Use Go generics to build type-safe ClickHouse query helpers that reduce boilerplate and eliminate runtime type assertion errors.

---

## Why Generics Help with ClickHouse

ClickHouse queries often return rows that need scanning into Go structs. Without generics, you write repetitive scanning code for every return type. Go 1.18+ generics let you write it once.

## A Generic Row Scanner

```go
package chutil

import (
    "context"
    "github.com/ClickHouse/clickhouse-go/v2"
)

func QueryAll[T any](
    ctx context.Context,
    conn clickhouse.Conn,
    query string,
    scan func(*T) []any,
    args ...any,
) ([]T, error) {
    rows, err := conn.Query(ctx, query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var results []T
    for rows.Next() {
        var item T
        if err := rows.Scan(scan(&item)...); err != nil {
            return nil, err
        }
        results = append(results, item)
    }
    return results, rows.Err()
}
```

## Defining a Domain Struct

```go
type PageView struct {
    Path  string
    Count uint64
    Date  time.Time
}
```

## Calling the Generic Helper

```go
views, err := chutil.QueryAll(
    ctx, conn,
    `SELECT path, count() AS count, toDate(timestamp) AS date
     FROM page_views
     WHERE date >= today() - 7
     GROUP BY path, date
     ORDER BY count DESC`,
    func(v *PageView) []any {
        return []any{&v.Path, &v.Count, &v.Date}
    },
)
```

The compiler infers `T = PageView` from the scan function. No type assertions at runtime.

## A Generic Single-Row Query

```go
func QueryOne[T any](
    ctx context.Context,
    conn clickhouse.Conn,
    query string,
    scan func(*T) []any,
    args ...any,
) (T, error) {
    var zero T
    rows, err := conn.Query(ctx, query, args...)
    if err != nil {
        return zero, err
    }
    defer rows.Close()
    if rows.Next() {
        if err := rows.Scan(scan(&zero)...); err != nil {
            return zero, err
        }
    }
    return zero, rows.Err()
}
```

## Handling Optional Results with a Generic Pointer

```go
func QueryOptional[T any](
    ctx context.Context,
    conn clickhouse.Conn,
    query string,
    scan func(*T) []any,
    args ...any,
) (*T, error) {
    result, err := QueryOne[T](ctx, conn, query, scan, args...)
    if err != nil {
        return nil, err
    }
    return &result, nil
}
```

## Adding Pagination

Generics compose cleanly with pagination wrappers.

```go
type Page[T any] struct {
    Items []T
    Total uint64
}
```

Pass your `QueryAll` results into `Page[PageView]` without any casting.

## Summary

Go generics eliminate repetitive scanning boilerplate when working with ClickHouse. A small set of generic helpers - `QueryAll`, `QueryOne`, and `QueryOptional` - covers the majority of read patterns while keeping code type-safe and easy to test.
