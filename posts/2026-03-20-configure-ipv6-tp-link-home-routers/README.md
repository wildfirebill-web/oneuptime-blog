# How to Configure IPv6 on TP-Link Home Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, TP-Link, Home Router, DHCPv6, Consumer Networking, Setup

Description: Configure IPv6 on TP-Link routers including Archer series and Deco mesh systems through the web admin interface.

## Supported TP-Link Routers

IPv6 is supported on most modern TP-Link Archer routers:
- Archer AX73, AX6000, AX11000
- Archer C7, C9, C2300
- Deco M4, M5, X60, XE75

Older models may have limited IPv6 support - update firmware first.

## Accessing the Admin Interface

Navigate to `http://192.168.0.1` or `http://tplinkwifi.net` in your browser. Use the admin password you set during initial setup.

## Step 1: Navigate to IPv6 Settings

1. Log into the router admin page
2. Click **Advanced** at the top navigation
3. Select **IPv6** from the left menu

## Step 2: Choose WAN Connection Type

TP-Link supports these IPv6 WAN types:

| Type | Use Case |
|------|---------|
| Auto Detect | TP-Link auto-selects the method (recommended first try) |
| Dynamic IPv6 | ISP provides IPv6 via DHCPv6 |
| Static IPv6 | ISP provides a fixed IPv6 address |
| PPPoEv6 | DSL connections with PPPoE |
| 6to4 | Legacy tunnel if ISP has no IPv6 |
| Pass-Through | Bridge IPv6 to a single device |

For most home users, select **Auto Detect** or **Dynamic IPv6**.

## Step 3: Dynamic IPv6 Configuration

If using Dynamic IPv6 (DHCPv6):

```text
WAN Type: Dynamic IPv6

Connection Type:
  ☑ Get IPv6 Address Dynamically
  ☑ Get IPv6 DNS Dynamically

Prefix Delegation:
  ☑ Enable IPv6 Prefix Delegation
  Prefix Length: /56 (or as ISP provides)
```

## Step 4: Configure LAN IPv6

In the LAN IPv6 settings:

```text
LAN IPv6 Assignment:
  Mode: SLAAC (Stateless Address Auto-Configuration)

Router Advertisement:
  ☑ Enable RA

DNS Assignment:
  Primary DNS: 2001:4860:4860::8888
  Secondary DNS: 2606:4700:4700::1111
```

## Step 5: Verify IPv6 Status

After saving, check the IPv6 Status page:

```text
Expected output:
  WAN IPv6 Status: Connected
  WAN IPv6 Address: 2001:xxxx:xxxx:1::1/64
  Delegated Prefix: 2001:xxxx:xxxx::/56

  LAN IPv6:
    LAN Address: 2001:xxxx:xxxx:1::1/64
    SLAAC: Active
```

## TP-Link Deco Mesh System

For Deco systems, IPv6 is configured through the Deco mobile app:

1. Open the Deco app
2. Tap **More** → **Advanced** → **IPv6**
3. Toggle IPv6 **On**
4. Select connection type (usually **Auto**)
5. Tap **Save**

The Deco app does not expose as many settings as the web interface but covers the essentials.

## Testing IPv6 on TP-Link

From a connected device:

```bash
# Windows

ping -6 ipv6.google.com

# Mac/Linux
ping6 ipv6.google.com

# Web test
# Visit https://test-ipv6.com in browser
```

## Troubleshooting TP-Link IPv6

**Auto Detect says "Not Connected":**
- Manually select Dynamic IPv6 instead of Auto Detect
- Verify ISP provides IPv6 - check at `test-ipv6.com` before router config
- Reboot modem (especially if it's a separate modem/ONT)

**LAN devices don't get IPv6:**
- Ensure RA is enabled in LAN settings
- Check that IPv6 firewall is not blocking internal traffic
- Try toggling off/on IPv6 to force RA resend

## Conclusion

TP-Link routers make IPv6 configuration accessible through their web admin interface. The Auto Detect option handles most ISP configurations automatically. For reliable results, use Dynamic IPv6 explicitly with Prefix Delegation enabled, and configure SLAAC on the LAN side for automatic device address assignment.
