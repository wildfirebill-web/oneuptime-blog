# How to Troubleshoot 'Request Timed Out' vs 'Destination Unreachable'

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ping, ICMP, Networking, Troubleshooting, IPv4, Firewall

Description: Understand the key differences between 'Request Timed Out' and 'Destination Unreachable' ping errors, and follow the right diagnostic path for each.

These two errors look similar but point to completely different problems. Confusing them leads to hours of wasted troubleshooting. Understanding the distinction immediately narrows your investigation.

## Key Differences

```yaml
Error Type                         What It Means
---------------------------------  ------------------------------------------
"Destination Host Unreachable"     ICMP error returned - routing failure
                                   You got feedback: network infrastructure
                                   knows the host is unreachable

"Request Timed Out" (Windows)      No response received within timeout
"No response" / silence (Linux)    Could be: host down, firewall blocking,
                                   MTU black hole, or routing to host works
                                   but return path is broken
```

## Destination Unreachable: Routing Problem

```bash
# Linux ping output showing unreachable:

ping 10.50.0.1
# From 192.168.1.1 icmp_seq=1 Destination Host Unreachable
# ^--- Response came back quickly, from router 192.168.1.1

# Diagnose: find which router is sending the error
traceroute 10.50.0.1
# hop 1: 192.168.1.1  - this router is sending the error
# Shows the exact router that has no route to the destination

# Fix options:
sudo ip route add 10.50.0.0/24 via 192.168.1.1  # Add route on your host
# Or fix routing on 192.168.1.1 itself
```

## Request Timed Out: Silent Drop or Return Path Issue

```bash
# Linux ping output showing timeout (silence):
ping 10.50.0.1
# (no response, just blank lines until Ctrl+C)
# 4 packets transmitted, 0 received, 100% packet loss

# Could be caused by:
# 1. Host is powered off
# 2. Firewall is silently dropping ICMP
# 3. Return path broken (host can receive but can't reply to you)
# 4. MTU black hole

# Test with traceroute:
traceroute 10.50.0.1
# If traceroute reaches the destination hop but ping times out:
# → destination is up but blocking ICMP
# → or return path is broken
```

## Distinguishing Firewall Block from Host Down

```bash
# Method 1: Try a TCP connection (firewalls may allow TCP but block ICMP)
nc -zv 10.50.0.1 22    # Test SSH port
nc -zv 10.50.0.1 80    # Test HTTP port
# If TCP connects but ping times out → firewall blocking ICMP

# Method 2: Check with traceroute --no-dns
traceroute -n 10.50.0.1
# If last successful hop is one before target → firewall or host down
# If last hop IS the target → host is up but blocking ICMP

# Method 3: Use TCP traceroute
sudo tcptraceroute 10.50.0.1 80
# Goes through firewalls that block UDP traceroute
```

## Asymmetric Routing (Request Timed Out Despite Host Being Up)

```bash
# Symptom: Traceroute reaches the host, but ping times out
# Cause: Host can receive your packets but replies go a different path
#        that doesn't reach you (or is blocked)

# Test from target host back to you:
# On target: ping <your-ip>
# If that also times out → routing is broken in both directions
# If that works → your return path to you is broken

# Fix: add a return route on the target
# On target machine:
sudo ip route add 10.0.0.0/8 via 10.50.0.254
```

## Quick Diagnostic Decision Tree

```text
Ping fails
    |
    ├─ "Destination Host Unreachable" received?
    │       → YES: Routing failure. Check ip route, fix gateways.
    │
    └─ Silence / 100% loss?
            |
            ├─ traceroute reaches destination?
            │       → YES: Host is up, firewall blocking ICMP
            │       → Try TCP connections to confirm
            │
            └─ traceroute also fails?
                    → Routing broken or host is down
                    → Check physical connectivity, power, interface
```

The critical takeaway: "Destination Unreachable" with a quick response means the network is telling you something; silence with 100% loss means you must investigate further.
