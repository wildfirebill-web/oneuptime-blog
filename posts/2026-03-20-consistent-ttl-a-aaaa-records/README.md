# How to Understand Consistent TTL Values for A and AAAA Records

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, TTL, AAAA Records, DNS Best Practices

Description: An explanation of why A and AAAA records for the same hostname should have consistent TTL values and how mismatched TTLs cause real-world connectivity issues.

## What Is TTL and Why Does It Matter?

TTL (Time To Live) is the number of seconds a DNS resolver is allowed to cache a record before querying again. When A and AAAA records for the same hostname have different TTLs, they expire from resolver caches at different times, leading to temporary states where only one record type is available.

## The Problem with Mismatched TTLs

Consider a hostname with these records:

```dns
www.example.com.  300   IN  A     203.0.113.1   ; expires in 5 minutes
www.example.com.  86400 IN  AAAA  2001:db8::1   ; expires in 24 hours
```

If you update the A record (e.g., during a failover):

1. Old A record (`203.0.113.1`) expires from caches after 5 minutes - clients see the new IPv4
2. But AAAA record (`2001:db8::1`) stays cached for up to 24 hours

If your AAAA backend also needs updating, IPv6 clients are stuck with a stale address for nearly a day. Conversely, if the AAAA record is updated first, IPv4 and IPv6 clients see different server versions for hours.

## Recommended TTL Strategy

Use **identical TTL values** for A and AAAA records of the same hostname. The choice of TTL depends on how frequently you expect to update the records:

| Use Case | Recommended TTL |
|---|---|
| Stable production services | 3600s (1 hour) |
| Services with frequent changes | 300s (5 minutes) |
| CDN / anycast services | 60–300s |
| Active migration / failover | 60–300s |
| Long-term stable infrastructure | 86400s (24 hours) |

## Correct Configuration Example

```dns
; CORRECT: Matching TTLs for A and AAAA
; Both records expire together, preventing inconsistent resolution
www     3600    IN  A       203.0.113.1
www     3600    IN  AAAA    2001:db8::1

mail    3600    IN  A       203.0.113.10
mail    3600    IN  AAAA    2001:db8::10
```

## Auditing TTL Consistency

Use a script to check for TTL mismatches across your zone:

```bash
#!/bin/bash
# Check TTL consistency between A and AAAA records

# Usage: ./check-ttl.sh example.com

ZONE=$1
DNS_SERVER="127.0.0.1"

# Get all hostnames with both A and AAAA records
while read HOSTNAME; do
    A_TTL=$(dig A $HOSTNAME @$DNS_SERVER +noall +answer | awk '{print $2}' | head -1)
    AAAA_TTL=$(dig AAAA $HOSTNAME @$DNS_SERVER +noall +answer | awk '{print $2}' | head -1)

    if [ -n "$A_TTL" ] && [ -n "$AAAA_TTL" ]; then
        if [ "$A_TTL" != "$AAAA_TTL" ]; then
            echo "MISMATCH: $HOSTNAME - A TTL=$A_TTL AAAA TTL=$AAAA_TTL"
        else
            echo "OK: $HOSTNAME - TTL=$A_TTL"
        fi
    fi
done < <(dig AXFR $ZONE @$DNS_SERVER | awk '{print $1}' | sort -u)
```

## Using $TTL for Consistency in Zone Files

The `$TTL` directive in zone files sets a default TTL for all records. Use it to ensure consistency:

```dns
; Set a global default TTL
$TTL 3600

; All records below inherit 3600s TTL unless overridden
@   IN  SOA ns1.example.com. admin.example.com. (2026032001 3600 900 604800 300)
@   IN  NS  ns1.example.com.

; These all use 3600s TTL (from $TTL)
@   IN  A       203.0.113.1
@   IN  AAAA    2001:db8::1
www IN  A       203.0.113.1
www IN  AAAA    2001:db8::1
```

## Lowering TTLs Before Changes

Before making DNS changes (IP address migration, failover), lower TTLs well in advance:

```bash
# Step 1: Lower TTLs 48 hours before the change
# Change zone file TTL from 86400 to 300
# Reload BIND
rndc reload example.com

# Step 2: Wait 48 hours for all caches to expire old long-TTL records
# (worst-case: some resolvers may have cached the 86400s TTL)

# Step 3: Make the IP change
# Update both A and AAAA records simultaneously

# Step 4: After change is confirmed stable, raise TTLs back
# Increase TTL from 300 back to 3600 or 86400
```

## TTL and Monitoring Considerations

Short TTLs increase DNS query load on authoritative servers. Balance change agility with server load:

```bash
# Monitor DNS query rates with BIND statistics
rndc stats
grep "queries resulted" /var/named/data/named_stats.txt

# If query rate is too high due to short TTLs, consider:
# 1. Using a CDN with its own TTL management
# 2. Increasing TTL for stable records
# 3. Adding more authoritative DNS capacity
```

## Summary

Consistent TTL values between A and AAAA records prevent split-brain caching scenarios where IPv4 and IPv6 paths diverge after changes. Use identical TTLs for both record types, lower TTLs in advance of planned changes, and audit your zones periodically for TTL mismatches. The `$TTL` directive in zone files is the simplest way to maintain consistency.
