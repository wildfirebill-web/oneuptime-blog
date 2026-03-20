# How to Calculate the Broadcast Address for Any Subnet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Subnetting, Broadcast, IPv4, CIDR, Network Fundamentals

Description: Calculate the directed broadcast address for any IPv4 subnet using binary math, Python, and Linux command-line tools, with worked examples for common CIDR prefixes.

## Introduction

The broadcast address of a subnet is the last address in the range — all host bits set to 1. Every device on that subnet receives a packet sent to this address. Calculating it correctly is essential for network configuration, firewall rules, and troubleshooting.

## The Formula

1. Take the network address (all host bits = 0)
2. Apply a **wildcard mask** (inverse of subnet mask) using bitwise OR
3. The result is the broadcast address

```
Broadcast = Network Address | Wildcard Mask
Wildcard  = ~SubnetMask  (bitwise NOT of the mask)
```

## Worked Example: 192.168.1.0/24

```
Subnet mask:   255.255.255.0   = 11111111.11111111.11111111.00000000
Wildcard mask:   0.  0.  0.255 = 00000000.00000000.00000000.11111111
Network:       192.168.1.0     = 11000000.10101000.00000001.00000000
                                 OR
Broadcast:     192.168.1.255   = 11000000.10101000.00000001.11111111
```

## Worked Example: 10.10.10.128/26

```
/26 mask: 255.255.255.192 = ...11000000
Wildcard:   0.  0.  0. 63 = ...00111111
Network:  10.10.10.128     = ...10000000
                              OR
Broadcast:10.10.10.191     = ...10111111
```

## Python Script for Any Subnet

```python
#!/usr/bin/env python3
import ipaddress
import sys

def subnet_info(cidr: str):
    """Print network, broadcast, range, and host count for a subnet."""
    net = ipaddress.IPv4Network(cidr, strict=False)
    print(f"CIDR:             {net}")
    print(f"Network:          {net.network_address}")
    print(f"Broadcast:        {net.broadcast_address}")
    print(f"Subnet Mask:      {net.netmask}")
    print(f"Wildcard Mask:    {net.hostmask}")
    print(f"First Host:       {net.network_address + 1}")
    print(f"Last Host:        {net.broadcast_address - 1}")
    print(f"Usable Hosts:     {net.num_addresses - 2}")

# Run from command line: python3 script.py 192.168.5.64/27
if len(sys.argv) > 1:
    subnet_info(sys.argv[1])
else:
    # Default examples
    for cidr in ["192.168.1.0/24", "10.0.0.0/8", "172.16.5.128/25", "192.168.10.64/26"]:
        subnet_info(cidr)
        print()
```

## Using ipcalc on Linux

```bash
# Install ipcalc
sudo apt install ipcalc

# Calculate broadcast and other fields
ipcalc 192.168.5.0/27
```

Output:

```
Address:   192.168.5.0          11000000.10101000.00000101. 00000000
Netmask:   255.255.255.224 = 27 11111111.11111111.11111111. 11100000
Wildcard:  0.0.0.31             00000000.00000000.00000000. 00011111
Network:   192.168.5.0/27
Broadcast: 192.168.5.31
HostMin:   192.168.5.1
HostMax:   192.168.5.30
Hosts/Net: 30
```

## Quick Reference Table

| CIDR | Hosts | Broadcast Offset |
|---|---|---|
| /30 | 2 | .3 from network |
| /29 | 6 | .7 |
| /28 | 14 | .15 |
| /27 | 30 | .31 |
| /26 | 62 | .63 |
| /25 | 126 | .127 |
| /24 | 254 | .255 |

## Conclusion

The broadcast address is always the bitwise OR of the network address and the wildcard mask. Use Python's `ipaddress` module or `ipcalc` for instant calculations, and keep the pattern — all host bits set to 1 — in mind when manually verifying subnet boundaries.
