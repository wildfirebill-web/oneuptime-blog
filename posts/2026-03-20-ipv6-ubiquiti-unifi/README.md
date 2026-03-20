# How to Configure IPv6 on Ubiquiti UniFi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Ubiquiti, UniFi, DHCPv6, SLAAC, Networking

Description: Configure IPv6 on Ubiquiti UniFi networks through the UniFi Network Controller, enabling DHCPv6 prefix delegation and SLAAC for connected devices.

## Introduction

Ubiquiti UniFi provides IPv6 configuration through its Network Controller interface. The setup covers WAN IPv6 connectivity (DHCPv6-PD or PPPoE), LAN network IPv6 mode (SLAAC, stateful, or stateless+DHCPv6), and per-SSID or per-VLAN IPv6 settings.

## Step 1: Configure WAN IPv6 in UniFi Controller

1. Navigate to **Settings > Internet** (or **WAN** in older versions)
2. Under **IPv6 Connection**, select the connection type:
   - **DHCPv6**: ISP assigns an IPv6 address directly
   - **Static IPv6**: Enter your ISP-provided static IPv6 address
   - **PPPoE**: For DSL connections with IPv6

3. For DHCPv6 with Prefix Delegation:
   - Set **IPv6 Connection** to **DHCPv6**
   - Enable **Prefix Delegation Size**: `/56` (check with your ISP)
   - Enable **Rapid Commit**
   - Save

## Step 2: Enable IPv6 on a LAN Network

1. Navigate to **Settings > Networks**
2. Click on the LAN network to edit (or create a new one)
3. Scroll to the **IPv6** section
4. Set **IPv6 Interface Type**:
   - **None**: IPv6 disabled
   - **Static**: Manually enter IPv6 prefix
   - **DHCPv6**: Get prefix from a DHCPv6 server upstream
   - **Prefix Delegation**: Use prefix delegated from WAN (most common for homes/offices)
5. For **Prefix Delegation**, set:
   - **Prefix Delegation Interface**: WAN
   - **Prefix ID**: `1` (for the first /64 from the delegated prefix)
6. Set **IPv6 RA** options:
   - **RA enabled**: On
   - **RA Mode**: Select **Stateless** (SLAAC) or **Stateful DHCPv6**
   - **RA Priority**: Medium (default)
7. Click **Save**

## Step 3: Configure IPv6 DNS

In the network settings:
- Under **DHCP/RA**, set **DNS Server** to your preferred IPv6 DNS:
  - `2606:4700:4700::1111` (Cloudflare)
  - `2001:4860:4860::8888` (Google)
  - Or your internal IPv6 DNS resolver

## Step 4: Verify via CLI on UniFi Gateway

SSH into the UniFi Gateway (USG or UDM/UDM-Pro):

```bash
# Show IPv6 addresses on all interfaces

show interfaces detail | grep -A3 inet6

# Show IPv6 routing table
show ipv6 route

# Show DHCPv6 prefix delegation status
show dhcpv6-pd detail

# Check RA configuration
show interfaces ethernet eth1 detail | grep "router-advert"

# Test IPv6 connectivity
ping6 2606:4700:4700::1111 count 3
```

## Step 5: Verify Client IPv6 Assignment

From a client connected to the UniFi network:

```bash
# Linux client - check for global IPv6 address
ip -6 addr show scope global

# macOS client
ifconfig en0 | grep inet6

# Windows client
ipconfig /all | findstr IPv6

# Verify the address uses the prefix from your delegated pool
# e.g., if ISP delegated 2001:db8::/56 and Prefix ID=1,
# client should have address in 2001:db8:0:1::/64
```

## Troubleshooting Common Issues

**Issue: No IPv6 address on clients**
```bash
# Check RA is being sent from the gateway
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 134" -c 3
# Or use rdisc6
rdisc6 eth0
```

**Issue: DHCPv6-PD not working with ISP**
- Some ISPs require a specific DUID type - check ISP documentation
- Try enabling **Rapid Commit** in WAN settings
- Verify the ISP actually provides IPv6 - test from the WAN interface directly

**Issue: IPv6 works but DNS fails**
- Verify RDNSS is being advertised in RA (`rdisc6 eth0`)
- Ensure DNS server in UniFi settings is a valid IPv6 address
- Check that `systemd-resolved` or the system resolver is accepting RDNSS

## Conclusion

UniFi's Controller-based configuration makes IPv6 setup accessible through a web interface. The key steps are enabling DHCPv6-PD on the WAN and configuring Prefix Delegation on LAN networks. For complex multi-VLAN setups, each VLAN network gets its own Prefix ID (0, 1, 2...) derived from the delegated prefix. The CLI on the gateway provides access to the underlying EdgeOS/VyOS commands for deeper troubleshooting.
