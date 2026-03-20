# How to Validate IPv6 Addresses in User Input

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Validation, Security, Web Development, Input Sanitization, Programming

Description: Implement robust IPv6 address validation for user input across multiple languages, handling edge cases like zone IDs, IPv4-mapped addresses, and CIDR notation.

## Introduction

Validating IPv6 addresses in user input prevents injection attacks, database errors, and application crashes. IPv6 validation is more complex than IPv4 due to multiple valid representations of the same address, optional zone IDs, and CIDR notation. This guide covers validation patterns for common languages.

## What Makes IPv6 Validation Hard

Valid IPv6 representations of the same address:
```text
2001:0db8:0000:0000:0000:0000:0000:0001  # Full notation
2001:db8::1                              # Compressed
2001:DB8::1                              # Case-insensitive
::1                                      # Loopback
::ffff:192.168.1.1                       # IPv4-mapped
2001:db8::1/64                           # CIDR notation
fe80::1%eth0                             # Link-local with zone ID
[2001:db8::1]                            # URL bracket notation
```

## Validation in Python

```python
import ipaddress
import re

def validate_ipv6_input(user_input: str) -> dict:
    """
    Validate an IPv6 address from user input.
    Returns a dict with is_valid, normalized form, and type info.
    """
    result = {
        'is_valid': False,
        'normalized': None,
        'type': None,
        'original': user_input,
    }

    # Strip surrounding whitespace
    cleaned = user_input.strip()

    # Remove brackets if present (URL notation)
    if cleaned.startswith('[') and ']' in cleaned:
        cleaned = cleaned[1:cleaned.index(']')]

    # Strip zone ID
    zone_id = None
    if '%' in cleaned:
        cleaned, zone_id = cleaned.split('%', 1)

    # Handle CIDR notation
    cidr = None
    if '/' in cleaned:
        try:
            network = ipaddress.IPv6Network(cleaned, strict=False)
            result['is_valid'] = True
            result['normalized'] = str(network)
            result['type'] = 'cidr'
            return result
        except ValueError:
            return result

    try:
        addr = ipaddress.IPv6Address(cleaned)
        result['is_valid'] = True
        result['normalized'] = str(addr)
        result['type'] = 'global' if addr.is_global else \
                         'loopback' if addr.is_loopback else \
                         'link_local' if addr.is_link_local else 'other'
        if zone_id:
            result['zone_id'] = zone_id
    except ValueError:
        pass

    return result

# Test cases

inputs = [
    '2001:db8::1',
    '  ::1  ',           # With whitespace
    '[2001:db8::1]',     # URL notation
    'fe80::1%eth0',      # Zone ID
    '2001:db8::/32',     # CIDR
    '192.168.1.1',       # IPv4 (invalid)
    'not-valid',
]

for inp in inputs:
    r = validate_ipv6_input(inp)
    status = 'VALID' if r['is_valid'] else 'INVALID'
    print(f"{inp!r:30s} → {status} {r.get('normalized', '')}")
```

## Validation in JavaScript/TypeScript

```typescript
/**
 * Validate an IPv6 address from user input.
 * Handles brackets, zone IDs, and CIDR notation.
 */
function validateIPv6Input(input: string): {
  isValid: boolean;
  normalized: string | null;
  error: string | null;
} {
  let cleaned = input.trim();

  // Remove URL brackets
  if (cleaned.startsWith('[')) {
    const bracketEnd = cleaned.indexOf(']');
    if (bracketEnd === -1) {
      return { isValid: false, normalized: null, error: 'Unmatched bracket' };
    }
    cleaned = cleaned.substring(1, bracketEnd);
  }

  // Strip zone ID
  if (cleaned.includes('%')) {
    cleaned = cleaned.split('%')[0];
  }

  // Strip CIDR suffix for basic address validation
  let prefix = '';
  if (cleaned.includes('/')) {
    const parts = cleaned.split('/');
    cleaned = parts[0];
    prefix = '/' + parts[1];
    const prefixNum = parseInt(parts[1], 10);
    if (isNaN(prefixNum) || prefixNum < 0 || prefixNum > 128) {
      return { isValid: false, normalized: null, error: 'Invalid prefix length' };
    }
  }

  // Validate using a regex (comprehensive but use library in production)
  const ipv6Regex = /^(([0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}|([0-9a-fA-F]{1,4}:){1,7}:|([0-9a-fA-F]{1,4}:){1,6}:[0-9a-fA-F]{1,4}|([0-9a-fA-F]{1,4}:){1,5}(:[0-9a-fA-F]{1,4}){1,2}|([0-9a-fA-F]{1,4}:){1,4}(:[0-9a-fA-F]{1,4}){1,3}|([0-9a-fA-F]{1,4}:){1,3}(:[0-9a-fA-F]{1,4}){1,4}|([0-9a-fA-F]{1,4}:){1,2}(:[0-9a-fA-F]{1,4}){1,5}|[0-9a-fA-F]{1,4}:((:[0-9a-fA-F]{1,4}){1,6})|:((:[0-9a-fA-F]{1,4}){1,7}|:)|fe80:(:[0-9a-fA-F]{0,4}){0,4}%[0-9a-zA-Z]{1,}|::(ffff(:0{1,4}){0,1}:){0,1}((25[0-5]|(2[0-4]|1{0,1}[0-9]){0,1}[0-9])\.){3,3}(25[0-5]|(2[0-4]|1{0,1}[0-9]){0,1}[0-9])|([0-9a-fA-F]{1,4}:){1,4}:((25[0-5]|(2[0-4]|1{0,1}[0-9]){0,1}[0-9])\.){3,3}(25[0-5]|(2[0-4]|1{0,1}[0-9]){0,1}[0-9]))$/;

  if (!ipv6Regex.test(cleaned)) {
    return { isValid: false, normalized: null, error: 'Invalid IPv6 format' };
  }

  return { isValid: true, normalized: cleaned + prefix, error: null };
}
```

## Sanitization Best Practices

```python
# Key sanitization steps before processing user IPv6 input:

def sanitize_ipv6(raw_input: str) -> str | None:
    """Return normalized IPv6 or None if invalid."""
    if not raw_input or len(raw_input) > 50:  # Max IPv6 string: 45 chars
        return None

    cleaned = raw_input.strip()
    # Remove brackets
    if cleaned.startswith('[') and cleaned.endswith(']'):
        cleaned = cleaned[1:-1]
    # Strip zone ID
    cleaned = cleaned.split('%')[0]

    try:
        return str(ipaddress.IPv6Address(cleaned))
    except ValueError:
        return None
```

## Conclusion

IPv6 input validation requires stripping brackets, zone IDs, and handling CIDR notation before parsing. Use language-native IP parsing libraries (`ipaddress` in Python, `net.ParseIP` in Go, `filter_var` in PHP) rather than custom regex patterns. Always normalize addresses to compressed form before storage or comparison to avoid treating equivalent addresses as different.
