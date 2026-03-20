# How to View All Network Interfaces and Their IPv4 Addresses on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, IPv4, ip command, Network Diagnostics

Description: List all network interfaces and their IPv4 addresses on Linux using the ip command, with filtering options to display only active interfaces or specific address families.

## Introduction

Quickly seeing all interfaces and their IPv4 addresses is one of the most common network administration tasks. The modern `ip` command provides rich output with filtering options.

## Show All Interfaces and Addresses

```bash
# Show all addresses on all interfaces

ip addr show

# Short alias
ip a
```

## Show Only IPv4 Addresses

```bash
# Filter to only IPv4 (inet) addresses
ip -4 addr show

# Or use -f inet
ip -f inet addr show
```

## Show a Specific Interface

```bash
# Show only eth0
ip addr show dev eth0

# Short form
ip a show eth0
```

## Show Only UP Interfaces

```bash
# List only interfaces that are currently UP
ip link show up

# Then get addresses for each UP interface
ip -4 addr show up
```

## One-Line Summary with awk

For a compact view (interface name and IP):

```bash
# Print interface name and IPv4 address on one line each
ip -4 addr show | awk '/inet / {print $NF, $2}'
```

Output:

```text
lo 127.0.0.1/8
eth0 192.168.1.100/24
eth1 10.0.0.5/30
```

## Using a Python Script for Programmatic Access

```python
#!/usr/bin/env python3
import subprocess
import re

def get_ipv4_interfaces():
    """Return a dict of {interface: [IPv4 addresses]}."""
    output = subprocess.check_output(["ip", "-4", "addr", "show"], text=True)
    result = {}
    iface = None
    for line in output.splitlines():
        # Detect interface line
        m = re.match(r'^\d+:\s+(\S+):', line)
        if m:
            iface = m.group(1)
            result[iface] = []
        # Detect inet line
        m = re.match(r'\s+inet (\S+)', line)
        if m and iface:
            result[iface].append(m.group(1))
    return result

for iface, addrs in get_ipv4_interfaces().items():
    for addr in addrs:
        print(f"{iface}: {addr}")
```

## Checking Interface Statistics

```bash
# Show interface statistics (bytes, packets, errors)
ip -s link show

# For a specific interface
ip -s link show dev eth0
```

## Legacy Method: ifconfig (deprecated)

```bash
# Still works on most systems (deprecated, prefer ip)
ifconfig -a

# Show only IPv4 addresses
ifconfig | grep "inet "
```

## Conclusion

Use `ip -4 addr show` for a clean IPv4-only view of all interfaces. Add `up` to filter to only active interfaces. For scripting, parse with `awk` or Python's subprocess module. The `ip` command is authoritative and consistent across all modern Linux distributions.
