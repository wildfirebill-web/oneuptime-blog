# How to Test IPv6 Firewall Rules with Packet Crafting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Firewall Testing, Packet Crafting, Security Testing, Ip6tables, Scapy

Description: A guide to testing IPv6 firewall rules using packet crafting tools to verify that ip6tables, pf, and hardware firewalls correctly filter IPv6 traffic.

Testing firewall rules with crafted packets verifies that your IPv6 firewall actually blocks what it should block - and permits what it should permit. This is especially important for IPv6 where firewalls may have incomplete rule sets compared to their IPv4 counterparts.

## Testing Methodology

```text
Blocked:  Crafted packet → Firewall → DROP (no response)
Permitted: Crafted packet → Firewall → Passes through
```

## Checking Existing ip6tables Rules

```bash
# View all IPv6 firewall rules

sudo ip6tables -L -n -v --line-numbers

# View specific table
sudo ip6tables -t filter -L -n -v
sudo ip6tables -t mangle -L -n -v

# Check if rules exist (common issue: IPv6 rules missing when IPv4 rules present)
sudo ip6tables -L | wc -l   # Should be more than 3
sudo iptables -L | wc -l    # Compare IPv4 rule count
```

## Testing with hping3 (IPv6)

```bash
# TCP SYN to blocked port (expect no response)
hping3 -6 -S -p 23 2001:db8::target

# TCP SYN to allowed port (expect SYN-ACK)
hping3 -6 -S -p 443 2001:db8::target

# UDP probe
hping3 -6 -2 -p 161 2001:db8::target

# ICMP echo test
hping3 -6 --icmpv6 2001:db8::target
```

## Testing with Scapy

```python
from scapy.all import *
from scapy.layers.inet6 import *

target = "2001:db8::target"

# Test TCP port (expect RST for open, no response for filtered)
pkt = IPv6(dst=target) / TCP(dport=22, flags="S")
resp = sr1(pkt, timeout=2, verbose=0, iface="eth0")

if resp:
    if resp.haslayer(TCP):
        if resp[TCP].flags == "SA":
            print(f"Port 22: OPEN")
        elif resp[TCP].flags == "RA":
            print(f"Port 22: CLOSED (RST)")
else:
    print(f"Port 22: FILTERED (no response)")

# Test ICMPv6 pass-through
pkt = IPv6(dst=target) / ICMPv6EchoRequest()
resp = sr1(pkt, timeout=2, verbose=0, iface="eth0")
print("ICMP Echo:", "ALLOWED" if resp else "BLOCKED")
```

## Testing Extension Header Filtering

Many firewalls have bugs in handling extension headers:

```bash
# Test RA with hop-by-hop header (bypass RA Guard)
sudo ra6 -i eth0 --hbh-opt -P 2001:db8:test::/64 -d 2001:db8::target

# Test fragments (ensure firewall reassembles before filtering)
sudo frag6 -i eth0 -d 2001:db8::target --frag-size 8 --proto tcp --dport 80

# Test type 0 routing header (should be blocked per RFC 5095)
python3 -c "
from scapy.all import *
from scapy.layers.inet6 import *
pkt = IPv6(dst='2001:db8::target') / IPv6ExtHdrRouting(type=0) / TCP(dport=80)
send(pkt, iface='eth0')
"
```

## Required ICMPv6 Types (Must Not Be Blocked)

Test that critical ICMPv6 is permitted:

```bash
# Packet Too Big - must be allowed (required for PMTUD)
python3 -c "
from scapy.all import *
from scapy.layers.inet6 import *
pkt = IPv6(dst='2001:db8::target') / ICMPv6PacketTooBig(mtu=1280)
send(pkt, iface='eth0')
"

# Verify by checking if connection establishes to large-packet services
# If PTB is blocked: large HTTP requests will hang
curl -6 --max-time 10 http://[2001:db8::target]/large-file
```

## Common IPv6 Firewall Test Cases

| Test | Expected Result | Why |
|---|---|---|
| ICMPv6 Echo Request | ALLOWED | Diagnostic |
| ICMPv6 Packet Too Big | ALLOWED | PMTUD requirement |
| NDP (Types 133-136) | ALLOWED | Address configuration |
| TCP to allowed ports | ALLOWED | Application traffic |
| TCP to denied ports | DROPPED | Security policy |
| Routing Header Type 0 | DROPPED | RFC 5095 |
| ICMPv6 RA from non-router | DROPPED | RA Guard |
| IPv6 fragments | Depends | Application-specific |

## Automated Firewall Rule Testing

```bash
#!/bin/bash
# ipv6-fw-test.sh - Test IPv6 firewall rules

TARGET="2001:db8::target"
IFACE="eth0"

test_port() {
  local port=$1
  local expected=$2
  result=$(nmap -6 -p $port --open $TARGET 2>/dev/null | grep -c "open")
  if [ "$result" = "$expected" ]; then
    echo "PASS: Port $port"
  else
    echo "FAIL: Port $port (expected=$expected, got=$result)"
  fi
}

# Should be open
test_port 443 1
test_port 22 1

# Should be filtered
test_port 23 0
test_port 3389 0
```

Regular firewall testing with packet crafting ensures that IPv6 firewall rules are as complete and correct as IPv4 rules - a gap that often exists in organizations that added IPv6 without reviewing their complete security posture.
