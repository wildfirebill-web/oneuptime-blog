# How to Configure Recursive DNS Resolvers for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, Recursive Resolver, BIND, Unbound

Description: A guide to configuring recursive DNS resolvers to handle IPv6 queries from clients and resolve AAAA records using IPv6-capable upstream servers.

## What Makes a Recursive Resolver IPv6-Ready?

A recursive DNS resolver needs IPv6 support at two levels:
1. **Client-facing**: Accept DNS queries from IPv6 clients
2. **Upstream-facing**: Query authoritative servers over IPv6 (do-ip6)

Both levels must be configured for full IPv6 functionality.

## Configuring BIND as an IPv6 Recursive Resolver

```named
// /etc/named.conf

options {
    // === Client-facing IPv6 ===
    // Listen on all IPv4 and IPv6 addresses
    listen-on { any; };
    listen-on-v6 { any; };

    // Allow queries from IPv6 clients
    allow-query {
        127.0.0.1; ::1;
        192.168.0.0/16;
        10.0.0.0/8;
        2001:db8::/32;   // Replace with your IPv6 prefix
        fd00::/8;        // ULA addresses
    };

    // Allow recursive queries from your clients
    allow-recursion {
        127.0.0.1; ::1;
        192.168.0.0/16;
        10.0.0.0/8;
        2001:db8::/32;
    };

    // === Upstream-facing IPv6 ===
    // Use IPv6 to query authoritative servers
    // (BIND uses both IPv4 and IPv6 by default when available)

    // Prefer IPv6 for upstream queries (optional)
    // BIND uses preference based on RFC 6724 by default

    // DNSSEC validation (recommended)
    dnssec-validation auto;

    // Root hints (BIND uses built-in or /etc/named.root file)
    recursion yes;
};
```

## Configuring Unbound as an IPv6 Recursive Resolver

```yaml
# /etc/unbound/unbound.conf

server:
    # Client-facing interface configuration
    interface: 0.0.0.0        # IPv4
    interface: ::0            # IPv6

    # Allow queries from IPv6 clients
    access-control: 127.0.0.0/8 allow
    access-control: ::1/128 allow
    access-control: 192.168.0.0/16 allow
    access-control: 10.0.0.0/8 allow
    access-control: 2001:db8::/32 allow  # your IPv6 prefix
    access-control: fd00::/8 allow       # ULA
    access-control: 0.0.0.0/0 refuse
    access-control: ::0/0 refuse

    # IPv6 for upstream queries
    do-ip6: yes
    prefer-ip6: no    # set yes to prefer IPv6 upstreams

    # DNSSEC validation
    auto-trust-anchor-file: "/var/lib/unbound/root.key"

    # Module order (important for DNS64 if used)
    module-config: "validator iterator"

    # Prefetch frequently queried records
    prefetch: yes
    prefetch-key: yes
```

## Configuring Root Hints for IPv6

The root name servers are reachable over IPv6. Ensure your resolver has up-to-date root hints:

```bash
# Download current root hints
wget -O /etc/named.root https://www.internic.net/domain/named.root

# Verify root hints include IPv6 addresses
grep "AAAA" /etc/named.root | head -5
# Should show IPv6 addresses for root servers (A.ROOT-SERVERS.NET, etc.)
```

For BIND, reference the root hints file:

```named
zone "." {
    type hint;
    file "/etc/named.root";
};
```

## Testing the IPv6 Recursive Resolver

```bash
# Test resolving an AAAA record via the local recursive resolver
dig AAAA google.com @::1

# Test that the resolver itself queries over IPv6
# (enable BIND query logging temporarily)
rndc querylog on
dig AAAA ipv6.google.com @127.0.0.1
# Check logs for outbound queries using IPv6 source addresses
tail -f /var/log/named/queries.log | grep -E 'ipv6|AAAA'

# For Unbound: enable verbosity to see upstream query IPs
unbound-control verbosity 3
dig AAAA google.com @127.0.0.1
unbound-control verbosity 1
```

## Forwarding IPv6 Queries to Upstream Resolvers

If your recursive resolver forwards to upstream (not full recursion):

```named
// BIND: forward to IPv6-capable upstream resolvers
options {
    forwarders {
        8.8.8.8;                    // Google IPv4
        8.8.4.4;                    // Google IPv4
        2001:4860:4860::8888;       // Google IPv6
        2001:4860:4860::8844;       // Google IPv6 secondary
    };
    forward only;  // or 'first' to try forwarders first, then recurse
};
```

```yaml
# Unbound: forward to upstream resolvers
forward-zone:
    name: "."
    forward-addr: 8.8.8.8
    forward-addr: 8.8.4.4
    forward-addr: 2001:4860:4860::8888
    forward-addr: 2001:4860:4860::8844
```

## Monitoring Resolver IPv6 Statistics

```bash
# BIND: check query statistics by IP family
rndc stats
grep -E "IPv6|queries" /var/named/data/named_stats.txt

# Unbound: detailed statistics
unbound-control stats | grep -E "num.queries|ip6"
```

## Summary

A fully IPv6-ready recursive resolver listens on `::0` (all IPv6 addresses), configures `access-control` to allow queries from IPv6 clients, enables `do-ip6: yes` for upstream IPv6 queries, and uses up-to-date root hints that include IPv6 addresses. Both BIND and Unbound support these settings. Test with `dig AAAA @::1` to confirm client-facing IPv6 and check logs to verify upstream queries go over IPv6.
