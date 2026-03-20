# How to Understand IPv6 Multicast for IoT Discovery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Multicast, IoT, mDNS, CoAP, Discovery

Description: Understand how IPv6 multicast enables efficient IoT device discovery and group communication without broadcast flooding, using mDNS, CoAP multicast, and MLD.

## Introduction

IPv6 replaces IPv4 broadcast with multicast, providing more efficient one-to-many communication. For IoT, multicast is essential for device discovery (mDNS, CoAP resource discovery), group commands (turn on all lights in a room), and control plane protocols (Neighbor Discovery, MLD). Unlike broadcast, multicast only reaches devices that have joined the relevant multicast group.

## Key IPv6 Multicast Addresses for IoT

| Address | Scope | Purpose |
|---|---|---|
| `ff02::1` | Link-local | All nodes on the link |
| `ff02::2` | Link-local | All routers (RA destination) |
| `ff02::fb` | Link-local | mDNS (Multicast DNS) |
| `ff02::fd` | Link-local | All RPL nodes |
| `ff02::1:2` | Link-local | All DHCPv6 servers/relays |
| `ff02::1:3` | Link-local | All DHCPv6 servers |
| `ff05::fd` | Site-local | All RPL nodes |
| `ff03::fd` | Realm-local | Matter/Thread mesh-local |

## CoAP Multicast Discovery

CoAP uses `ff02::fd` (All CoAP nodes) for device discovery:

```python
#!/usr/bin/env python3
# coap_discover.py
# Discover CoAP IoT devices using IPv6 multicast

import asyncio
import aiocoap

async def discover_iot_devices():
    """Send CoAP GET to all-CoAP-nodes multicast address to discover devices."""
    protocol = await aiocoap.Context.create_client_context()

    # RFC 6690 CoAP resource discovery via /.well-known/core
    # The CoAP all-nodes multicast address is ff02::fd
    request = aiocoap.Message(
        code=aiocoap.GET,
        uri='coap://[ff02::fd]/.well-known/core',
        mtype=aiocoap.NON  # Non-confirmable for multicast
    )

    discovered_devices = []

    try:
        # Multicast responses come from individual devices
        response = await asyncio.wait_for(
            protocol.request(request).response,
            timeout=5.0
        )
        print(f"Device at {response.remote.hostinfo}:")
        print(f"  Resources: {response.payload.decode()}")
        discovered_devices.append(response.remote.hostinfo)
    except asyncio.TimeoutError:
        print("Discovery timeout - no devices responded")

    return discovered_devices

asyncio.run(discover_iot_devices())
```

## mDNS Discovery for IPv6 IoT Devices

Matter and many IPv6 IoT devices use mDNS (Multicast DNS) on `ff02::fb`:

```bash
# Discover Matter devices via mDNS on IPv6
avahi-browse -a --resolve | grep -i matter
# or
dns-sd -B _matter._tcp
dns-sd -B _hap._tcp    # HomeKit Accessory Protocol

# List all mDNS services on the IPv6 network
avahi-browse -a -r -t

# Monitor mDNS discovery in real time
sudo tcpdump -i eth0 "ip6 and udp port 5353"
```

## Setting Up an mDNS Responder for an IoT Device

```python
#!/usr/bin/env python3
# mdns_iot_device.py
# Register an IoT device with mDNS/DNS-SD for discovery

from zeroconf import Zeroconf, ServiceInfo
import socket
import time

def register_iot_device(
    service_name: str,
    service_type: str,
    port: int,
    ipv6_addr: str,
    properties: dict
):
    """Register an IoT device service via mDNS."""
    zeroconf = Zeroconf()

    # Convert IPv6 address to bytes
    addr_bytes = socket.inet_pton(socket.AF_INET6, ipv6_addr)

    info = ServiceInfo(
        type_=service_type,
        name=f"{service_name}.{service_type}",
        addresses=[addr_bytes],
        port=port,
        properties=properties,
        server=f"{service_name}.local."
    )

    print(f"Registering {service_name} on {ipv6_addr}:{port}")
    zeroconf.register_service(info)

    return zeroconf

# Register a temperature sensor as a CoAP service
zc = register_iot_device(
    service_name="temp-sensor-1",
    service_type="_coap._udp.local.",
    port=5683,
    ipv6_addr="2001:db8:iot:1::sensor1",
    properties={"type": "temperature", "unit": "celsius", "location": "room-1"}
)

try:
    input("Service registered. Press Enter to unregister...\n")
finally:
    zc.unregister_all_services()
    zc.close()
```

## MLD (Multicast Listener Discovery)

IPv6 uses MLD to manage multicast group membership (equivalent to IGMP in IPv4):

```bash
# Check which multicast groups a device has joined
ip -6 maddr show eth0

# Output example:
# inet6 ff02::1:ff00:1 users 1 (Solicited-Node multicast for own address)
# inet6 ff02::1 users 1 (All nodes)
# inet6 ff02::fb users 1 (mDNS)
# inet6 ff02::1:3 users 1 (DHCPv6 all-servers)

# Manually join a multicast group for testing
sudo ip -6 maddr add ff02::1234 dev eth0

# Monitor MLD reports
sudo tcpdump -i eth0 "icmp6 and (ip6[40] == 143 or ip6[40] == 130)"
# Type 130 = MLDv1 Query, 143 = MLDv2 Report
```

## Group Commands with IPv6 Multicast

For IoT scenarios like "turn off all lights in Zone 1":

```python
# Send a CoAP multicast command to all lights in Zone 1
# Zone 1 uses a site-local multicast address ff05::zone1
async def turn_off_zone_1():
    protocol = await aiocoap.Context.create_client_context()

    # Register multicast group for Zone 1 lights
    # ff05::1 = site-local all-nodes (or use a custom group)
    request = aiocoap.Message(
        code=aiocoap.PUT,
        uri='coap://[ff05::1]/lights/zone1',
        payload=b'{"on": false}',
        mtype=aiocoap.NON  # Non-confirmable for multicast
    )
    # Devices in Zone 1 joined this multicast group during commissioning
    await protocol.request(request).response
```

## Conclusion

IPv6 multicast is foundational to IoT efficiency: CoAP discovery uses `ff02::fd`, mDNS uses `ff02::fb`, Neighbor Discovery uses `ff02::2`, and DHCPv6 uses `ff02::1:2`. Understanding these multicast groups and how to monitor them (via `ip -6 maddr show` and `tcpdump`) is essential for troubleshooting IoT device discovery and group communication in IPv6 networks.
