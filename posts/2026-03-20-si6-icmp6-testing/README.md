# How to Use the SI6 Networks icmp6 Tool for ICMPv6 Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SI6 Networks, ICMP6, IPv6, ICMPv6, Security Testing, Network Testing

Description: A guide to using the SI6 Networks icmp6 tool to craft and send ICMPv6 messages for testing IPv6 network behavior and security controls in authorized environments.

The `icmp6` tool from the SI6 Networks IPv6 toolkit provides fine-grained control over ICMPv6 packet construction. Unlike standard ping6, `icmp6` allows crafting any ICMPv6 type and code, making it useful for testing how hosts and firewalls respond to various ICMPv6 messages, including error messages, unreachable conditions, and informational requests.

## Installing the SI6 Networks Toolkit

```bash
sudo apt-get install ipv6toolkit   # Debian/Ubuntu
sudo pacman -S ipv6toolkit          # Arch Linux
```

## Basic icmp6 Usage

```bash
# Send an ICMPv6 Echo Request (like ping6)

sudo icmp6 -i eth0 -d 2001:db8::target -t 128 -c 0

# -t = ICMPv6 type
# -c = ICMPv6 code

# Send ICMPv6 Echo Reply
sudo icmp6 -i eth0 -d 2001:db8::target -t 129 -c 0
```

## ICMPv6 Type Reference

| Type | Name | Use |
|---|---|---|
| 1 | Destination Unreachable | Error: host/port unreachable |
| 2 | Packet Too Big | PMTUD signaling |
| 3 | Time Exceeded | Hop limit exceeded (traceroute) |
| 4 | Parameter Problem | Malformed header |
| 128 | Echo Request | Ping |
| 129 | Echo Reply | Ping response |
| 133 | Router Solicitation | NDP: request RA |
| 134 | Router Advertisement | NDP: announce prefix |
| 135 | Neighbor Solicitation | NDP: address resolution |
| 136 | Neighbor Advertisement | NDP: address response |

## Testing Destination Unreachable Handling

```bash
# Send Destination Unreachable - No route to host (code 0)
sudo icmp6 -i eth0 \
  -s 2001:db8::router \
  -d 2001:db8::target \
  -t 1 -c 0

# Send Destination Unreachable - Port unreachable (code 4)
sudo icmp6 -i eth0 \
  -s 2001:db8::server \
  -d 2001:db8::client \
  -t 1 -c 4

# Send Destination Unreachable - Address unreachable (code 3)
sudo icmp6 -i eth0 -t 1 -c 3 -d 2001:db8::target
```

## Testing Packet Too Big (PMTUD)

ICMPv6 Packet Too Big (Type 2) is critical for Path MTU Discovery:

```bash
# Send a Packet Too Big message announcing MTU 1280
sudo icmp6 -i eth0 \
  -s 2001:db8::router \
  -d 2001:db8::source \
  -t 2 -c 0 \
  --mtu 1280

# Test if host correctly reduces packet size after receiving PTB
# Verify with: ip route show cache | grep mtu
```

## Testing Time Exceeded (Traceroute Behavior)

```bash
# Send Time Exceeded - Hop limit exceeded (simulates router)
sudo icmp6 -i eth0 \
  -s 2001:db8::router \
  -d 2001:db8::originator \
  -t 3 -c 0    # code 0 = hop limit exceeded in transit
```

## Flooding Tests

```bash
# ICMPv6 echo request flood (test rate limiting)
sudo icmp6 -i eth0 -d 2001:db8::target -t 128 -c 0 \
  --loop --sleep 0 --loop-count 10000

# Flood with varying source addresses (test source-based rate limiting)
sudo icmp6 -i eth0 -d 2001:db8::target -t 128 -c 0 \
  --src-addr-shuffle \
  --loop --sleep 0
```

## Testing Firewall ICMPv6 Rules

A properly configured IPv6 firewall must permit specific ICMPv6 types for basic connectivity:

```bash
# Test if Packet Too Big passes through firewall (required for PMTUD)
sudo icmp6 -i eth0 -s 2001:db8::router -d 2001:db8::internal -t 2 -c 0 --mtu 1280

# Test if Echo Request passes (for ping connectivity checks)
sudo icmp6 -i eth0 -d 2001:db8::target -t 128 -c 0

# Test if Destination Unreachable passes (required for proper TCP)
sudo icmp6 -i eth0 -s 2001:db8::server -d 2001:db8::client -t 1 -c 4
```

## Minimum Required ICMPv6 Types (RFC 4890)

| Type | Required | Reason |
|---|---|---|
| 1 (Dest. Unreachable) | Yes | TCP proper teardown |
| 2 (Packet Too Big) | Yes | PMTUD (critical) |
| 3 (Time Exceeded) | Yes | Traceroute, loop detection |
| 128/129 (Echo) | Recommended | Diagnostic |
| 133-136 (NDP) | Yes | Address autoconfiguration |

Never block ICMPv6 wholesale - it breaks IPv6 fundamental protocols. Use `icmp6` to verify your firewall permits the correct types.
