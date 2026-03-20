# How to Configure DNS64 with BIND

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS64, BIND, NAT64, DNS Configuration

Description: Step-by-step instructions for enabling DNS64 in BIND (named) to synthesize AAAA records for IPv4-only domains and support IPv6-only clients through a NAT64 gateway.

## Prerequisites

- BIND 9.8 or later (DNS64 support was introduced in BIND 9.8)
- A working NAT64 gateway configured with the NAT64 prefix
- The NAT64 prefix to use (e.g., `64:ff9b::/96` or your own)

## Understanding BIND's dns64 Statement

BIND implements DNS64 natively via the `dns64` configuration block in `named.conf`. When a AAAA query arrives and no native AAAA record exists, BIND automatically synthesizes one using the configured prefix.

## Basic DNS64 Configuration

Add the `dns64` block to your `named.conf` or `named.conf.options` file:

```named
// /etc/named.conf or /etc/bind/named.conf.options

options {
    // Listen on IPv6 as well as IPv4
    listen-on-v6 { any; };

    // Allow queries from IPv6-only clients on your network
    allow-query { 192.168.0.0/16; 2001:db8::/32; };

    // Enable DNS64 with the well-known NAT64 prefix
    dns64 64:ff9b::/96 {
        // clients: which source addresses should receive synthesized records
        // Use "any" for all clients, or restrict to specific IPv6 prefixes
        clients { any; };

        // mapped: which IPv4 addresses to synthesize records for
        // "any" means synthesize for all IPv4 addresses
        mapped { any; };

        // exclude: IPv4 addresses to never synthesize records for
        // RFC 1918 addresses are often excluded
        exclude { 10.0.0.0/8; 172.16.0.0/12; 192.168.0.0/16; };

        // suffix: optional IPv6 suffix to append (rarely needed)
        // suffix ::;

        // recursive-only: only synthesize for recursive queries (default yes)
        recursive-only yes;
    };
};
```

## Restricting DNS64 to Specific Clients

If you want only IPv6-only clients to receive synthesized records (dual-stack clients should use real AAAA records):

```named
dns64 64:ff9b::/96 {
    // Only synthesize records for clients on the IPv6-only subnet
    clients { 2001:db8:ipv6only::/48; };

    mapped { any; };

    // Never synthesize for RFC 1918 IPv4 addresses
    exclude {
        10.0.0.0/8;
        172.16.0.0/12;
        192.168.0.0/16;
        127.0.0.0/8;
        ::1/128;
    };
};
```

## Multiple DNS64 Prefixes

BIND supports multiple `dns64` blocks, useful when different NAT64 gateways serve different client groups:

```named
// First NAT64 prefix for internal IPv6-only subnet
dns64 2001:db8:nat64a::/96 {
    clients { 2001:db8:subnet-a::/48; };
    mapped { any; };
    exclude { 192.168.0.0/16; };
};

// Second NAT64 prefix for a different subnet
dns64 2001:db8:nat64b::/96 {
    clients { 2001:db8:subnet-b::/48; };
    mapped { any; };
    exclude { 192.168.0.0/16; };
};
```

## Reloading BIND After Configuration Changes

```bash
# Check configuration syntax before reloading

named-checkconf /etc/named.conf

# Reload BIND without restarting
rndc reload

# Or restart the service
systemctl restart named
# or on Debian/Ubuntu
systemctl restart bind9
```

## Testing DNS64 Resolution

```bash
# Query a domain that has only an A record
# Should return a synthesized AAAA in the 64:ff9b::/96 range
dig AAAA example.com @127.0.0.1

# Expected output for example.com (IPv4: 93.184.216.34)
# ;; ANSWER SECTION:
# example.com.  60  IN  AAAA  64:ff9b::5db8:d822

# Verify that real AAAA records are returned unchanged
dig AAAA ipv6.google.com @127.0.0.1
```

## Verifying DNS64 Does Not Interfere with DNSSEC

When DNSSEC validation is enabled, DNS64 synthesis is skipped for DNSSEC-signed responses. This is the correct behavior:

```bash
# This should NOT return a synthesized record when DNSSEC is enabled
# and the domain has a signed A record with no AAAA
dig AAAA dnssec-protected-domain.example @127.0.0.1 +dnssec
```

## Logging DNS64 Activity

Enable query logging in BIND to see DNS64 activity:

```named
logging {
    channel dns64_log {
        file "/var/log/named/dns64.log" versions 3 size 5m;
        severity dynamic;
        print-time yes;
    };
    category queries { dns64_log; };
};
```

## Summary

Configuring DNS64 in BIND requires adding a `dns64` block to `named.conf` specifying the NAT64 prefix, the clients that should receive synthesized records, and any IPv4 exclusions. After reloading BIND, test with `dig AAAA` against a domain that only has A records. Make sure the NAT64 gateway uses the same prefix configured in BIND.
