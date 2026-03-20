# How to Validate IPv4 Addresses in PHP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: PHP, IPv4, Validation, filter_var, Regex, Networking

Description: Learn how to validate IPv4 address strings in PHP using filter_var with FILTER_VALIDATE_IP, custom regex patterns, and practical request validation middleware examples.

## Using filter_var (Recommended)

```php
<?php

function isValidIPv4(string $ip): bool {
    return filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4) !== false;
}

$tests = [
    '192.168.1.1'     => true,
    '0.0.0.0'         => true,
    '255.255.255.255' => true,
    '256.0.0.1'       => false,   // Octet > 255
    '192.168.1'       => false,   // Missing octet
    '192.168.1.1.1'   => false,   // Extra octet
    '192.168.01.1'    => false,   // Leading zero — rejected by filter_var
    '::1'             => false,   // IPv6
    ''                => false,   // Empty
];

foreach ($tests as $ip => $expected) {
    $result = isValidIPv4($ip);
    $status = ($result === $expected) ? 'PASS' : 'FAIL';
    printf("[%s] %-25s -> %s\n", $status, $ip, $result ? 'true' : 'false');
}
```

## Strict Regex (For When Leading Zeros Must Be Rejected Explicitly)

```php
<?php

const IPV4_REGEX =
    '/^(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)' .
    '(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}$/';

function isValidIPv4Regex(string $ip): bool {
    return (bool) preg_match(IPV4_REGEX, $ip);
}

echo isValidIPv4Regex('192.168.1.1')   ? "valid\n" : "invalid\n";  // valid
echo isValidIPv4Regex('192.168.01.1')  ? "valid\n" : "invalid\n";  // invalid
echo isValidIPv4Regex('256.0.0.1')     ? "valid\n" : "invalid\n";  // invalid
```

## Checking Private vs Public

```php
<?php

function isPrivateIPv4(string $ip): bool {
    return filter_var(
        $ip,
        FILTER_VALIDATE_IP,
        FILTER_FLAG_IPV4 | FILTER_FLAG_NO_PRIV_RANGE
    ) === false
    && filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4) !== false;
}

function isPublicIPv4(string $ip): bool {
    return filter_var(
        $ip,
        FILTER_VALIDATE_IP,
        FILTER_FLAG_IPV4 | FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE
    ) !== false;
}

echo isPublicIPv4('8.8.8.8')        ? "public\n"  : "private\n";  // public
echo isPublicIPv4('192.168.1.1')    ? "public\n"  : "private\n";  // private
echo isPrivateIPv4('192.168.1.1')   ? "private\n" : "public\n";   // private
```

## Laravel Request Validation

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class ConnectRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'target_ip'   => ['required', 'ip', 'ipv4'],
            'target_port' => ['required', 'integer', 'between:1,65535'],
        ];
    }

    public function messages(): array
    {
        return [
            'target_ip.ipv4' => 'The target must be a valid IPv4 address.',
        ];
    }
}
```

## Extracting IPs from Text

```php
<?php

function extractIPv4(string $text): array {
    $octet = '(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)';
    preg_match_all(
        "/\b{$octet}(?:\.{$octet}){3}\b/",
        $text,
        $matches
    );
    return $matches[0];
}

$log = '[2026-03-20] Request from 192.168.1.50 forwarded to 10.0.0.1';
print_r(extractIPv4($log));
// Array ( [0] => 192.168.1.50 [1] => 10.0.0.1 )
```

## Conclusion

`filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4)` is the idiomatic PHP approach — it correctly rejects leading zeros, out-of-range octets, and IPv6 addresses. Combine with `FILTER_FLAG_NO_PRIV_RANGE` and `FILTER_FLAG_NO_RES_RANGE` to enforce public-only IPs. In Laravel, the built-in `ip` and `ipv4` validation rules delegate to `filter_var` under the hood. Use the regex approach only when you need to extract IPs from freeform text.
