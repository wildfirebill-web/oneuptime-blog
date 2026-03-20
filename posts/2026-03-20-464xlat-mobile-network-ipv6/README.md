# How to Set Up 464XLAT for Mobile Network IPv6 Transition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: 464XLAT, IPv6, IPv4, Mobile Networks, CLAT, PLAT, Transition

Description: Configure 464XLAT on mobile networks by setting up PLAT (NAT64 on the carrier side) and CLAT (stateless translation on the device side) to allow IPv4 applications to function over IPv6-only mobile networks.

## Introduction

464XLAT (RFC 6877) enables IPv4 applications to work over IPv6-only mobile networks. It uses two translation components: PLAT (Provider-side Translator, a NAT64 gateway) and CLAT (Customer-side Translator, a stateless 1:1 translator on the device). The combination provides end-to-end IPv4 connectivity over an IPv6-only access network.

## Architecture

```
IPv4 App → CLAT (device) → IPv6 network → PLAT (carrier NAT64) → IPv4 Internet
           [stateless]       [IPv6 only]    [stateful NAT64]
```

- **CLAT**: Translates private IPv4 packets from the device to IPv6 using the PLAT prefix (usually 64:ff9b::/96)
- **PLAT**: NAT64 gateway that translates the IPv6 packets back to IPv4 for the public internet

## Setting Up PLAT (NAT64 Gateway)

The PLAT is typically a NAT64 gateway running on the carrier infrastructure. On Linux, TAYGA implements NAT64:

```bash
# Install TAYGA
sudo apt-get install tayga

# /etc/tayga.conf
tun-device nat64
ipv4-addr 192.0.0.1
ipv6-addr 2001:db8:1:ffff::1
prefix 64:ff9b::/96
dynamic-pool 203.0.113.0/24
data-dir /var/lib/tayga
```

```bash
# Create TUN interface and configure routes
sudo tayga --mktun
sudo ip link set nat64 up
sudo ip addr add 192.0.0.0/31 dev nat64
sudo ip addr add 2001:db8:1:ffff::1/128 dev nat64

# Route the NAT64 prefix to TAYGA
sudo ip -6 route add 64:ff9b::/96 dev nat64

# Enable IPv4 forwarding and masquerade
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Start TAYGA
sudo tayga
```

## Setting Up CLAT (Device-Side Translator)

The CLAT runs on the mobile device or CPE. On Linux, `clatd` is a common implementation:

```bash
# Install clatd
sudo apt-get install clatd

# /etc/clatd.conf
clat-dev=clat
plat-prefix=64:ff9b::/96
# IPv4 address for the CLAT interface (RFC 6598 shared address space)
clat-v4-addr=192.0.0.2
```

```bash
# Start clatd
sudo clatd &

# clatd creates a 'clat' interface with the configured IPv4 address
ip addr show clat
# Expected: inet 192.0.0.2/32 scope global clat

# The default IPv4 route is set via the clat interface
ip route show
# Expected: default dev clat
```

## DNS64 Requirement

For 464XLAT to work, the device also needs DNS64 to synthesize AAAA records for IPv4-only destinations:

```bash
# DNS64 server (Unbound)
# /etc/unbound/unbound.conf
server:
    interface: ::0
    do-ip6: yes
    do-ip4: yes

    module-config: "dns64 validator iterator"
    dns64-prefix: 64:ff9b::/96
```

Mobile devices typically use the carrier's DNS64 resolver automatically via DHCPv6 or SLAAC RA DNS options.

## Verifying 464XLAT

```bash
# On a device with CLAT configured:

# Check CLAT interface is up
ip addr show clat

# Ping an IPv4-only host (should work via CLAT/PLAT)
ping 8.8.8.8

# Trace the path — first hop should be CLAT interface
traceroute 8.8.8.8

# Check that IPv6 connectivity is native
ping6 2001:4860:4860::8888

# DNS lookup for IPv4-only domain should return synthesized AAAA
dig AAAA ipv4only.arpa @<dns64-server>
# Should return: 64:ff9b::8.8.8.8 (or similar)
```

## Android and iOS

Mobile operating systems implement CLAT natively:

- **Android**: Built-in CLAT via `clatd` in the kernel; activates automatically on IPv6-only networks when the carrier provides NAT64 prefix (discovered via DNS64 or RFC 7050)
- **iOS**: Also implements CLAT natively since iOS 9; Apple mandates NAT64/DNS64 support for App Store apps

## Firewall on PLAT

```bash
# Allow PLAT to forward translated traffic
sudo ip6tables -A FORWARD -i eth0 -o nat64 -j ACCEPT
sudo ip6tables -A FORWARD -i nat64 -o eth0 -j ACCEPT

# Log translations (optional, useful for troubleshooting)
sudo iptables -t nat -A POSTROUTING -o eth0 -j LOG --log-prefix "CLAT-PLAT: "
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

## Conclusion

464XLAT enables IPv4 applications on IPv6-only mobile networks through two-component translation: CLAT on the device (stateless, 1:1 NAT46) and PLAT in the carrier network (stateful NAT64). Deploy TAYGA as the PLAT for the carrier NAT64 function. On devices, use `clatd` for the CLAT. DNS64 (via Unbound or BIND) synthesizes AAAA records for IPv4-only destinations. Mobile OSes (Android, iOS) implement CLAT natively and activate it automatically on IPv6-only networks.
