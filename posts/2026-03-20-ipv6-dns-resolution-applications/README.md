# How to Implement IPv6-Aware DNS Resolution in Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, AAAA Records, Getaddrinfo, Networking, Happy Eyeballs, Programming

Description: Implement IPv6-aware DNS resolution in applications using getaddrinfo, handling AAAA and A record lookups, Happy Eyeballs, and fallback behavior.

## Introduction

IPv6-aware DNS resolution means correctly handling AAAA records alongside A records in application code. Modern applications should implement Happy Eyeballs (RFC 8305) to prefer IPv6 while quickly falling back to IPv4 when IPv6 is unavailable. This guide covers implementing proper DNS resolution across common platforms.

## Using getaddrinfo (C/POSIX)

```c
#include <sys/socket.h>
#include <netdb.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <string.h>

/* Resolve a hostname and print all results */
void resolve_hostname(const char *hostname) {
    struct addrinfo hints, *result, *p;

    memset(&hints, 0, sizeof(hints));
    hints.ai_family   = AF_UNSPEC;        /* Accept IPv4 and IPv6 */
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags    = AI_ADDRCONFIG;    /* Only usable addresses */

    int status = getaddrinfo(hostname, NULL, &hints, &result);
    if (status != 0) {
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(status));
        return;
    }

    printf("Addresses for %s:\n", hostname);
    for (p = result; p != NULL; p = p->ai_next) {
        char ip[INET6_ADDRSTRLEN];
        void *addr;

        if (p->ai_family == AF_INET6) {
            addr = &((struct sockaddr_in6 *)p->ai_addr)->sin6_addr;
            inet_ntop(AF_INET6, addr, ip, sizeof(ip));
            printf("  AAAA: %s\n", ip);
        } else {
            addr = &((struct sockaddr_in *)p->ai_addr)->sin_addr;
            inet_ntop(AF_INET, addr, ip, sizeof(ip));
            printf("  A:    %s\n", ip);
        }
    }

    freeaddrinfo(result);
}

/* Prefer IPv6, then IPv4 */
int connect_prefer_ipv6(const char *hostname, const char *port) {
    struct addrinfo hints, *result, *p;
    int sockfd = -1;
    int found_v6 = 0, found_v4 = 0;

    memset(&hints, 0, sizeof(hints));
    hints.ai_family   = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;

    if (getaddrinfo(hostname, port, &hints, &result) != 0) return -1;

    /* First pass: try IPv6 */
    for (p = result; p != NULL; p = p->ai_next) {
        if (p->ai_family != AF_INET6) continue;
        sockfd = socket(AF_INET6, SOCK_STREAM, 0);
        if (sockfd < 0) continue;
        if (connect(sockfd, p->ai_addr, p->ai_addrlen) == 0) {
            found_v6 = 1;
            break;
        }
        close(sockfd);
        sockfd = -1;
    }

    /* Second pass: fall back to IPv4 */
    if (!found_v6) {
        for (p = result; p != NULL; p = p->ai_next) {
            if (p->ai_family != AF_INET) continue;
            sockfd = socket(AF_INET, SOCK_STREAM, 0);
            if (sockfd < 0) continue;
            if (connect(sockfd, p->ai_addr, p->ai_addrlen) == 0) {
                found_v4 = 1;
                break;
            }
            close(sockfd);
            sockfd = -1;
        }
    }

    freeaddrinfo(result);
    printf("Connected via %s\n", found_v6 ? "IPv6" : found_v4 ? "IPv4" : "none");
    return sockfd;
}
```

## Python: Async DNS Resolution

```python
import asyncio
import socket
import ipaddress
from typing import Optional, Tuple

async def resolve_prefer_ipv6(hostname: str) -> Optional[Tuple[str, int]]:
    """
    Resolve hostname, preferring IPv6 addresses.
    Returns (address, family) tuple or None.
    """
    loop = asyncio.get_event_loop()

    try:
        # Resolve all addresses
        results = await loop.getaddrinfo(
            hostname, None,
            family=socket.AF_UNSPEC,
            type=socket.SOCK_STREAM,
            flags=socket.AI_ADDRCONFIG
        )
    except socket.gaierror as e:
        print(f"DNS resolution failed: {e}")
        return None

    # Sort: IPv6 first, then IPv4
    ipv6_addrs = [(r[4][0], r[0]) for r in results if r[0] == socket.AF_INET6]
    ipv4_addrs = [(r[4][0], r[0]) for r in results if r[0] == socket.AF_INET]

    all_addrs = ipv6_addrs + ipv4_addrs
    if not all_addrs:
        return None

    addr, family = all_addrs[0]
    print(f"Resolved {hostname} to {addr} ({'IPv6' if family == socket.AF_INET6 else 'IPv4'})")
    return addr, family

# Check for AAAA record existence

import dns.resolver

def check_aaaa_record(hostname: str) -> bool:
    """Check if a hostname has an AAAA record."""
    try:
        dns.resolver.resolve(hostname, 'AAAA')
        return True
    except (dns.resolver.NXDOMAIN, dns.resolver.NoAnswer, dns.exception.Timeout):
        return False

async def main():
    result = await resolve_prefer_ipv6('google.com')
    if result:
        print(f"Will connect to: {result[0]}")

asyncio.run(main())
```

## Node.js: DNS AAAA Resolution

```javascript
const dns = require('dns').promises;
const net = require('net');

/**
 * Resolve a hostname with IPv6 preference.
 * Returns the first address, preferring IPv6.
 */
async function resolvePreferIPv6(hostname) {
  try {
    // Try AAAA first
    const ipv6Addrs = await dns.resolve6(hostname);
    if (ipv6Addrs.length > 0) {
      return { address: ipv6Addrs[0], family: 6 };
    }
  } catch (e) {
    // No AAAA records, fall through to IPv4
  }

  try {
    // Fall back to A record
    const ipv4Addrs = await dns.resolve4(hostname);
    if (ipv4Addrs.length > 0) {
      return { address: ipv4Addrs[0], family: 4 };
    }
  } catch (e) {
    // No records at all
  }

  throw new Error(`Could not resolve: ${hostname}`);
}

// Happy Eyeballs-inspired connection
async function connectHappyEyeballs(hostname, port) {
  const [v6Result, v4Result] = await Promise.allSettled([
    dns.resolve6(hostname).then(addrs => ({ address: addrs[0], family: 6 })),
    dns.resolve4(hostname).then(addrs => ({ address: addrs[0], family: 4 }))
  ]);

  // Try IPv6 first, then IPv4 with a 250ms delay
  return new Promise((resolve, reject) => {
    let connected = false;

    const tryConnect = (addrInfo) => {
      if (connected) return;
      const { address } = addrInfo;

      const sock = net.createConnection({ host: address, port, family: addrInfo.family });
      sock.once('connect', () => {
        if (!connected) {
          connected = true;
          resolve(sock);
        }
      });
      sock.once('error', () => {});
    };

    if (v6Result.status === 'fulfilled') tryConnect(v6Result.value);
    setTimeout(() => {
      if (v4Result.status === 'fulfilled') tryConnect(v4Result.value);
    }, 250);  // Happy Eyeballs delay
  });
}
```

## Checking AAAA Records Before Connecting

```bash
# Shell script to check AAAA record before connecting
check_ipv6_ready() {
    local hostname="$1"
    if dig AAAA "$hostname" +short | grep -q ":"; then
        echo "$hostname has AAAA records - IPv6 ready"
        return 0
    else
        echo "$hostname has no AAAA records - IPv4 only"
        return 1
    fi
}

check_ipv6_ready "google.com"
check_ipv6_ready "ipv6.google.com"
```

## Conclusion

IPv6-aware DNS resolution starts with using `getaddrinfo(AF_UNSPEC)` in C or protocol-agnostic resolver APIs in higher-level languages. Prefer IPv6 by sorting results to put AAAA addresses first, then implement a short fallback delay (Happy Eyeballs, ~250ms) before trying IPv4. Always check `AI_ADDRCONFIG` to avoid returning addresses that can't be used on the current host's network configuration.
