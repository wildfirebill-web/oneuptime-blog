# How to Implement Cache-Aside Pattern with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cache, State Management, Cache-Aside, Pattern

Description: Learn how to implement the cache-aside (lazy loading) pattern with Dapr state management to cache data on demand and reduce database load.

---

## What is Cache-Aside?

Cache-aside (also called lazy loading) puts the application in charge of the cache. The application checks the cache before going to the database, loads data from the database on a miss, writes it to the cache, and returns it. Unlike read-through or write-through, the cache is not involved in every operation automatically - the application manages it explicitly.

This makes cache-aside simple to implement and easy to reason about, but it means every cache miss causes two round trips (cache + database).

## Setting Up a Redis State Store for Caching

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: cache
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis:6379
  - name: redisPassword
    value: ""
```

## Implementing Cache-Aside in Go

```go
package cache

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
)

const daprPort = "3500"
const cacheStore = "cache"

type Product struct {
    ID    string  `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

func GetFromCache(ctx context.Context, key string) (*Product, error) {
    url := fmt.Sprintf("http://localhost:%s/v1.0/state/%s/%s", daprPort, cacheStore, key)
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil || resp.StatusCode != 200 {
        return nil, nil
    }
    defer resp.Body.Close()
    var product Product
    if err := json.NewDecoder(resp.Body).Decode(&product); err != nil {
        return nil, nil
    }
    return &product, nil
}

func SetInCache(ctx context.Context, key string, product *Product, ttlSeconds int) error {
    payload := []map[string]interface{}{
        {
            "key":   key,
            "value": product,
            "metadata": map[string]string{
                "ttlInSeconds": fmt.Sprintf("%d", ttlSeconds),
            },
        },
    }
    body, _ := json.Marshal(payload)
    url := fmt.Sprintf("http://localhost:%s/v1.0/state/%s", daprPort, cacheStore)
    req, _ := http.NewRequestWithContext(ctx, http.MethodPost, url, strings.NewReader(string(body)))
    req.Header.Set("Content-Type", "application/json")
    _, err := http.DefaultClient.Do(req)
    return err
}

func GetProduct(ctx context.Context, productID string, db ProductDB) (*Product, error) {
    cacheKey := fmt.Sprintf("product:%s", productID)

    // Check cache first
    cached, _ := GetFromCache(ctx, cacheKey)
    if cached != nil {
        return cached, nil
    }

    // Cache miss - load from database
    product, err := db.FindByID(ctx, productID)
    if err != nil {
        return nil, err
    }

    // Populate cache (fire and forget on error)
    _ = SetInCache(ctx, cacheKey, product, 300)

    return product, nil
}
```

## Invalidating the Cache on Updates

When data changes, delete the cache entry so the next read fetches fresh data:

```go
func UpdateProduct(ctx context.Context, product *Product, db ProductDB) error {
    // Update the database
    if err := db.Save(ctx, product); err != nil {
        return err
    }

    // Invalidate the cache entry
    cacheKey := fmt.Sprintf("product:%s", product.ID)
    url := fmt.Sprintf("http://localhost:%s/v1.0/state/%s/%s", daprPort, cacheStore, cacheKey)
    req, _ := http.NewRequestWithContext(ctx, http.MethodDelete, url, nil)
    http.DefaultClient.Do(req)

    return nil
}
```

## When to Use Cache-Aside vs Other Patterns

Cache-aside is the right choice when:
- Data is read frequently but updated infrequently
- Cache misses are acceptable (first read is slower)
- You want full control over what goes into the cache
- The cache can be populated lazily without a warm-up phase

Avoid cache-aside for write-heavy workloads where the cache would be constantly invalidated.

## Summary

The cache-aside pattern with Dapr state management keeps the application in control of caching decisions. Each read checks the Dapr state store first, falls back to the database on a miss, and populates the cache for subsequent reads. Updates invalidate the relevant cache key, ensuring consistency without complex synchronization logic.
