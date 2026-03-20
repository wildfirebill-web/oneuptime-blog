# How to Conduct an IPv6 Security Assessment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Security, Penetration Testing, Assessment, Tools

Description: Learn how to conduct a technical IPv6 security assessment using scanning, protocol analysis, and attack simulation to identify vulnerabilities in your IPv6 deployment.

## Overview

An IPv6 security assessment goes beyond auditing configuration - it actively tests the network to verify that defenses work as expected. This includes probing for rogue RA vulnerabilities, testing extension header filtering, checking PMTUD behavior, and identifying IPv6 hosts that aren't in your inventory.

## Assessment Tools

| Tool | Purpose |
|------|---------|
| nmap | IPv6 host/service scanning |
| THC-IPv6 toolkit | IPv6-specific attack simulation |
| scapy | Craft custom IPv6 packets |
| radvdump | Capture and analyze Router Advertisements |
| tcpdump/tshark | Protocol-level traffic analysis |
| ndpmon | NDP monitoring and anomaly detection |
| metasploit | Exploit known IPv6 vulnerabilities |
| fierce | DNS enumeration for AAAA records |

## Phase 1: IPv6 Host Discovery

```bash
# Passive discovery - listen for NDP

tcpdump -i eth0 'icmp6' -l -n | grep -E 'NS|NA|RS|RA' &

# Active discovery - ping all-nodes multicast
ping6 -c 3 ff02::1%eth0

# Scan known host patterns
nmap -6 -sn 2001:db8:1::1-ff     # Sequential low addresses
nmap -6 -sn "2001:db8:1::1,::2,::10,::100,::dead,::cafe"

# EUI-64 pattern scan - if you know OUI of devices on link
# MAC 00:50:56:xx:xx:xx (VMware) → ::0250:56ff:feXX:XXXX
nmap -6 -sn 2001:db8:1::250:56ff:fe00:0/112   # VMware EUI-64 range
```

## Phase 2: Service Discovery

```bash
# nmap: Full service scan of discovered IPv6 hosts
nmap -6 -sV -A -p- 2001:db8:1::10

# Check for dual-stack - same services exposed on IPv6?
nmap -6 -p 22,80,443,8080,3306 2001:db8:1::0/64 --open

# DNS AAAA enumeration
fierce --domain example.com --dns-servers ns1.example.com
# Or with dnsx
dnsx -d example.com -aaaa
```

## Phase 3: Rogue RA Testing

```bash
# Test RA Guard - are access ports protected?
# From a host on the segment, try sending an RA
# If other hosts reconfigure - RA Guard is NOT working

# Method 1: Using fake_router6 (THC-IPv6)
fake_router6 eth0 2001:db8:test::/64

# Method 2: Using scapy
python3 -c "
from scapy.all import *
ra = (Ether(dst='33:33:00:00:00:01') /
      IPv6(dst='ff02::1') /
      ICMPv6ND_RA(M=1, O=1, routerlifetime=1800) /
      ICMPv6NDOptPrefixInfo(prefix='2001:db8:attacker::', prefixlen=64, L=1, A=1))
sendp(ra, iface='eth0', count=5, inter=1)
"

# After sending - check if a victim host received the RA
# SSH to victim and check:
ip -6 route show | grep 'proto ra'
```

## Phase 4: Extension Header Testing

```bash
# Test Routing Header Type 0 filtering
python3 -c "
from scapy.all import *
pkt = IPv6(dst='target-ip')/IPv6ExtHdrRouting(type=0, addresses=['2001:db8::dead'])/TCP(dport=80)
send(pkt, count=1)
"
# Should be blocked - if it arrives, RH0 filtering is missing

# Test Hop-by-Hop header
python3 -c "
from scapy.all import *
pkt = IPv6(dst='target-ip')/IPv6ExtHdrHopByHop()/TCP(dport=80)
send(pkt, count=1)
"
```

## Phase 5: Fragment Testing

```bash
# Test tiny fragment attack (RFC 7112 compliance)
python3 -c "
from scapy.all import *
# First fragment so small TCP header doesn't fit
frag1 = IPv6(dst='target')/IPv6ExtHdrFragment(offset=0, m=1, id=0x1234)/TCP(dport=80)[:40]
send(frag1)
"
# RFC 7112 compliant devices should reject this

# Test atomic fragment handling
python3 -c "
from scapy.all import *
atomic = IPv6(dst='target')/IPv6ExtHdrFragment(offset=0, m=0, id=0xdeadbeef)/ICMP()
send(atomic)
"
```

## Phase 6: ICMPv6 Policy Testing

```bash
# Test that Packet Too Big passes through (required by RFC 4821)
# If PTB is blocked, PMTUD fails and large TCP transfers break
hping3 -6 --icmpv6 -C 2 --icmpv6-code 0 target-ip
# Should not be blocked at any intermediate device

# Test that echo requests are handled as expected
ping6 -c 5 target-ip
# Document response - note if rate limited
```

## Assessment Report Template

```text
IPv6 Security Assessment Report
================================
Date:
Assessed Network:
Assessor:

Findings:

1. [CRITICAL] IPv6 active with no firewall policy
   - Hosts discovered: X
   - No ip6tables/nftables rules found
   - Recommendation: Deploy host firewall policy

2. [HIGH] RA Guard not deployed on access switches
   - Rogue RA test successful - hosts reconfigured
   - Recommendation: Deploy RA Guard on all access ports

3. [MEDIUM] RH0 not filtered
   - Packets with Type 0 Routing Header reached target
   - Recommendation: Block protocol 43 type 0 at perimeter

4. [LOW] IPv6 traffic not logged in SIEM
   - Recommendation: Add IPv6 log sources

Risk Score:
Remediation Priority:
```

## Summary

An IPv6 security assessment progresses through discovery, service scanning, rogue RA testing, extension header probing, fragment testing, and ICMPv6 policy verification. Use a combination of passive listening (tcpdump, ndpmon), active scanning (nmap), and custom packet crafting (scapy, THC-IPv6 tools) to simulate real attacks and validate defenses. Document findings with risk levels and prioritize remediation.
