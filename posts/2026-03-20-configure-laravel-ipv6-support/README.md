# How to Configure Laravel for IPv6 Support

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Laravel, PHP, IPv6, Web Framework, Octane, FPM, Dual-Stack

Description: Configure Laravel to handle IPv6 client requests, trust IPv6 proxies, validate IPv6 inputs, and run behind an IPv6-capable web server.

## Introduction

Laravel itself is PHP and relies on the web server (NGINX/Apache/Laravel Octane) for IPv6 binding. The framework's role is to correctly parse IPv6 client addresses from request headers and validate IPv6 inputs in forms and API endpoints.

## Step 1: Trust IPv6 Proxies

```php
// app/Http/Middleware/TrustProxies.php
namespace App\Http\Middleware;

use Illuminate\Http\Middleware\TrustProxies as Middleware;
use Illuminate\Http\Request;

class TrustProxies extends Middleware
{
    // Trust specific IPv6 proxy
    protected $proxies = [
        '2001:db8::1',        // NGINX IPv6
        '::1',                 // Loopback
        '2001:db8::/32',       // Subnet (CIDR)
    ];

    // Or trust all proxies (use carefully)
    // protected $proxies = '*';

    protected $headers =
        Request::HEADER_X_FORWARDED_FOR |
        Request::HEADER_X_FORWARDED_HOST |
        Request::HEADER_X_FORWARDED_PORT |
        Request::HEADER_X_FORWARDED_PROTO;
}
```

## Step 2: Get Client IPv6 in Controllers

```php
// app/Http/Controllers/ApiController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class ApiController extends Controller
{
    public function clientInfo(Request $request): \Illuminate\Http\JsonResponse
    {
        // Laravel uses TrustProxies to resolve the real IP
        $ip = $request->ip();

        // Check if IPv6
        $isIPv6 = filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV6) !== false;

        return response()->json([
            'ip'         => $ip,
            'is_ipv6'    => $isIPv6,
            'is_private' => $this->isPrivateIP($ip),
        ]);
    }

    private function isPrivateIP(string $ip): bool
    {
        return filter_var(
            $ip,
            FILTER_VALIDATE_IP,
            FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE
        ) === false;
    }
}
```

## Step 3: Form Validation for IPv6

```php
// app/Http/Requests/NetworkRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class NetworkRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            // Validate as IPv6 specifically
            'ipv6_address' => [
                'required',
                'ip',
                function ($attribute, $value, $fail) {
                    if (!filter_var($value, FILTER_VALIDATE_IP, FILTER_FLAG_IPV6)) {
                        $fail("The $attribute must be a valid IPv6 address.");
                    }
                    // Reject loopback and unspecified
                    if (in_array($value, ['::1', '::'])) {
                        $fail("The $attribute cannot be a special-purpose address.");
                    }
                },
            ],
            // Accept both IPv4 and IPv6
            'ip_address' => 'required|ip',
        ];
    }
}
```

## Step 4: Store IPv6 in Database

```php
// database/migrations/xxxx_create_connections_table.php

Schema::create('connections', function (Blueprint $table) {
    $table->id();
    $table->string('ip_address', 45);  // 45 chars for longest IPv6
    $table->ipAddress('client_ip');     // Laravel helper (varchar 45)
    $table->timestamps();
});
```

```php
// Eloquent model accessor to normalize IPv6
// app/Models/Connection.php

public function getIpAddressAttribute(string $value): string
{
    // Normalize IPv4-mapped IPv6 addresses
    if (str_starts_with($value, '::ffff:')) {
        return substr($value, 7);
    }
    return $value;
}
```

## Step 5: NGINX Configuration for IPv6 + PHP-FPM

```nginx
server {
    listen [::]:80;
    listen 80;
    server_name example.com;
    root /var/www/html/public;

    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        # Pass real IPv6 client address
        fastcgi_param REMOTE_ADDR $remote_addr;
    }
}
```

## Step 6: Laravel Octane on IPv6

```bash
# Install Octane with Swoole
composer require laravel/octane
php artisan octane:install --server=swoole

# Start Octane on IPv6
php artisan octane:start --server=swoole --host="::" --port=8000

# Or with FrankenPHP
php artisan octane:start --server=frankenphp --host="::" --port=8000
```

## Conclusion

Laravel's IPv6 support centers on `TrustProxies` middleware with IPv6 proxy addresses, `ip()` on `Request` for client IP extraction, and `ipAddress()` column type for IPv6-aware database storage. Deploy behind NGINX with `listen [::]:80` and pass real client IPs via FastCGI. Monitor Laravel with OneUptime from both IPv4 and IPv6 vantage points.
