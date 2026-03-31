# How to Cache Laravel Eloquent Queries with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Laravel, Eloquent, Cache, PHP

Description: Learn how to cache Laravel Eloquent query results with Redis to reduce database load and speed up your application responses.

---

Caching Eloquent query results with Redis is one of the quickest wins you can get in a Laravel application. Instead of hitting the database on every request, you store query results in Redis and serve them from memory.

## Install and Configure the Redis Driver

Make sure you have `predis/predis` or the `phpredis` extension installed, then set `CACHE_DRIVER=redis` in your `.env` file.

```bash
composer require predis/predis
```

Update `.env`:

```text
CACHE_DRIVER=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

## Cache a Simple Eloquent Query

Use `Cache::remember` to wrap any Eloquent query. The closure only runs when the cache key is missing.

```php
use Illuminate\Support\Facades\Cache;
use App\Models\Product;

$products = Cache::remember('products.all', now()->addMinutes(30), function () {
    return Product::with('category')->where('active', true)->get();
});
```

The first argument is the cache key, the second is the TTL, and the third is the callable that fetches data when the cache is cold.

## Cache Queries with Dynamic Keys

When queries depend on user input, embed that input in the key to avoid cache collisions.

```php
$categoryId = 5;
$page = 1;

$products = Cache::remember("products.category.{$categoryId}.page.{$page}", 600, function () use ($categoryId, $page) {
    return Product::where('category_id', $categoryId)
        ->paginate(20, ['*'], 'page', $page);
});
```

## Invalidate the Cache on Model Changes

Use Eloquent model observers to clear stale cache entries whenever a record is created, updated, or deleted.

```php
namespace App\Observers;

use App\Models\Product;
use Illuminate\Support\Facades\Cache;

class ProductObserver
{
    public function saved(Product $product): void
    {
        Cache::forget('products.all');
        Cache::forget("products.category.{$product->category_id}.page.1");
    }

    public function deleted(Product $product): void
    {
        Cache::forget('products.all');
    }
}
```

Register the observer in `AppServiceProvider`:

```php
use App\Models\Product;
use App\Observers\ProductObserver;

public function boot(): void
{
    Product::observe(ProductObserver::class);
}
```

## Tag Related Cache Entries

Laravel's Redis cache driver supports cache tags, making bulk invalidation simple.

```php
$products = Cache::tags(['products', 'category-5'])->remember('products.category.5', 600, function () {
    return Product::where('category_id', 5)->get();
});

// Invalidate all product-tagged cache entries at once
Cache::tags(['products'])->flush();
```

Tags require a Redis or Memcached cache driver - the file and database drivers do not support them.

## Avoid Caching Sensitive or User-Specific Data

Never cache queries that return user-specific sensitive data under a shared key. Always include the user ID in the cache key for personalized results.

```php
$userId = auth()->id();
$orders = Cache::remember("orders.user.{$userId}", 300, function () use ($userId) {
    return Order::where('user_id', $userId)->latest()->take(10)->get();
});
```

## Monitor Cache Performance

You can check whether your cache is being hit with a simple wrapper:

```php
$hit = Cache::has('products.all');
logger("Cache hit: " . ($hit ? 'yes' : 'no'));
```

For production monitoring, use OneUptime or a similar observability platform to track cache hit rates and application latency over time.

## Summary

Caching Eloquent queries with Redis in Laravel is straightforward using `Cache::remember` and cache tags. The key to correctness is proper invalidation - use model observers to clear stale entries whenever data changes. Always include dynamic parameters like user IDs or page numbers in your cache keys to prevent data leakage between users.
