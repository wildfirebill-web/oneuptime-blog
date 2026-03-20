# How to Configure IPv6 on Netgear Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Netgear, Router, DHCPv6, SLAAC, Home Network

Description: Configure IPv6 on Netgear routers including Nighthawk and Orbi series, enabling DHCPv6 prefix delegation and SLAAC for home and small business networks.

## Introduction

Netgear routers support IPv6 across the Nighthawk, Orbi, and business-grade lineup. IPv6 is configured through the web admin interface at `http://routerlogin.net` or `http://192.168.1.1`. This guide covers setting up IPv6 for ISP connectivity and LAN distribution.

## Step 1: Access IPv6 Settings

1. Log in to the router admin interface
2. Navigate to **Advanced > Advanced Setup > IPv6**
3. If you don't see IPv6, check under **Internet Setup > IPv6**

## Step 2: Configure WAN IPv6

Select the appropriate connection type based on your ISP:

**For Auto Config (most common - ISP provides DHCPv6):**
1. Select **Auto Config** from the IP Address Type dropdown
2. Check the status after saving - you should see a WAN IPv6 address assigned

**For DHCPv6 with Prefix Delegation:**
1. Select **DHCP** from the IP Address Type dropdown
2. Enable **Use This Router's MAC Address**: On (some ISPs bind to MAC)
3. **Prefix Length**: Set to the prefix length your ISP delegates (`/56` or `/48`)
4. Click **Apply**

**For Static IPv6:**
1. Select **Fixed** from the IP Address Type dropdown
2. Enter:
   - IPv6 Address: (your ISP-provided address)
   - Subnet Prefix Length: (your ISP-provided prefix length)
   - Default IPv6 Gateway: (your ISP-provided gateway)
   - Primary/Secondary DNS: IPv6 DNS addresses
3. Click **Apply**

## Step 3: Configure LAN IPv6

1. Under the LAN section of the IPv6 settings:
2. Set **LAN Address Type**:
   - **Auto Config**: Router assigns SLAAC prefix to LAN automatically
   - **Fixed**: Manually specify the LAN IPv6 prefix

3. For RA settings:
   - **Enable Router Advertisement**: Checked
   - **Router Advertisement Address Type**: **STATELESS** (for SLAAC)
   - **Advertisement Life Time**: 1800 seconds
   - **DNS Server**: Your preferred IPv6 DNS or `::` (use ISP DNS)

4. Click **Apply**

## Step 4: Verify the Configuration

In the router's status page:

1. Go to **Advanced > Administration > Router Status** or **Status > Overview**
2. Look for the **IPv6** section showing:
   - WAN IPv6 Address (should be a global unicast address)
   - LAN IPv6 Prefix (should be a /64 derived from the delegated prefix)

## Step 5: Test from a Client

```bash
# From a Windows PC on the network

ipconfig /all
# Look for "IPv6 Address" under the adapter - should show a global address

# Test connectivity
ping -6 2606:4700:4700::1111

# From a Linux/macOS client
ip -6 addr show scope global
ping6 2606:4700:4700::1111
curl -6 https://ifconfig.me
```

## Troubleshooting Common Issues

**Issue: "Auto Detect" not finding IPv6 connection**

```bash
# Check if the WAN port is receiving IPv6 via SLAAC or DHCPv6
# Connect a laptop directly to the modem/ONT and test:
sudo dhclient -6 -v eth0
# If this works, the router needs to be configured for DHCPv6
```

**Issue: WAN IPv6 address is assigned but LAN clients don't get IPv6**

This typically means Router Advertisements are not being sent:
1. Ensure **Enable Router Advertisement** is checked in LAN settings
2. Verify **Advertisement Address Type** is STATELESS or STATEFUL
3. Try power-cycling the router after making changes

**Issue: IPv6 connectivity but no DNS over IPv6**

1. Manually set IPv6 DNS servers in the router settings
2. Verify the DNS server is reachable: `ping6 2606:4700:4700::1111`
3. Check that RDNSS is included in RA from a client using `rdisc6 eth0`

## Netgear Orbi Mesh Specifics

For Orbi systems:
1. Log in at `http://orbilogin.com`
2. Navigate to **Advanced > Advanced Setup > IPv6**
3. The main Orbi router handles IPv6; satellites inherit settings automatically
4. Ensure all satellites have updated firmware matching the router

## Conclusion

Netgear routers provide a functional if somewhat variable IPv6 experience depending on the model and firmware version. The Auto Config or DHCP with prefix delegation modes cover the majority of ISP connectivity scenarios. After enabling IPv6 on the WAN, set the LAN to STATELESS (SLAAC) mode for the simplest client configuration, and verify that both the WAN and LAN sections show valid IPv6 addresses in the router status page.
