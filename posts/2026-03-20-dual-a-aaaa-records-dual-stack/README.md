# How to Set Up Dual A and AAAA Records for Dual-Stack Domains

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, Dual-Stack, AAAA Records, A Records

Description: A guide to correctly setting up both A and AAAA records for dual-stack services so that IPv4 and IPv6 clients both get the best connection path.

## Why Both A and AAAA Records Matter

Dual-stack means a service is reachable over both IPv4 and IPv6. For clients to use both, the DNS must return both A and AAAA records for the same hostname. A missing AAAA record forces IPv6-capable clients to fall back to IPv4, wasting IPv6 capacity. A missing A record breaks connectivity for IPv4-only clients.

## Correct Dual-Stack DNS Configuration

Both A and AAAA records should coexist for the same hostname with matching TTLs:

```dns
; /var/named/example.com.zone

; Dual-stack records for web service
www     3600    IN  A       93.184.216.34
www     3600    IN  AAAA    2001:db8::1

; Dual-stack records for mail
mail    3600    IN  A       93.184.216.100
mail    3600    IN  AAAA    2001:db8::100

; Zone apex dual-stack
@       3600    IN  A       93.184.216.34
@       3600    IN  AAAA    2001:db8::1

; Name server glue records (dual-stack NS)
ns1     3600    IN  A       93.184.216.200
ns1     3600    IN  AAAA    2001:db8::200
```

## Verifying Dual-Stack Records Exist

```bash
# Check both record types exist for the same hostname
dig A www.example.com +short
dig AAAA www.example.com +short

# Alternatively, check both with a single ANY query
dig ANY www.example.com

# Script to audit all hostnames for dual-stack completeness
for HOST in www mail api ftp; do
    A=$(dig A $HOST.example.com +short)
    AAAA=$(dig AAAA $HOST.example.com +short)
    if [ -z "$A" ]; then
        echo "WARNING: No A record for $HOST.example.com"
    fi
    if [ -z "$AAAA" ]; then
        echo "WARNING: No AAAA record for $HOST.example.com"
    fi
    if [ -n "$A" ] && [ -n "$AAAA" ]; then
        echo "OK: $HOST.example.com - A=$A AAAA=$AAAA"
    fi
done
```

## TTL Consistency Between A and AAAA

Both records should have the same TTL. Mismatched TTLs cause inconsistent behavior when records are cached and refreshed at different times:

```dns
; CORRECT: Matching TTLs
www     3600    IN  A       93.184.216.34
www     3600    IN  AAAA    2001:db8::1

; INCORRECT: Mismatched TTLs
www     300     IN  A       93.184.216.34
www     86400   IN  AAAA    2001:db8::1
```

## How Happy Eyeballs Uses Dual-Stack Records

Modern clients use the Happy Eyeballs algorithm (RFC 8305) when both A and AAAA records exist. The algorithm:

1. Issues AAAA and A queries simultaneously (or slightly offset)
2. Prefers the AAAA response if it arrives first and connects successfully
3. Falls back to A records if IPv6 connectivity fails within ~250ms
4. Result: Fastest connection wins, users never notice the difference

This means dual-stack records give the performance benefits of IPv6 while maintaining IPv4 fallback safety.

## Handling Load-Balanced Services

For load-balanced services with multiple backends, add multiple records for each protocol:

```dns
; IPv4 load balancing with multiple A records
www     300     IN  A       203.0.113.1
www     300     IN  A       203.0.113.2
www     300     IN  A       203.0.113.3

; IPv6 load balancing with multiple AAAA records
www     300     IN  AAAA    2001:db8::1
www     300     IN  AAAA    2001:db8::2
www     300     IN  AAAA    2001:db8::3
```

Use a short TTL (300 seconds) for load-balanced records to allow quick updates.

## Using CNAME for Simplified Dual-Stack Management

When many hostnames point to the same service, use CNAMEs to simplify updates:

```dns
; Central dual-stack record
web-cluster     3600    IN  A       203.0.113.1
web-cluster     3600    IN  AAAA    2001:db8::1

; CNAME records all pointing to the central record
www             3600    IN  CNAME   web-cluster.example.com.
api             3600    IN  CNAME   web-cluster.example.com.
app             3600    IN  CNAME   web-cluster.example.com.
```

Note: CNAMEs cannot be used at the zone apex (@). Use A/AAAA directly there.

## Testing Dual-Stack Connectivity

After setting up dual-stack DNS, verify both paths work:

```bash
# Test IPv4 connectivity via A record
curl -4 https://www.example.com/

# Test IPv6 connectivity via AAAA record
curl -6 https://www.example.com/

# Test that Happy Eyeballs picks IPv6 when available
curl -v https://www.example.com/ 2>&1 | grep "Connected to"
# Should show an IPv6 address if available
```

## Summary

Proper dual-stack DNS requires both A and AAAA records with matching TTLs for every hostname. Audit your zones to find hostnames missing either record type. With both records present, modern clients use Happy Eyeballs to automatically prefer IPv6 while falling back to IPv4 when needed — providing the best experience for all clients without any additional configuration.
