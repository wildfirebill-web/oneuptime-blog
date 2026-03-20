# How to Enable IPv6 on Your Home Router

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Home Router, SLAAC, DHCPv6, Setup, Consumer Networking

Description: Enable IPv6 on your home router step-by-step, covering DHCPv6, SLAAC, and prefix delegation for typical residential connections.

## Prerequisites

Before enabling IPv6 on your home router, confirm:
1. Your ISP provides IPv6 (check at `test-ipv6.com` from any device)
2. Your router firmware is up to date
3. You know your router's admin panel address (usually `192.168.1.1` or `192.168.0.1`)

## Step 1: Log Into Your Router Admin Panel

Open a browser and navigate to your router's IP. Common defaults:
- Asus: `192.168.1.1`
- TP-Link: `192.168.0.1` or `tplinkwifi.net`
- Netgear: `192.168.1.1` or `routerlogin.net`
- Linksys: `192.168.1.1` or `linksyssmartwifi.com`

## Step 2: Find IPv6 Settings

Navigate to the IPv6 settings section. The exact location varies by router:
- Asus: Advanced Settings → IPv6
- TP-Link: Advanced → IPv6
- Netgear: Advanced → Advanced Setup → IPv6
- Linksys: Connectivity → Local Network → IPv6

## Step 3: Choose the Right IPv6 Connection Type

Most home routers support these IPv6 connection types:

| Connection Type | When to Use |
|----------------|------------|
| Auto-detect / Automatic | Try this first — works with most ISPs |
| DHCPv6 | ISP assigns a fixed IPv6 prefix |
| Native IPv6 (SLAAC) | ISP sends Router Advertisements |
| IPv6 Prefix Delegation | ISP delegates a /56 or /48 block |
| 6to4 / 6in4 / Teredo | Only if ISP doesn't provide native IPv6 |

For most residential ISPs, select **Auto-detect** or **DHCPv6**.

## Step 4: Configure Prefix Delegation for LAN

Enable prefix delegation so your router distributes IPv6 addresses to all home devices:

```
WAN IPv6 Type: DHCPv6
Prefix Delegation: Enable
Prefix Size: /56 or /64 (depends on ISP)

LAN IPv6 Settings:
  Mode: SLAAC + RDNSS (stateless autoconfiguration)
  IPv6 Address Auto-Assign: Enable
```

This automatically assigns IPv6 addresses to phones, laptops, smart TVs, and IoT devices.

## Step 5: Configure IPv6 DNS

Set IPv6 DNS servers. You can use your ISP's DNS or a public DNS over IPv6:

| Provider | IPv6 DNS Addresses |
|---------|-------------------|
| Google | `2001:4860:4860::8888`, `2001:4860:4860::8844` |
| Cloudflare | `2606:4700:4700::1111`, `2606:4700:4700::1001` |
| Quad9 | `2620:fe::fe`, `2620:fe::9` |

## Step 6: Save and Verify

After saving settings, verify IPv6 is working:

1. From a connected device, visit `https://test-ipv6.com`
2. Check your device has a global IPv6 address (not just fe80::):
   - Windows: `ipconfig` (look for IPv6 Address line)
   - Mac: `ifconfig en0 | grep inet6`
   - Android/iOS: Settings → WiFi → tap your network → IPv6 addresses

## Troubleshooting

If IPv6 still doesn't work after configuration:
- Reboot the router and modem
- Check if the ISP requires a specific DHCPv6 Client DUID
- Try changing the prefix delegation size (some ISPs give /60 instead of /56)
- Contact ISP support to confirm IPv6 is provisioned on your account

## Conclusion

Enabling IPv6 on a home router typically requires selecting DHCPv6 or Auto mode in the IPv6 WAN settings and enabling prefix delegation for the LAN. Once configured, all modern devices on your network automatically receive IPv6 addresses and can connect to IPv6 services.
