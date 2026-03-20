# How to Plan IPv6 Addressing for IoT Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IoT, 6LoWPAN, Networking, Security

Description: Design an IPv6 addressing plan for IoT networks, covering device segmentation, 6LoWPAN considerations, security zones, and management of large device populations.

## Introduction

IPv6 is uniquely suited for IoT: its vast address space eliminates the NAT complexity that plagues large IPv4 IoT deployments, and protocols like 6LoWPAN enable IPv6 on even the most constrained devices. Planning IPv6 addressing for IoT requires thinking about device segmentation, security isolation, management access, and the unique characteristics of IoT protocols.

## IoT Network Segmentation

IoT devices should be isolated from the main network. A typical IPv6 IoT addressing plan creates dedicated subnets per device category:

```
Site prefix: 2001:db8:site::/48

IoT zones (using high subnet numbers):
  2001:db8:site:4001::/64   Smart lighting
  2001:db8:site:4002::/64   HVAC and climate control
  2001:db8:site:4003::/64   Security cameras (CCTV)
  2001:db8:site:4004::/64   Access control (badge readers)
  2001:db8:site:4005::/64   Environmental sensors
  2001:db8:site:4010::/64   Industrial control (SCADA)
  2001:db8:site:4020::/64   Medical devices
  2001:db8:site:40ff::/64   Guest IoT / consumer devices
```

## IPv6 for Constrained Devices: 6LoWPAN

6LoWPAN (IPv6 over Low-Power Wireless Personal Area Networks, RFC 4944) adapts IPv6 for IEEE 802.15.4 networks used by Zigbee, Thread, and similar protocols:

```
Key 6LoWPAN features:
  - Header compression (reduce 40-byte IPv6 header to ~4 bytes)
  - Fragmentation for 127-byte 802.15.4 frame size
  - Mesh addressing for multi-hop sensor networks
  - Supports full IPv6 /64 subnets

Addressing:
  - EUI-64 IIDs derived from IEEE EUI-64 device identifiers
  - Each 6LoWPAN PAN gets a /64 subnet
  - A coordinator/border router connects the PAN to the IPv6 network
```

## Network Architecture

```mermaid
graph TD
    INTERNET["IPv6 Internet"] --> FIREWALL["Firewall<br/>(IoT rules)"]
    FIREWALL --> CORE["Core Router<br/>2001:db8:site::/48"]
    CORE --> IOT_GW["IoT Gateway / Border Router"]
    IOT_GW --> WIFI["WiFi IoT<br/>2001:db8:site:4001::/64"]
    IOT_GW --> BLE["BLE Mesh<br/>(via gateway)"]
    IOT_GW --> THREAD["Thread/6LoWPAN<br/>2001:db8:site:4005::/64"]
    THREAD --> SENS1["Sensor 1"]
    THREAD --> SENS2["Sensor 2"]
    THREAD --> SENS3["Sensor N"]
```

## Device Address Assignment

For IoT devices, SLAAC (EUI-64 based) provides a useful advantage — the MAC/EUI-64 address is embedded in the IPv6 address, making it easy to identify devices:

```python
def device_eui64_address(subnet_prefix, eui64_mac):
    """
    Build the IPv6 address for an IoT device using EUI-64.
    Many IoT devices have EUI-64 (not EUI-48) addresses.
    """
    import ipaddress

    # For 8-byte EUI-64, convert directly (no ff:fe insertion needed)
    mac_clean = eui64_mac.replace(":", "").replace("-", "")
    # Flip the U/L bit
    first_byte = int(mac_clean[:2], 16) ^ 0x02
    iid = f"{first_byte:02x}{mac_clean[2:4]}:{mac_clean[4:8]}:{mac_clean[8:12]}:{mac_clean[12:]}"

    net = ipaddress.IPv6Network(subnet_prefix)
    prefix = str(net.network_address)
    return f"{prefix}{iid}"

# Example: temperature sensor with EUI-64 = 00:11:22:ff:fe:33:44:55
subnet = "2001:db8:site:4005::"
eui64 = "00:11:22:ff:fe:33:44:55"
print(device_eui64_address(subnet, eui64))
# Output: 2001:db8:site:4005:0211:22ff:fe33:4455
```

## IoT Firewall Policy

```bash
# Isolate IoT subnets from user networks
# Allow IoT → cloud (specific destinations only)
sudo ip6tables -A FORWARD -s 2001:db8:site:4001::/64 -d <cloud-endpoint>/128 -p tcp --dport 443 -j ACCEPT

# Block IoT → user LAN (IoT should never initiate to users)
sudo ip6tables -A FORWARD -s 2001:db8:site:4001::/64 -d 2001:db8:site:0001::/64 -j DROP

# Allow management → IoT (one-way: management can reach IoT)
sudo ip6tables -A FORWARD -s 2001:db8:site:0001::/64 -d 2001:db8:site:4001::/64 -j ACCEPT

# Block all other IoT inter-subnet traffic
sudo ip6tables -A FORWARD -s 2001:db8:site:4000::/52 -j DROP
```

## DHCPv6 for IoT Devices

Some IoT devices do not support SLAAC and require stateful DHCPv6:

```bash
# dnsmasq: DHCPv6 for IoT subnet
cat /etc/dnsmasq.d/iot-dhcpv6.conf

# dhcp-range=2001:db8:site:4001::100,2001:db8:site:4001::fff,64,12h
# dhcp-option=option6:dns-server,2001:db8:site:0001::53
# dhcp-option=option6:ntp-server,2001:db8:site:0001::123

# Enable RA for IoT subnet (set M flag for stateful only)
# interface eth0.4001 {
#     AdvManagedFlag on;
#     AdvOtherConfigFlag on;
#     prefix 2001:db8:site:4001::/64 {
#         AdvAutonomous off;  # No SLAAC for tightly controlled IoT
#     };
# };
```

## Conclusion

IPv6's address abundance makes it ideal for IoT deployments where device counts can reach thousands per site. Use dedicated /64 subnets per IoT category to enforce security zones. 6LoWPAN enables IPv6 on resource-constrained sensors, and EUI-64 addressing makes device identification straightforward. Always isolate IoT subnets from user and server networks with explicit firewall rules, and consider disabling SLAAC in favor of stateful DHCPv6 where strict device control is required.
