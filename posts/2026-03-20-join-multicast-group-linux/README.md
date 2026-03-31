# How to Join a Multicast Group on a Linux Interface

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Multicast, IGMP, Linux, Socket, Networking, UDP

Description: Join IPv4 multicast groups on Linux using socket options, the ip command, and programming interfaces, and verify group membership with system commands.

## Introduction

Joining a multicast group on Linux instructs the network stack to accept packets sent to a multicast group address and optionally triggers IGMP membership reports to inform the local router. This is done through socket options (`IP_ADD_MEMBERSHIP`), the `ip maddr` command, or by applications that call the appropriate system calls. Understanding how to join groups is essential for deploying multicast applications and troubleshooting multicast connectivity.

## Join Multicast Group via Socket Options

```python
#!/usr/bin/env python3
# Join multicast group and receive packets

import socket
import struct

MCAST_GRP = '239.255.0.1'
MCAST_PORT = 5007

# Create UDP socket:

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)

# Allow multiple sockets on same port:
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)

# Bind to multicast port:
sock.bind(('', MCAST_PORT))

# Join multicast group (on any interface):
mreq = struct.pack('4s4s',
    socket.inet_aton(MCAST_GRP),
    socket.inet_aton('0.0.0.0'))  # 0.0.0.0 = INADDR_ANY (default interface)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)

print(f"Joined {MCAST_GRP} on all interfaces, listening on port {MCAST_PORT}")

try:
    while True:
        data, (src_ip, src_port) = sock.recvfrom(4096)
        print(f"From {src_ip}:{src_port}: {data.decode(errors='replace')}")
except KeyboardInterrupt:
    pass
finally:
    # Leave group on exit:
    sock.setsockopt(socket.IPPROTO_IP, socket.IP_DROP_MEMBERSHIP, mreq)
    sock.close()
```

## Join on Specific Interface

```python
#!/usr/bin/env python3
# Join multicast group on a specific interface

import socket
import struct

MCAST_GRP = '239.255.0.1'
MCAST_PORT = 5007
IFACE_IP = '192.168.1.10'  # IP address of the interface to join on

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

# Bind to the specific interface:
sock.bind((IFACE_IP, MCAST_PORT))

# Join on specific interface using its IP:
mreq = struct.pack('4s4s',
    socket.inet_aton(MCAST_GRP),
    socket.inet_aton(IFACE_IP))   # Specific interface IP
sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)

# Alternative using IP_ADD_MEMBERSHIP with ip_mreqn (includes interface index):
import ctypes
class ip_mreqn(ctypes.Structure):
    _fields_ = [
        ('imr_multiaddr', ctypes.c_byte * 4),
        ('imr_address', ctypes.c_byte * 4),
        ('imr_ifindex', ctypes.c_int),
    ]

IFACE_INDEX = socket.if_nametoindex('eth0')
mreqn = ip_mreqn()
mreqn.imr_multiaddr = (ctypes.c_byte * 4)(*socket.inet_aton(MCAST_GRP))
mreqn.imr_address = (ctypes.c_byte * 4)(0, 0, 0, 0)
mreqn.imr_ifindex = IFACE_INDEX

sock2 = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock2.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP,
                  bytes(mreqn))
```

## Join Multicast Group via ip Command

```bash
# Add static multicast group membership (no socket needed):
ip maddr add 239.255.0.1 dev eth0
# This makes the interface accept multicast for this group
# But does NOT send IGMP reports (no daemon involved)

# View multicast groups on interface:
ip maddr show eth0
# Or all interfaces:
ip maddr show

# Example output:
# 2:   eth0
#     link  01:00:5e:7f:00:01 static
#     inet  239.255.0.1
#     inet  224.0.0.1

# Remove static multicast group:
ip maddr del 239.255.0.1 dev eth0

# Check kernel's view of IGMP memberships:
cat /proc/net/igmp
# Format: Idx DevName Count Querier Group(hex) Users Timer Reporter
# Example: 2 eth0 2 V3 EF000001 1 0:00000000 0
#           Group EF000001 = 239.0.0.1 in little-endian hex
```

## Source-Specific Multicast (SSM) with IGMPv3

```python
#!/usr/bin/env python3
# Join SSM group (specific source + group) using IGMPv3

import socket
import struct
import ctypes

# Source-Specific Multicast: receive only from specific source
MCAST_GRP = '232.1.1.1'  # SSM range 232.0.0.0/8
MCAST_SRC = '10.20.0.5'  # Only receive from this source
MCAST_PORT = 5009

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(('', MCAST_PORT))

# IP_ADD_SOURCE_MEMBERSHIP for SSM:
# struct: group (4 bytes) + interface (4 bytes) + source (4 bytes)
mreq_source = struct.pack('4s4s4s',
    socket.inet_aton(MCAST_GRP),
    socket.inet_aton('0.0.0.0'),   # Any interface
    socket.inet_aton(MCAST_SRC))

sock.setsockopt(socket.IPPROTO_IP,
                socket.IP_ADD_SOURCE_MEMBERSHIP,
                mreq_source)

print(f"Joined SSM group {MCAST_GRP} from source {MCAST_SRC}")

while True:
    data, addr = sock.recvfrom(4096)
    print(f"Received: {data} from {addr}")
```

## Verify Group Membership

```bash
# Check IGMP group memberships reported to kernel:
cat /proc/net/igmp

# Parse in human-readable form:
python3 << 'EOF'
with open('/proc/net/igmp') as f:
    lines = f.readlines()

for line in lines[1:]:  # Skip header
    parts = line.split()
    if len(parts) >= 5:
        iface = parts[1] if len(parts[1]) > 1 else None
        group_hex = parts[3] if len(parts) > 3 else parts[-1]
        if group_hex.strip() not in ('0', 'Querier'):
            try:
                # Convert little-endian hex to IP:
                g = int(group_hex, 16)
                ip = f"{g & 0xff}.{(g >> 8) & 0xff}.{(g >> 16) & 0xff}.{(g >> 24) & 0xff}"
                print(f"  {ip}")
            except ValueError:
                pass
EOF

# Monitor IGMP join/leave events with tcpdump:
tcpdump -i eth0 -n 'igmp'
# Shows IGMP membership reports, queries, and leaves

# Check interface-level multicast groups:
netstat -g
```

## Conclusion

Joining a multicast group on Linux is done via `IP_ADD_MEMBERSHIP` socket option, specifying the group address and interface (use 0.0.0.0 for any interface). The kernel sends an IGMP membership report to inform the local router and switch. Use `ip maddr show` to see current group memberships and `cat /proc/net/igmp` for kernel-level IGMP state. For Source-Specific Multicast (SSM), use `IP_ADD_SOURCE_MEMBERSHIP` to join a `(source, group)` pair - more efficient than ASM as it eliminates unnecessary traffic from other sources. Always call `IP_DROP_MEMBERSHIP` when your application exits to send IGMP leave messages.
