# How to Handle IPv6 Addresses in PHP Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: PHP, IPv6, Networking, Validation, Socket Programming, Web Development

Description: Handle, validate, parse, and use IPv6 addresses in PHP applications using filter_var, inet_pton, and socket functions for web and CLI applications.

## Introduction

PHP provides IPv6 support through built-in functions including `filter_var()` with `FILTER_VALIDATE_IP`, `inet_pton()`, and `inet_ntop()`. PHP web applications running behind proxies need careful handling of IPv6 client addresses in `$_SERVER` superglobals.

## Validating IPv6 Addresses

```php
<?php

/**
 * Validate an IPv6 address using PHP's built-in filter.
 */
function isIPv6(string $address): bool {
    // Strip zone ID if present (e.g., "fe80::1%eth0")
    $clean = explode('%', $address)[0];

    return filter_var($clean, FILTER_VALIDATE_IP, FILTER_FLAG_IPV6) !== false;
}

/**
 * Check if an address is IPv4 or IPv6.
 */
function getIPVersion(string $address): ?int {
    $clean = explode('%', $address)[0];

    if (filter_var($clean, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4)) {
        return 4;
    }
    if (filter_var($clean, FILTER_VALIDATE_IP, FILTER_FLAG_IPV6)) {
        return 6;
    }
    return null;
}

// Test cases
$addresses = [
    '2001:db8::1',
    '::1',
    'fe80::1%eth0',
    '192.168.1.1',
    'not-an-address',
];

foreach ($addresses as $addr) {
    $version = getIPVersion($addr);
    echo sprintf("%-25s version=%s\n", $addr, $version ?? 'invalid');
}
```

## Expanding and Compressing IPv6 Addresses

```php
<?php

/**
 * Expand a compressed IPv6 address to full notation.
 * e.g., "2001:db8::1" → "2001:0db8:0000:0000:0000:0000:0000:0001"
 */
function expandIPv6(string $address): string {
    $clean = explode('%', $address)[0];
    // inet_pton converts to packed binary, inet_ntop back to string
    $packed = inet_pton($clean);
    if ($packed === false) {
        throw new InvalidArgumentException("Invalid IPv6: $address");
    }
    // Unpack as hex, then format in groups of 4
    $hex = bin2hex($packed);
    $groups = str_split($hex, 4);
    return implode(':', $groups);
}

/**
 * Compress an IPv6 address to shortest form.
 */
function compressIPv6(string $address): string {
    $packed = inet_pton($address);
    if ($packed === false) {
        throw new InvalidArgumentException("Invalid IPv6: $address");
    }
    return inet_ntop($packed);
}

echo expandIPv6('2001:db8::1');
// Output: 2001:0db8:0000:0000:0000:0000:0000:0001

echo compressIPv6('2001:0db8:0000:0000:0000:0000:0000:0001');
// Output: 2001:db8::1
```

## Getting the Real Client IP in PHP

```php
<?php

/**
 * Get the real client IP address from the request,
 * handling IPv6, IPv4-mapped IPv6, and proxy headers.
 */
function getClientIP(): string {
    // Check proxy headers (order of trust)
    $headers = [
        'HTTP_CF_CONNECTING_IP',     // Cloudflare
        'HTTP_X_FORWARDED_FOR',
        'HTTP_X_REAL_IP',
        'REMOTE_ADDR',
    ];

    foreach ($headers as $header) {
        if (!empty($_SERVER[$header])) {
            // X-Forwarded-For can be a comma-separated list
            $ip = trim(explode(',', $_SERVER[$header])[0]);

            // Remove IPv4-mapped IPv6 prefix
            $ip = preg_replace('/^::ffff:/i', '', $ip);

            // Strip zone ID
            $ip = explode('%', $ip)[0];

            if (filter_var($ip, FILTER_VALIDATE_IP)) {
                return $ip;
            }
        }
    }

    return '0.0.0.0';
}

$clientIP = getClientIP();
$isIPv6 = filter_var($clientIP, FILTER_VALIDATE_IP, FILTER_FLAG_IPV6);
echo "Client IP: $clientIP (" . ($isIPv6 ? 'IPv6' : 'IPv4') . ")";
```

## Checking Subnet Membership

```php
<?php

/**
 * Check if an IPv6 address belongs to a given CIDR block.
 */
function ipv6InCidr(string $ip, string $cidr): bool {
    [$network, $prefix] = explode('/', $cidr);
    $prefix = (int) $prefix;

    // Convert addresses to packed binary
    $ipBinary = inet_pton($ip);
    $networkBinary = inet_pton($network);

    if ($ipBinary === false || $networkBinary === false) {
        return false;
    }

    // Build bitmask from prefix length
    $bits = str_repeat('1', $prefix) . str_repeat('0', 128 - $prefix);
    $mask = '';
    foreach (str_split($bits, 8) as $byte) {
        $mask .= chr(bindec($byte));
    }

    return ($ipBinary & $mask) === ($networkBinary & $mask);
}

// Test
echo ipv6InCidr('2001:db8::1', '2001:db8::/32') ? 'yes' : 'no';   // yes
echo ipv6InCidr('2001:db9::1', '2001:db8::/32') ? 'yes' : 'no';   // no
```

## Creating a Socket Connection to IPv6

```php
<?php

// Create an IPv6 TCP socket
$socket = socket_create(AF_INET6, SOCK_STREAM, SOL_TCP);
if ($socket === false) {
    die('socket_create failed: ' . socket_strerror(socket_last_error()));
}

// Connect to an IPv6 server (no brackets needed in socket functions)
$result = socket_connect($socket, '2001:db8::1', 8080);
if ($result === false) {
    die('socket_connect failed: ' . socket_strerror(socket_last_error($socket)));
}

// Send and receive data
socket_write($socket, "GET / HTTP/1.0\r\nHost: [2001:db8::1]\r\n\r\n");
$response = socket_read($socket, 4096);
echo $response;
socket_close($socket);
```

## Formatting IPv6 for URLs in PHP

```php
<?php

/**
 * Format an IP address for use in a URL.
 * IPv6 addresses require bracket notation per RFC 2732.
 */
function formatIPForUrl(string $ip): string {
    $clean = explode('%', $ip)[0];
    if (filter_var($clean, FILTER_VALIDATE_IP, FILTER_FLAG_IPV6)) {
        return "[{$clean}]";
    }
    return $clean;
}

$ipv6 = '2001:db8::1';
$url = 'https://' . formatIPForUrl($ipv6) . ':443/api/v1';
echo $url;  // https://[2001:db8::1]:443/api/v1
```

## Conclusion

PHP handles IPv6 through `filter_var()` with `FILTER_FLAG_IPV6`, `inet_pton()`/`inet_ntop()` for binary conversion, and `AF_INET6` for socket creation. Always strip zone IDs and IPv4-mapped prefixes when processing client IP addresses, and use bracket notation when building IPv6 URLs.
