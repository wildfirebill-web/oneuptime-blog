# How to Troubleshoot Dual-Stack IPv4/IPv6 Connectivity Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dual-Stack, IPv4, IPv6, Troubleshooting, Connectivity, Networking

Description: Diagnose and fix dual-stack networking issues including Happy Eyeballs failures, IPv6 preference problems, DNS resolution inconsistencies, and routing asymmetry.

## Introduction

Dual-stack deployments introduce new failure modes: an application may prefer IPv6 but the IPv6 path is broken, DNS returns both A and AAAA but one is unreachable, or routing works on one protocol but not the other. This guide covers systematic diagnosis of dual-stack connectivity problems.

## Happy Eyeballs and Protocol Selection

```bash
# Happy Eyeballs (RFC 8305): clients try IPv6 first, fall back to IPv4 if IPv6 fails

# When something is "slow" in dual-stack, IPv6 is often broken

# Test which protocol is being used:
curl -v https://example.com 2>&1 | grep "Connected to"
# Look for IPv4 or IPv6 address in the output

# Force specific protocol:
curl -4 https://example.com  # Force IPv4
curl -6 https://example.com  # Force IPv6 (will fail if IPv6 broken)
```

## Diagnosing IPv6 Issues in Dual-Stack

```bash
# Step 1: Is IPv6 working at all?
ping -6 ::1              # Loopback (should always work)
ping -6 2001:4860:4860::8888   # Google's IPv6 DNS

# Step 2: Is the IPv6 default route configured?
ip -6 route show default
# Expected: default via <gateway> dev eth0

# Step 3: Can you reach the IPv6 gateway?
ping -6 <your-ipv6-gateway>

# Step 4: Is IPv6 DNS resolving?
dig AAAA google.com
# Expected: AAAA records returned

# Step 5: Check IPv6 is not disabled:
cat /proc/sys/net/ipv6/conf/eth0/disable_ipv6
# 0 = enabled, 1 = disabled
```

## DNS Issues in Dual-Stack

```bash
# DNS returns AAAA but IPv6 is broken → connection hangs then falls back

# Check what DNS returns:
host example.com
# Should show both A and AAAA if host is dual-stack

# Test specific record types:
dig -4 A example.com       # Force IPv4 DNS query
dig -6 AAAA example.com    # Force IPv6 DNS query

# Compare:
host -t A example.com
host -t AAAA example.com

# If AAAA exists but IPv6 doesn't work:
# The application will be slow (waits for IPv6 timeout)
# Fix: either make IPv6 work OR remove AAAA record
```

## Routing Issues

```bash
# Check routing tables for both protocols
ip -4 route show
ip -6 route show

# Test path for specific destination:
traceroute -4 google.com   # IPv4 path
traceroute -6 google.com   # IPv6 path

# Compare path lengths (asymmetric paths are normal)

# Check if specific prefix is reachable:
ip -6 route get 2001:4860:4860::8888
# Shows: via <gateway> dev <interface> src <local-ipv6>

# Test MTU (IPv6 requires minimum 1280 bytes):
ping -6 -s 1452 -M do 2001:4860:4860::8888
# If this fails, MTU issues exist on the IPv6 path
```

## Protocol Preference Issues

```bash
# Check /etc/gai.conf for address preference ordering
cat /etc/gai.conf | grep -v "^#\|^$"

# Default: IPv6 preferred over IPv4
# To prefer IPv4 in gai.conf:
sudo sed -i 's/^#\s*precedence ::ffff:0:0\/96  100/precedence ::ffff:0:0\/96  100/' /etc/gai.conf

# Or add at end of /etc/gai.conf:
echo "precedence ::ffff:0:0/96  100" | sudo tee -a /etc/gai.conf

# Test after change (what getaddrinfo returns first):
python3 -c "import socket; print([r[4][0] for r in socket.getaddrinfo('google.com',80)])"
```

## Application-Level Dual-Stack Testing

```bash
# Test if an application listens on both protocols:
sudo ss -tlnp | grep :443
# Expect: both 0.0.0.0:443 and :::443 (or *:443 for both)

# Or for Python/Ruby/Node apps that may only bind to IPv4:
# Check bind address in application configuration

# Test with nc:
nc -4 example.com 80    # IPv4 connection
nc -6 example.com 80    # IPv6 connection
echo "GET / HTTP/1.0" | nc -6 example.com 80
```

## Conclusion

Dual-stack troubleshooting starts by isolating which protocol family has the issue using `curl -4` / `curl -6` or `ping -4` / `ping -6`. Check DNS returns the correct records for both families, verify default routes exist for both IPv4 and IPv6, and ensure IPv6 isn't disabled on the interface. Slow connections often indicate a broken IPv6 path causing Happy Eyeballs delays - fix the IPv6 path or disable AAAA records until IPv6 is properly working.
