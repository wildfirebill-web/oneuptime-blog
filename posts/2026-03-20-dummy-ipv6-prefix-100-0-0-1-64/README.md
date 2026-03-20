# How to Understand the Dummy IPv6 Prefix (100:0:0:1::/64)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Dummy Prefix, 100::/64, Networking, Testing, Special-Purpose

Description: Understand the 100:0:0:1::/64 dummy IPv6 prefix, its relationship to the discard-only 100::/64 block, and appropriate testing use cases.

## Introduction

The `100:0:0:1::/64` prefix is sometimes used as a "dummy" or placeholder IPv6 address in configurations and tests. It falls within the broader `100::/64` discard-only block (RFC 6666). Understanding its proper use prevents unintended behavior.

## Address Block Context

```python
import ipaddress

# 100:0:0:1::/64 is within the discard-only 100::/64
discard_block = ipaddress.IPv6Network("100::/64")
dummy_prefix = ipaddress.IPv6Network("100:0:0:1::/64")

# Check if dummy prefix is within discard block
print(dummy_prefix.subnet_of(discard_block))  # False — different /64
# 100::/64 and 100:0:0:1::/64 are different /64 subnets within 100::/56

# Correctly checking the discard block
addr = ipaddress.IPv6Address("100:0:0:1::1")
print(addr in discard_block)  # False (100::/64 ends at 100::ffff:ffff:ffff:ffff)

# 100:: block is only the /64 starting at 100::
print(ipaddress.IPv6Address("100::1") in discard_block)  # True
print(ipaddress.IPv6Address("100:0:0:1::1") in discard_block)  # False
```

## Common Use Cases for Dummy IPv6 Addresses

### Testing Timeout Behavior

```bash
# Use a non-routable address to test connection timeout
# (Traffic gets dropped, not rejected)
curl --connect-timeout 5 http://[100::1]/test
# Result: connection timeout after 5 seconds

# Test application fallback when IPv6 is unavailable
curl --connect-timeout 3 http://[100::1]/ || \
  curl http://fallback.example.com/
```

### Placeholder in Configuration Templates

```nginx
# Use dummy IPv6 in config templates before real IP is known
upstream backend {
    server [100:0:0:1::1]:8080 down;  # Placeholder — marked down
    server [2001:db8::10]:8080;        # Real server
}
```

### Black-Hole Routing Tests

```bash
# Test routing with a known discard address
ip -6 route add blackhole 100::/64

# Verify traffic to 100:: is discarded
ping6 -c 3 100::1
# Should fail (no route or ICMP unreachable)

# Cleanup
ip -6 route del blackhole 100::/64
```

## Proper Alternatives for Documentation

For documentation and examples, always prefer `2001:db8::/32`:

```
# WRONG for documentation:
ip -6 addr add 100:0:0:1::1/64 dev lo

# CORRECT for documentation:
ip -6 addr add 2001:db8::1/64 dev lo  # Use documentation space
```

## Conclusion

The `100:0:0:1::/64` address is sometimes used informally as a dummy IPv6 address. Understand it falls near (but not within) the `100::/64` discard-only block. For documentation use `2001:db8::/32`, for testing black-holes use `100::/64`. Monitor your test infrastructure and ensure dummy addresses don't accidentally appear in production configs tracked by OneUptime.
