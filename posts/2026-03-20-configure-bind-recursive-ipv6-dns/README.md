# How to Configure BIND as a Recursive IPv6 DNS Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BIND, DNS, IPv6, Recursive, Resolver, named, DNSSEC

Description: Configure BIND9 as a recursive (caching) DNS resolver that communicates over IPv6 transport, with proper forwarders, ACLs, and DNSSEC validation.

## Introduction

A recursive BIND resolver forwards queries to authoritative servers and caches results. Enabling IPv6 means it can receive queries from IPv6 clients and use IPv6 transport to reach upstream nameservers.

## Step 1: Basic Recursive Configuration

```nginx
# /etc/bind/named.conf.options

options {
    directory "/var/cache/bind";

    # Listen for queries from IPv6 and IPv4 clients
    listen-on-v6 { any; };
    listen-on    { 127.0.0.1; };

    # Enable recursion
    recursion yes;

    # Allow IPv6 and IPv4 local clients
    allow-query {
        ::1;
        fe80::/10;
        2001:db8::/32;
        127.0.0.0/8;
        192.168.0.0/16;
        10.0.0.0/8;
    };

    # DNSSEC validation
    dnssec-validation auto;

    # Use IPv6 for outbound queries when possible
    # (BIND prefers IPv6 when the OS resolves the nameserver via AAAA)
};
```

## Step 2: Configure Forwarders (Optional)

```nginx
options {
    # Forward to dual-stack resolvers
    forwarders {
        2606:4700:4700::1111;  # Cloudflare IPv6
        2606:4700:4700::1001;
        8.8.8.8;               # Google IPv4 fallback
        8.8.4.4;
    };

    # forward only — don't do full recursion if forwarders fail
    forward first;  # Try forwarders first, fall back to recursion
};
```

## Step 3: Prefer IPv6 for Outbound

```nginx
# /etc/bind/named.conf.options

options {
    # Prefer IPv6 addresses when resolving NS records
    # (enabled by default if IPv6 is available)

    # Explicitly set source address for outbound IPv6 queries
    query-source-v6 address 2001:db8::53;

    # Set outbound interface for queries
    # (not usually needed — let the OS route)
};
```

## Step 4: Rate Limiting

```nginx
options {
    # Limit response rate (DNS amplification protection)
    rate-limit {
        responses-per-second 20;
        window 5;
        slip 2;
    };
};
```

## Step 5: Validate Configuration

```bash
named-checkconf

# Restart and verify
systemctl restart bind9
systemctl status bind9

# Query the recursive resolver over IPv6
dig AAAA google.com @::1
dig A example.com @::1

# Check that DNSSEC validation works
dig +dnssec AAAA cloudflare.com @::1

# Confirm it's listening on IPv6
ss -lnp | grep ":53"
# udp  UNCONN  0  0  [::]:53  [::]:*  users:(("named",...))
```

## Step 6: Logging

```nginx
# /etc/bind/named.conf

logging {
    channel default_log {
        file "/var/log/named/default.log" versions 3 size 10m;
        severity info;
        print-time yes;
        print-severity yes;
    };

    channel query_log {
        file "/var/log/named/queries.log" versions 5 size 50m;
        severity info;
        print-time yes;
    };

    category default    { default_log; };
    category queries    { query_log; };
    category resolver   { default_log; };
};
```

## Testing

```bash
# Test recursion with a known AAAA record
dig AAAA ipv6.google.com @::1 +stats
# ;; Query time: 12 msec
# ;; SERVER: ::1#53(::1)

# Test reverse lookup
dig -x 2001:4860:4860::8888 @::1

# Verify DNSSEC chain of trust
dig +cd +dnssec AAAA www.dnssec-failed.org @::1
# AD flag should not appear for failed zones
```

## Conclusion

BIND recursive resolver with IPv6 requires `listen-on-v6 { any; }`, appropriate `allow-query` ACLs for IPv6 subnets, and optional forwarders with IPv6 addresses. Enable DNSSEC validation to authenticate responses. Monitor resolver latency and cache hit rate with OneUptime.
