# How to Use the SI6 Networks rs6 Tool for Router Solicitation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SI6 Networks, rs6, IPv6, Router Solicitation, NDP, Security Testing

Description: A guide to using the SI6 Networks rs6 tool to send ICMPv6 Router Solicitation messages for IPv6 network testing and security assessment.

The `rs6` tool from the SI6 Networks IPv6 toolkit crafts and sends ICMPv6 Router Solicitation (RS) messages. Router Solicitations are sent by IPv6 hosts to request Router Advertisements (RAs) immediately rather than waiting for the periodic RA interval. `rs6` is useful for testing router RA behavior, triggering address autoconfiguration, and stress-testing RA rate limiting.

## Installing the SI6 Networks Toolkit

```bash
sudo apt-get install ipv6toolkit   # Debian/Ubuntu
sudo pacman -S ipv6toolkit          # Arch Linux
```

## What is a Router Solicitation?

When an IPv6 host connects to a network, it sends a Router Solicitation to `ff02::2` (all-routers multicast) to request Router Advertisements immediately. Routers respond with RAs containing prefix information, default gateway, and other network configuration.

```
Host                    Router
 |                        |
 |--- RS to ff02::2 ----->|
 |                        |
 |<-- RA with prefixes ---|
 |                        |
```

## Basic rs6 Usage

```bash
# Send a Router Solicitation from eth0
sudo rs6 -i eth0

# Send RS to the all-routers multicast (standard behavior)
sudo rs6 -i eth0 -d ff02::2

# Send RS from a specific source address
sudo rs6 -i eth0 -s fe80::1234 -d ff02::2

# Send RS to a specific router's link-local address
sudo rs6 -i eth0 -d fe80::router
```

## Source Link-Layer Address (SLLA) Option

```bash
# Include source link-layer address option (standard in RS)
sudo rs6 -i eth0 --slla

# Spoof the source link-layer address
sudo rs6 -i eth0 --slla 00:11:22:33:44:55

# Send from unspecified source (::) — allowed in RS per RFC 4861
sudo rs6 -i eth0 -s ::
```

## Testing Router RA Rate Limiting

IPv6 routers should rate-limit RAs to prevent flooding. Test this with rapid RS messages:

```bash
# Send 100 RS messages to trigger multiple RAs
sudo rs6 -i eth0 --loop --sleep 0 --loop-count 100

# Measure RA response rate
sudo tcpdump -i eth0 -n icmp6 and ip6[40] == 134 &   # Type 134 = RA
sudo rs6 -i eth0 --loop --sleep 0 --loop-count 100
```

## RS from Multiple Sources (Flooding Test)

```bash
# Send RS from many different source addresses to stress-test the router
# This tests router behavior under high RS load
sudo rs6 -i eth0 \
  --src-addr-shuffle \   # Randomize source addresses
  --loop \
  --sleep 0 \
  --loop-count 1000
```

## Triggering SLAAC on Demand

`rs6` can be used to manually trigger address autoconfiguration without waiting for the next periodic RA:

```bash
# Trigger RA from default router (useful after interface comes up)
sudo rs6 -i eth0

# Verify new addresses appeared after RA
ip -6 addr show eth0
```

## Capturing the RA Response

```bash
# Listen for RA responses while sending RS
sudo tcpdump -i eth0 -n -v 'icmp6 and ip6[40] == 134' &
sudo rs6 -i eth0
```

## rs6 vs rdisc6

Both `rs6` and `rdisc6` (from ndisc6 package) can trigger RS/RA exchanges:

| Tool | Source | Best For |
|---|---|---|
| rs6 | SI6 Networks | Custom RS crafting, flood testing |
| rdisc6 | ndisc6 | Simple RA display |
| nmap --script ipv6-ra-flood | nmap | Basic testing |

```bash
# Simple RA discovery using rdisc6
rdisc6 eth0

# More control using rs6
sudo rs6 -i eth0 -s :: -d ff02::2
```

## Defensive Considerations

`rs6`-style attacks don't directly harm hosts, but routers responding to many RS messages can be overwhelmed:

- Router RA rate limiting should be configured (typically 1 RA per 3-10 seconds per interface)
- Monitor for unusual RS floods using NDPMon or Wireshark
- Legitimate IPv6 hosts send RS only on interface state changes, not continuously

`rs6` is primarily useful for understanding how routers respond to solicitation messages and verifying that proper rate limiting is in place to prevent RA flooding triggered by RS storms.
