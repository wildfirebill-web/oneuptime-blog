# How to Use Redis for Laravel Session Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Laravel, Sessions, PHP, Authentication

Description: Configure Laravel to store sessions in Redis for scalable, cross-server session sharing in multi-instance deployments with configurable TTL.

---

## Introduction

By default, Laravel stores sessions in files on the local filesystem, which prevents session sharing when running multiple application instances behind a load balancer. Switching to Redis session storage solves this problem and also offers faster session reads and writes compared to file-based storage.

## Prerequisites

Ensure Predis or phpredis is installed:

```bash
composer require predis/predis
```

## Environment Configuration

In `.env`:

```text
SESSION_DRIVER=redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=null
SESSION_LIFETIME=120
```

## Session Configuration

In `config/session.php`, verify the Redis connection is set:

```php
'driver'     => env('SESSION_DRIVER', 'redis'),
'lifetime'   => env('SESSION_LIFETIME', 120),
'connection' => 'session',  // Uses a dedicated Redis connection
'cookie'     => env('SESSION_COOKIE', 'laravel_session'),
'secure'     => env('SESSION_SECURE_COOKIE', false),
'http_only'  => true,
'same_site'  => 'lax',
```

## Dedicated Redis Connection for Sessions

In `config/database.php`, add a dedicated Redis connection to isolate session data:

```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'predis'),

    'default' => [
        'url'      => env('REDIS_URL'),
        'host'     => env('REDIS_HOST', '127.0.0.1'),
        'port'     => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],

    'session' => [
        'url'      => env('REDIS_URL'),
        'host'     => env('REDIS_HOST', '127.0.0.1'),
        'port'     => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_SESSION_DB', '1'), // Separate DB for sessions
    ],
],
```

## Using Sessions in Controllers

Laravel's session API is the same regardless of driver:

```php
use Illuminate\Http\Request;

class CartController extends Controller
{
    public function addItem(Request $request)
    {
        $cart = $request->session()->get('cart', []);
        $cart[] = [
            'product_id' => $request->input('product_id'),
            'quantity'   => $request->input('quantity', 1),
        ];
        $request->session()->put('cart', $cart);

        return response()->json(['cart_count' => count($cart)]);
    }

    public function viewCart(Request $request)
    {
        return response()->json([
            'cart' => $request->session()->get('cart', []),
        ]);
    }

    public function clearCart(Request $request)
    {
        $request->session()->forget('cart');
        return response()->json(['message' => 'Cart cleared']);
    }
}
```

## Session Flash Messages

```php
class CheckoutController extends Controller
{
    public function process(Request $request)
    {
        // Process order...

        // Flash a success message for the next request only
        $request->session()->flash('success', 'Order placed successfully!');
        return redirect('/orders');
    }
}
```

## Inspecting Sessions in Redis

```bash
# List all session keys
redis-cli keys "laravel:*"

# Get a specific session value
redis-cli get "laravel:<session-id>"

# Check session TTL
redis-cli ttl "laravel:<session-id>"
```

## Configuring Session Prefix

In `config/cache.php`, set the Redis prefix to separate session keys from cache keys:

```php
'prefix' => env('REDIS_PREFIX', 'laravel_session:'),
```

## Clearing All Sessions

```php
use Illuminate\Support\Facades\Redis;

class AdminController extends Controller
{
    public function clearAllSessions()
    {
        // Get all session keys and delete them
        $prefix = config('database.redis.options.prefix', 'laravel_session:');
        $keys = Redis::connection('session')->keys("{$prefix}*");

        if ($keys) {
            Redis::connection('session')->del($keys);
        }

        return response()->json(['message' => 'All sessions cleared']);
    }
}
```

## Summary

Configuring Laravel to use Redis for session storage requires setting `SESSION_DRIVER=redis` in `.env` and optionally specifying a dedicated Redis database connection for sessions in `config/database.php`. All session operations through the `$request->session()` API work identically to file-based sessions, but the data is stored in Redis, enabling session sharing across multiple application instances behind a load balancer.
