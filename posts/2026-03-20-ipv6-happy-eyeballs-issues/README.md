# How to Troubleshoot IPv6 Happy Eyeballs Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Happy Eyeballs, Dual-Stack, Troubleshooting, Connection Delays, RFC 8305

Description: Diagnose and fix Happy Eyeballs (RFC 8305) connection delays, unexpected IPv4 fallback, and application behavior when both IPv4 and IPv6 are available.

## Introduction

Happy Eyeballs (RFC 8305) is the algorithm applications use to race IPv4 and IPv6 connection attempts, using whichever responds first. When implemented correctly, users get the best of both protocols. Problems arise when IPv6 connections consistently fail or are slow, causing 250ms+ delays before IPv4 fallback — or when IPv6 is preferred but should fall back to IPv4.

## Understanding Happy Eyeballs

```
Happy Eyeballs flow:
1. DNS query for both A and AAAA records (in parallel)
2. Sort addresses by preference (IPv6 preferred per RFC 6724)
3. Start connection attempt to first address (usually IPv6)
4. If no response in 250ms, start IPv4 attempt in parallel
5. Use whichever connection succeeds first
6. Cancel the slower attempt
```

## Step 1: Diagnose Connection Preference

```bash
# Check which protocol curl actually uses
curl -w "Connected to: %{remote_ip}\n" -s -o /dev/null https://example.com

# Force IPv6 to see if it succeeds
curl -6 -s -o /dev/null https://example.com && echo "IPv6 works"

# Force IPv4 to see if it succeeds
curl -4 -s -o /dev/null https://example.com && echo "IPv4 works"

# Measure time difference between IPv4 and IPv6
echo "IPv6 connection time:"
time curl -6 -s -o /dev/null https://example.com

echo "IPv4 connection time:"
time curl -4 -s -o /dev/null https://example.com
```

## Step 2: Test DNS Response Times for AAAA vs A

```bash
# Measure DNS resolution time for AAAA
time dig AAAA example.com +short

# Measure DNS resolution time for A
time dig A example.com +short

# If AAAA takes significantly longer, this delays Happy Eyeballs
# Some DNS resolvers are slow to return AAAA records

# Check if AAAA queries time out (causing 5-30s delays)
dig AAAA example.com +time=2 +tries=1
echo "Exit code: $? (0=OK, 1=timeout/error)"
```

## Step 3: Simulate Happy Eyeballs Behavior

```python
#!/usr/bin/env python3
"""Simulate Happy Eyeballs connection racing."""

import socket
import time
import asyncio

async def try_connect(host, port, family, delay_ms=0):
    """Try to connect with a specific address family."""
    await asyncio.sleep(delay_ms / 1000)

    try:
        start = time.time()
        info = socket.getaddrinfo(
            host, port, family, socket.SOCK_STREAM
        )
        for res in info:
            af, socktype, proto, canonname, sockaddr = res
            try:
                sock = socket.socket(af, socktype, proto)
                sock.setblocking(False)
                conn_start = time.time()
                await asyncio.wait_for(
                    asyncio.get_event_loop().sock_connect(sock, sockaddr),
                    timeout=5
                )
                elapsed = (time.time() - conn_start) * 1000
                family_name = "IPv6" if af == socket.AF_INET6 else "IPv4"
                print(f"{family_name} connected to {sockaddr[0]} in {elapsed:.0f}ms")
                sock.close()
                return family_name, elapsed
            except Exception as e:
                print(f"Failed: {e}")
    except Exception as e:
        print(f"getaddrinfo failed: {e}")
    return None, None

async def happy_eyeballs(host, port):
    """Race IPv6 and IPv4 connections."""
    print(f"Happy Eyeballs test to {host}:{port}")

    # Start IPv6 first
    ipv6_task = asyncio.create_task(
        try_connect(host, port, socket.AF_INET6, delay_ms=0)
    )
    # Start IPv4 with 250ms delay (RFC 8305)
    ipv4_task = asyncio.create_task(
        try_connect(host, port, socket.AF_INET, delay_ms=250)
    )

    done, pending = await asyncio.wait(
        [ipv6_task, ipv4_task],
        return_when=asyncio.FIRST_COMPLETED
    )

    for task in pending:
        task.cancel()

    for task in done:
        winner, elapsed = task.result()
        if winner:
            print(f"Winner: {winner} ({elapsed:.0f}ms)")

asyncio.run(happy_eyeballs("google.com", 443))
```

## Step 4: Fix Happy Eyeballs Delays in Applications

```bash
# Problem: IPv6 SYN never returns, causing 250ms delay for every connection
# Solution: Fix underlying IPv6 connectivity or disable IPv6 for the service

# Python: adjust connection behavior
# Set IPv6 timeout shorter than Happy Eyeballs delay
import socket
socket.setdefaulttimeout(0.2)  # 200ms < 250ms Happy Eyeballs threshold

# Go: configure dialer timeout
# dialer := &net.Dialer{
#     Timeout:        5 * time.Second,
#     FallbackDelay: 100 * time.Millisecond,  # Custom Happy Eyeballs delay
# }

# Node.js: dns.lookup returns addresses in Happy Eyeballs order
# Use dns.resolve6 + dns.resolve4 directly to control behavior
```

## Step 5: Fix Broken IPv6 That Causes Fallback Delays

```bash
# If IPv6 is assigned but broken (causing 250ms delays):

# Option 1: Fix IPv6 (preferred)
# Ensure default route exists:
ip -6 route show default

# Ensure ICMPv6 is not blocked:
ping6 -c 1 2001:4860:4860::8888 || echo "IPv6 broken!"

# Option 2: Remove problematic IPv6 addresses
# Remove the IPv6 default route to force IPv4
sudo ip -6 route del default

# Option 3: Disable IPv6 on specific interface
sudo sysctl -w net.ipv6.conf.eth0.disable_ipv6=1

# Option 4: Adjust /etc/gai.conf to deprioritize IPv6
# Add to /etc/gai.conf to prefer IPv4:
# precedence ::ffff:0:0/96 100
```

## Conclusion

Happy Eyeballs delays are caused by IPv6 connection attempts that fail silently — the SYN is sent but never acknowledged. The fix is almost always to either repair IPv6 connectivity (fix the default route or unblock ICMPv6) or remove broken IPv6 routes that cause fruitless connection attempts. Measure with `curl -w "%{remote_ip}"` to confirm which protocol is actually used and `time curl -6` vs `time curl -4` to quantify the latency difference. A properly working dual-stack network shows similar response times for both protocols.
