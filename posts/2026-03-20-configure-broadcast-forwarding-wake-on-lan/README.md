# How to Configure Broadcast Forwarding for Wake-on-LAN

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Wake-on-LAN, Broadcast, Linux, Cisco, Network Administration

Description: Configure directed broadcast forwarding on a router or Linux system to send Wake-on-LAN magic packets to computers on remote subnets.

## Introduction

Wake-on-LAN (WoL) relies on a broadcast UDP magic packet sent to port 9. This works easily within a single subnet, but reaching machines on remote subnets requires either directed broadcast forwarding on the router or a WoL relay agent.

## How Wake-on-LAN Works

A magic packet consists of 6 bytes of `FF` followed by the target MAC address repeated 16 times, sent as a UDP broadcast:

```
FF FF FF FF FF FF
AA BB CC DD EE FF  (×16)
```

The packet is sent to the subnet's directed broadcast address (e.g., `192.168.2.255`) or `255.255.255.255`, on UDP port 9 (or port 7/40000).

## Option 1: Enable Directed Broadcast on Cisco Router

Directed broadcast must be explicitly enabled on the interface facing the target subnet:

```
! Enable directed broadcast on the interface facing 192.168.2.0/24
interface GigabitEthernet0/2
 ip directed-broadcast
```

Now a packet sent from any subnet to `192.168.2.255:9` will be forwarded into that subnet as a broadcast.

## Option 2: Linux IP Helper (ip forward-protocol)

On a Linux router, use `iptables` to forward UDP port 9 broadcasts:

```bash
# Enable IP forwarding
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Forward WoL packets arriving on eth0 to the target subnet broadcast
sudo iptables -t nat -A PREROUTING -i eth0 \
  -d 255.255.255.255 -p udp --dport 9 \
  -j DNAT --to-destination 192.168.2.255

sudo iptables -A FORWARD -d 192.168.2.255 -p udp --dport 9 -j ACCEPT
```

## Option 3: WoL Relay with wakeonlan Tool

If you do not want to modify router settings, run a relay script on a host that is already on the target subnet:

```bash
# Install wakeonlan
sudo apt install wakeonlan

# Send a magic packet directly from a host on the target subnet
wakeonlan -i 192.168.2.255 AA:BB:CC:DD:EE:FF
```

You can call this from any machine via SSH:

```bash
# SSH to a Linux host on subnet 192.168.2.0/24 and send WoL from there
ssh user@192.168.2.1 "wakeonlan -i 192.168.2.255 AA:BB:CC:DD:EE:FF"
```

## Option 4: Send Magic Packet with Python

```python
#!/usr/bin/env python3
import socket
import struct

def wake_on_lan(mac: str, broadcast: str = "255.255.255.255", port: int = 9):
    """Send a Wake-on-LAN magic packet to the given MAC address."""
    mac_bytes = bytes.fromhex(mac.replace(":", "").replace("-", ""))
    magic = b'\xff' * 6 + mac_bytes * 16

    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as sock:
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
        sock.sendto(magic, (broadcast, port))
        print(f"Magic packet sent to {mac} via {broadcast}:{port}")

# Send to local subnet
wake_on_lan("AA:BB:CC:DD:EE:FF", "255.255.255.255")

# Send to a specific remote subnet's directed broadcast
wake_on_lan("AA:BB:CC:DD:EE:FF", "192.168.2.255")
```

## Verifying with tcpdump

```bash
# On the target subnet — confirm the magic packet arrives
sudo tcpdump -i eth0 -n -X "udp dst port 9" | head -40
```

The hex dump should show `FF FF FF FF FF FF` followed by the MAC repeated 16 times.

## Conclusion

For cross-subnet WoL, the cleanest option is enabling directed broadcast on the router interface facing the target segment. If router changes are not possible, a relay host or SSH tunnel achieves the same result without exposing directed broadcast broadly.
