# How to Create a Subnetting Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Subnetting, Networking, Reference, Cheat Sheet, Certification

Description: A subnetting cheat sheet consolidates prefix lengths, subnet masks, wildcard masks, block sizes, subnet counts, and host counts into a single quick-reference table for rapid mental subnetting.

## Complete Subnetting Reference Table

| Prefix | Subnet Mask | Wildcard | Block | Subnets from /24 | Hosts |
|--------|-------------|---------|-------|-----------------|-------|
| /24 | 255.255.255.0 | 0.0.0.255 | 256 | 1 | 254 |
| /25 | 255.255.255.128 | 0.0.0.127 | 128 | 2 | 126 |
| /26 | 255.255.255.192 | 0.0.0.63 | 64 | 4 | 62 |
| /27 | 255.255.255.224 | 0.0.0.31 | 32 | 8 | 30 |
| /28 | 255.255.255.240 | 0.0.0.15 | 16 | 16 | 14 |
| /29 | 255.255.255.248 | 0.0.0.7 | 8 | 32 | 6 |
| /30 | 255.255.255.252 | 0.0.0.3 | 4 | 64 | 2 |
| /31 | 255.255.255.254 | 0.0.0.1 | 2 | 128 | 2* |
| /32 | 255.255.255.255 | 0.0.0.0 | 1 | 256 | 1* |

*RFC 3021 (/31) and host routes (/32)

## Generating the Cheat Sheet with Python

```python
import ipaddress

def generate_cheat_sheet(start_prefix: int = 8, end_prefix: int = 32):
    """Generate a subnetting reference table."""
    print(f"{'Pfx':>4} {'Subnet Mask':18} {'Wildcard':16} {'Block':>6} "
          f"{'Total Addrs':>12} {'Usable':>8}")
    print("-" * 72)
    for p in range(start_prefix, end_prefix + 1):
        net = ipaddress.IPv4Network(f"0.0.0.0/{p}")
        block = 2 ** (32 - p)
        usable = max(block - 2, 0) if p < 31 else block
        print(f"/{p:<3} {str(net.netmask):18} {str(net.hostmask):16} "
              f"{block:>6,} {block:>12,} {usable:>8,}")

generate_cheat_sheet(20, 32)
```

## Powers of 2 Reference

```
2^1  = 2        2^9  = 512
2^2  = 4        2^10 = 1,024
2^3  = 8        2^11 = 2,048
2^4  = 16       2^12 = 4,096
2^5  = 32       2^13 = 8,192
2^6  = 64       2^16 = 65,536
2^7  = 128      2^24 = 16,777,216
2^8  = 256
```

## Exam Quick-Answer Rules

1. **Subnets from /N within /8**: 2^(N-8), hosts = 2^(32-N) − 2
2. **Block size**: 256 − last_non_255_octet_of_mask
3. **Subnet**: largest multiple of block ≤ IP octet
4. **Broadcast**: next subnet − 1 (all remaining octets = 255)
5. **/30 P2P**: 2 hosts, /31: 2 hosts no broadcast, /32: 1 host

## Key Takeaways

- Memorize block sizes for /25 through /30: 128, 64, 32, 16, 8, 4.
- The number of subnets doubles and host count halves with each additional borrowed bit.
- Keep this cheat sheet handy for certification exams (CCNA, Network+).
- Verify any mental calculation with `ipaddress.IPv4Network` in Python.
