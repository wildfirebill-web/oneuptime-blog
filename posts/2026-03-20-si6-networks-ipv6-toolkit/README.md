# How to Use the SI6 Networks IPv6 Toolkit

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, SI6 Networks, Security, Network Analysis, Toolkit, Diagnostics

Description: Use the SI6 Networks IPv6 Toolkit for advanced IPv6 security auditing, protocol analysis, and network reconnaissance including address scanning and NDP testing.

## Introduction

The SI6 Networks IPv6 Toolkit is a comprehensive set of tools for IPv6 security assessment and network analysis. It includes tools for address scanning, header manipulation, Neighbor Discovery Protocol (NDP) analysis, and path MTU testing. Unlike standard tools, the SI6 toolkit operates at the packet level for deep protocol analysis.

## Installation

```bash
# Install on Ubuntu/Debian
sudo apt install -y ipv6toolkit

# Or build from source
git clone https://github.com/fgont/ipv6toolkit.git
cd ipv6toolkit
make
sudo make install

# Verify installation
addr6 --version
scan6 --help
```

## addr6: IPv6 Address Analysis

`addr6` analyzes and manipulates IPv6 addresses:

```bash
# Analyze a single IPv6 address
addr6 -a 2001:db8::1
# Output shows: address type, scope, interface ID analysis

# Check if address is SLAAC-generated (based on MAC)
addr6 -a fe80::1234:56ff:fe78:9abc

# Generate all addresses in a prefix
addr6 -p 2001:db8::/64 --gen-addr

# Convert between different address formats
addr6 -a 2001:0db8:0000:0000:0000:0000:0000:0001 -e
# Outputs compressed form: 2001:db8::1

# Check if address is in a specific type (multicast, link-local, etc.)
addr6 -a ff02::1 -d
```

## scan6: IPv6 Network Scanning

`scan6` discovers live IPv6 hosts on a network:

```bash
# Scan the local link for live hosts
sudo scan6 -i eth0 -L

# Scan a specific prefix
sudo scan6 -i eth0 -d 2001:db8::/64

# Scan with verbose output
sudo scan6 -i eth0 -L -v

# Use multiple probing techniques
sudo scan6 -i eth0 -L --probe-type all

# Scan for specific port (TCP)
sudo scan6 -i eth0 -d 2001:db8::/64 --tcp-scan --dst-port 443

# Save results to file
sudo scan6 -i eth0 -L -o /tmp/ipv6-hosts.txt
```

## na6/ns6: Neighbor Advertisement/Solicitation

Tools for NDP protocol manipulation and testing:

```bash
# Send a Neighbor Solicitation to discover a host
sudo ns6 -i eth0 -d 2001:db8::1

# Send a Neighbor Advertisement
sudo na6 -i eth0 -d ff02::1 -t 2001:db8::10 \
    --target-lla 00:11:22:33:44:55

# Flood with Neighbor Advertisements (security testing)
sudo na6 -i eth0 -d ff02::1 --flood-sources 100 \
    --target-lla 00:11:22:33:44:55 -l

# Test NDP cache exhaustion resistance
sudo na6 -i eth0 -d fe80::1 --flood-sources 200
```

## ra6: Router Advertisement Testing

```bash
# Send a Router Advertisement
sudo ra6 -i eth0 -d ff02::1 \
    --prefix 2001:db8:test::/64 \
    --prefix-life 1800 3600 \
    -A -l

# Test RA guard bypass techniques (research/audit)
sudo ra6 -i eth0 -d ff02::1 --loop \
    --prefix 2001:db8:rogue::/64

# Send RA to suppress existing prefixes (lifetime=0)
sudo ra6 -i eth0 -d ff02::1 \
    --prefix 2001:db8::/64 \
    --prefix-life 0 0
```

## frag6: IPv6 Fragmentation Testing

```bash
# Send fragmented IPv6 packets for firewall testing
sudo frag6 -i eth0 -s 2001:db8::10 -d 2001:db8::1 \
    --frag-size 512 --data-size 2048

# Test minimum fragment size handling
sudo frag6 -i eth0 -s 2001:db8::10 -d 2001:db8::1 \
    --frag-size 8

# Assess fragment handling behavior
sudo frag6 -i eth0 -d 2001:db8::1 --assess
```

## flow6: IPv6 Flow Label Analysis

```bash
# Analyze flow labels in captured traffic
flow6 -i /tmp/capture.pcap

# Test flow label handling
sudo flow6 -i eth0 -d 2001:db8::1 --flow-label 0x12345
```

## ipv6loopback: Loopback Testing

```bash
# Test IPv6 loopback path
sudo ipv6loopback -i eth0 -d 2001:db8::1

# Measure loopback latency
sudo ipv6loopback -i eth0 -d 2001:db8::1 --count 10
```

## Real-World Security Audit Use Cases

```bash
# 1. Discover all IPv6 hosts on your network segment
sudo scan6 -i eth0 -L -v 2>/dev/null | grep "^2"

# 2. Check for rogue Router Advertisements
sudo tcpdump -n -i eth0 icmp6 and ip6[40] == 134 2>/dev/null &
sleep 10; kill %1
# Type 134 = Router Advertisement

# 3. Scan for IPv6 hosts that don't appear in your inventory
sudo scan6 -i eth0 -L -o /tmp/ipv6-actual.txt
diff /tmp/ipv6-expected.txt /tmp/ipv6-actual.txt

# 4. Test NDP cache sizes (DoS resistance)
echo "Testing NDP cache with 100 fake entries..."
sudo na6 -i eth0 -d fe80::router \
    --flood-sources 100 \
    --target-lla 00:00:00:00:00:00 \
    --loop --rate 10
```

## Conclusion

The SI6 Networks IPv6 Toolkit provides tools that go far beyond standard system utilities for IPv6 analysis and security auditing. `scan6` discovers hosts using multiple NDP techniques, `addr6` analyzes address properties, and tools like `ra6` and `na6` enable deep testing of NDP protocol behavior. These tools are valuable for security auditing, network inventory, and IPv6 protocol research. Always use these tools only on networks you own or have explicit permission to test.
