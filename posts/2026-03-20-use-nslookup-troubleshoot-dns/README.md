# How to Use nslookup to Troubleshoot DNS Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, nslookup, Troubleshooting, Linux, Windows, Networking

Description: Use nslookup to query DNS records, test different resolvers, perform reverse lookups, and diagnose DNS resolution problems on Linux and Windows.

## Introduction

`nslookup` is one of the oldest DNS query tools, available on virtually every operating system including Windows, macOS, and Linux. While `dig` provides more detailed output, `nslookup` is simpler to use and available everywhere. It is useful for quick checks, resolving from specific servers, and testing DNS across platforms with the same commands.

## Basic nslookup Usage

```bash
# Simple lookup (uses system resolver):

nslookup example.com

# Output:
# Server:    8.8.8.8        ← which resolver was used
# Address:   8.8.8.8#53
# Non-authoritative answer:  ← means answer came from cache
# Name:      example.com
# Address:   93.184.216.34

# Query specific record type:
nslookup -type=MX example.com      # Mail records
nslookup -type=NS example.com      # Nameservers
nslookup -type=TXT example.com     # TXT records
nslookup -type=AAAA example.com    # IPv6 addresses
nslookup -type=SOA example.com     # Start of Authority
nslookup -type=ANY example.com     # All record types

# Reverse lookup:
nslookup 93.184.216.34
```

## Query Specific DNS Server

```bash
# Query a specific server (syntax: nslookup domain server):
nslookup example.com 8.8.8.8          # Use Google DNS
nslookup example.com 1.1.1.1          # Use Cloudflare DNS
nslookup example.com ns1.example.com  # Use authoritative server directly

# Test internal DNS:
nslookup internal-server.company.local 10.20.0.10
```

## Interactive Mode

```bash
# Enter interactive mode:
nslookup

# Inside interactive mode:
> server 8.8.8.8        # Change resolver
> set type=MX           # Change query type
> example.com           # Query
> set type=NS
> example.com
> set debug             # Show verbose output
> example.com
> exit
```

## Common Troubleshooting Patterns

```bash
# Check if domain exists:
nslookup nonexistent.example.com
# "** server can't find nonexistent.example.com: NXDOMAIN"
# NXDOMAIN = domain does not exist

# Check if it's a caching or auth issue:
# Query authoritative server (bypasses cache):
AUTH_NS=$(nslookup -type=NS example.com | grep "nameserver" | head -1 | awk '{print $NF}')
echo "Authoritative server: $AUTH_NS"
nslookup example.com $AUTH_NS

# Compare resolver vs authoritative (split-horizon check):
echo "System resolver:"
nslookup example.com

echo "Google Public DNS:"
nslookup example.com 8.8.8.8

echo "Cloudflare DNS:"
nslookup example.com 1.1.1.1
# Different answers: split-horizon or propagation issue

# Test DNS server accessibility:
nslookup google.com 10.20.0.1
# If fails: DNS server 10.20.0.1 is unreachable or not responding
```

## Windows-Specific nslookup

```powershell
# Windows command prompt:
nslookup example.com
nslookup -type=MX example.com

# Set timeout (Windows):
nslookup -timeout=5 example.com

# Set retry count:
nslookup -retry=3 example.com

# Verbose debug output:
nslookup -debug example.com
```

## nslookup vs dig Comparison

```bash
# Feature comparison:

# nslookup: simple, available everywhere, no TTL shown by default
nslookup example.com

# dig: more detail, TTL shown, scriptable output
dig example.com +short

# When to use nslookup:
# - Quick check from Windows (dig not natively available)
# - Cross-platform consistency (same syntax on all OS)
# - Interactive exploration with changing servers/types

# When to use dig instead:
# - Need TTL values
# - Scripting/automation (dig +short, dig -f batch_file)
# - +trace for delegation chain
# - DNSSEC validation checks (+dnssec flag)
```

## Conclusion

`nslookup` provides quick DNS queries available on all platforms. The most useful patterns are: querying a specific server (`nslookup domain server`), changing record type with `-type=`, and reverse lookups with IP addresses. For systematic DNS troubleshooting, compare results from your system resolver versus public DNS servers to identify split-horizon configurations. On Windows where `dig` is not available, `nslookup -debug` provides verbose output similar to `dig`.
