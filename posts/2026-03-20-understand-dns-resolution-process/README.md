# How to Understand DNS Resolution Process Step by Step

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Resolution, Networking, Linux, Protocol, Troubleshooting

Description: Trace the complete DNS resolution process from application query through stub resolver, recursive resolver, root servers, TLD servers, and authoritative servers.

## Introduction

DNS resolution converts a hostname like `api.example.com` into an IP address through a series of delegated queries. Understanding each step enables you to diagnose exactly where resolution fails, why responses are slow, and how caching affects results. The process involves multiple components: the application's stub resolver, a recursive resolver, root nameservers, TLD nameservers, and authoritative nameservers.

## The Resolution Chain

```text
Application queries: "What is the IP for api.example.com?"

Step 1: Check local cache
  - Application DNS cache (JVM, browser)
  - OS stub resolver cache (/etc/hosts, nscd, systemd-resolved)

Step 2: Stub resolver asks recursive resolver (e.g., 8.8.8.8)
  - Configured in /etc/resolv.conf or systemd-resolved

Step 3: Recursive resolver checks its cache
  - If cached and not expired: return immediately

Step 4: If not cached, recursive resolver queries root servers
  - 13 root server sets (a.root-servers.net through m.root-servers.net)
  - Asks: "Who knows about .com?"
  - Root server says: "Ask g.gtld-servers.net for .com"

Step 5: Query .com TLD servers
  - Asks: "Who knows about example.com?"
  - TLD server says: "Ask ns1.example.com for example.com"

Step 6: Query example.com authoritative nameserver
  - Asks: "What is the A record for api.example.com?"
  - Returns: 93.184.216.34

Step 7: Recursive resolver caches result (per TTL) and returns to client
  - Client stub resolver caches result
  - Application gets IP address
```

## Trace Resolution with dig +trace

```bash
# Follow the complete delegation chain:

dig +trace api.example.com

# Output shows each step:
# . (root) → .com (TLD) → example.com (auth) → final answer
# Each step shows which nameserver was queried and what was returned

# Important details in +trace output:
# [query] is the question asked
# QUERY TIME: how long each step took
# SERVER: which nameserver answered each step
```

## Check Each Resolution Component

```bash
# 1. Check /etc/hosts (first lookup):
grep api.example.com /etc/hosts

# 2. Check local resolver:
cat /etc/resolv.conf
# Shows: nameserver IP (e.g., 127.0.0.53 for systemd-resolved, or 8.8.8.8)

# 3. Query the recursive resolver directly:
dig @8.8.8.8 api.example.com

# 4. Query the authoritative nameserver directly (bypass cache):
# First find the authoritative server:
dig NS example.com
# Then query it directly:
dig @ns1.example.com api.example.com

# 5. Check if result is cached (AA flag):
dig api.example.com
# "aa" flag in flags = answer from authoritative server
# No "aa" flag = cached answer from recursive resolver
```

## Measure Resolution Time at Each Step

```bash
# Total resolution time (includes all steps):
time dig api.example.com

# Recursive resolver response time:
dig @8.8.8.8 api.example.com | grep "Query time"

# Authoritative server response time (bypasses cache):
AUTH_NS=$(dig NS example.com +short | head -1)
dig @$AUTH_NS api.example.com | grep "Query time"

# Simulated full resolution (dig +trace shows timing per step):
dig +trace +stats api.example.com 2>&1 | grep -E "Query time|SERVER|msec"
```

## Common Resolution Failures

```bash
# Failure: SERVFAIL (resolver error)
dig api.example.com
# status: SERVFAIL → recursive resolver couldn't get answer
# Could be: authoritative server down, DNSSEC validation failure, network issue

# Failure: NXDOMAIN (domain doesn't exist)
dig nonexistent.example.com
# status: NXDOMAIN → authoritative server says domain doesn't exist
# Check: typo in hostname? DNS record not created?

# Failure: REFUSED (server refused query)
dig @10.20.0.5 api.example.com
# status: REFUSED → resolver doesn't accept queries from your IP
# Check: resolver ACL, allow-query configuration

# Failure: No response (timeout)
dig +time=2 api.example.com
# ;; connection timed out; no servers could be reached
# Check: DNS port 53 blocked? Resolver down?
```

## Conclusion

DNS resolution is a hierarchical delegation chain: root → TLD → authoritative → client. Use `dig +trace` to follow the complete chain and see which step is slow or failing. A cached response bypasses steps 4-6 and is limited by the configured recursive resolver's cache. For troubleshooting, always distinguish between cached responses (from recursive resolver) and authoritative responses (from the domain's nameserver) - TTL expiration forces a fresh authoritative query.
