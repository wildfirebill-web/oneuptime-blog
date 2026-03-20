# How to Test AAAA Record Resolution with dig

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, dig, AAAA Records, DNS Testing

Description: A comprehensive guide to using the dig command to test IPv6 AAAA record resolution, with practical examples for troubleshooting DNS issues.

## Basic AAAA Query with dig

The most basic way to query for a AAAA record is:

```bash
# Query for AAAA record of a hostname
dig AAAA example.com

# Short output (just the answer)
dig AAAA example.com +short

# Query against a specific DNS server
dig AAAA example.com @8.8.8.8

# Query against a specific server using IPv6 transport
dig AAAA example.com @2001:4860:4860::8888
```

## Understanding dig AAAA Output

```bash
# Full dig output with explanation
dig AAAA google.com

# Output:
# ; <<>> DiG 9.16.1 <<>> AAAA google.com
# ;; global options: +cmd
# ;; Got answer:
# ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
# ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
#
# ;; QUESTION SECTION:
# ;google.com.            IN  AAAA
#
# ;; ANSWER SECTION:                     ← This is the IPv6 address
# google.com.    55  IN  AAAA  2607:f8b0:4004:c08::65
#
# ;; Query time: 10 msec
# ;; SERVER: 8.8.8.8#53(8.8.8.8)        ← DNS server used
# ;; WHEN: Thu Mar 20 10:00:00 UTC 2026
# ;; MSG SIZE rcvd: 67
```

## Common dig AAAA Queries for Troubleshooting

```bash
# Check if a domain has any AAAA records
dig AAAA www.example.com +short
# Empty output = no AAAA record

# Check AAAA record TTL (useful during TTL changes)
dig AAAA www.example.com | grep AAAA
# www.example.com. 3600 IN AAAA 2001:db8::1
#                  ^^^^--- TTL in seconds

# Check authoritative answer (not from cache)
dig AAAA www.example.com +norecurse @ns1.example.com

# Check AAAA with DNSSEC validation info
dig AAAA www.example.com +dnssec

# Check NXDOMAIN vs NODATA for AAAA
# NXDOMAIN: domain doesn't exist at all
# NODATA: domain exists, but no AAAA record
dig AAAA no-aaaa.example.com
# status: NOERROR with empty ANSWER = NODATA (has A but no AAAA)
# status: NXDOMAIN = domain doesn't exist
```

## Testing AAAA Records Against Multiple DNS Servers

```bash
# Compare AAAA responses from different resolvers
for DNS in 8.8.8.8 1.1.1.1 9.9.9.9 2001:4860:4860::8888; do
    echo -n "$DNS: "
    dig AAAA example.com @$DNS +short
done
```

## Verifying DNS64 Synthesis

When using DNS64+NAT64, verify that synthesized AAAA records start with the NAT64 prefix:

```bash
# Query via DNS64 resolver for a domain with only A records
dig AAAA example.com @dns64-resolver-ip +short
# Expected: 64:ff9b::5db8:d822 (NAT64 prefix + embedded IPv4)

# Verify the prefix is correct
dig AAAA example.com @dns64-resolver-ip +short | grep -c "^64:ff9b::"
# Should return: 1 (synthesis working)

# Compare with native resolver (should return empty for IPv4-only domain)
dig AAAA example.com @8.8.8.8 +short
# Should return: (empty)
```

## Testing Reverse DNS (PTR) for IPv6 Addresses

```bash
# Test reverse lookup for an IPv6 address
dig -x 2001:db8::1 +short
# Expected: server1.example.com.

# Without +short to see full response
dig -x 2001:db8::1

# Test against specific server
dig -x 2001:db8::1 @127.0.0.1
```

## Checking Response Time and Caching

```bash
# Check query time (first query vs cached)
dig AAAA google.com | grep "Query time"
# First query: 50 msec

# Second query (should hit cache)
dig AAAA google.com | grep "Query time"
# Cached: 1 msec

# Force non-cached query
dig AAAA google.com +cd  # disable DNSSEC cache
dig AAAA google.com +nocache  # some versions support this
```

## Batch Testing Multiple Domains

```bash
#!/bin/bash
# Test AAAA records for a list of domains
# Usage: ./test-aaaa.sh domains.txt

DOMAINS_FILE=$1
DNS_SERVER=${2:-8.8.8.8}

echo "Testing AAAA records against $DNS_SERVER"
echo "---"

while read DOMAIN; do
    RESULT=$(dig AAAA $DOMAIN @$DNS_SERVER +short | head -1)
    if [ -z "$RESULT" ]; then
        echo "MISSING: $DOMAIN (no AAAA record)"
    else
        echo "OK: $DOMAIN → $RESULT"
    fi
done < "$DOMAINS_FILE"
```

## Testing Over IPv6 Transport

```bash
# Force dig to use IPv6 transport for the query (not just query type)
dig -6 AAAA example.com @2001:4860:4860::8888

# This sends the DNS query over IPv6 UDP
# Verify with: dig -6 A example.com @2001:4860:4860::8888
# SERVER shows IPv6 address used
```

## Summary

`dig AAAA <hostname>` is the primary tool for testing IPv6 DNS records. Use `+short` for concise output, `@<server>` to test specific resolvers, `-x <ipv6-addr>` for reverse lookups, and loop through multiple DNS servers to compare responses. For DNS64 verification, check that synthesized addresses start with the NAT64 prefix (e.g., `64:ff9b::`). Always check both the presence of AAAA records and the NODATA/NXDOMAIN status when debugging missing IPv6 resolution.
