# How to Configure IPv6 on Linksys Home Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Linksys, Home Router, DHCPv6, Velop, Consumer Networking

Description: Configure IPv6 on Linksys routers including EA and MR series and Velop mesh systems through the web admin interface.

## Supported Linksys Routers

IPv6 is supported on:
- Linksys EA7500, EA8300, EA9500
- Linksys MR7350, MR9600
- Linksys Velop MX4200, MX10600, WHW03

## Accessing Admin Interface

Navigate to `http://192.168.1.1` or `http://linksyssmartwifi.com`. Default credentials are admin/admin (change these immediately if you haven't).

## Step 1: Access IPv6 Settings

1. Log into the admin panel
2. Navigate to **Connectivity** (or **Advanced** on older firmware)
3. Click **Local Network** tab
4. Scroll to **IPv6** section

## Step 2: Enable IPv6

Toggle IPv6 to **Enabled** and select the connection type:

```
IPv6: Enabled

IPv6 Connection Type:
  ○ Automatic Configuration - SLAAC
  ○ DHCPv6 (recommended for most ISPs)
  ○ Static IPv6
  ○ PPPoE
  ○ 6to4
  ○ Manual
```

## Step 3: DHCPv6 Configuration

For DHCPv6 (most common for cable/fiber ISPs):

```
IPv6 Connection Type: DHCPv6

DHCPv6 Settings:
  ☑ Request IPv6 Address
  ☑ Request IPv6 Prefix (Delegation)
  Prefix Size: /56

DUID:
  Type: DUID-LLT (default)
```

## Step 4: LAN IPv6 Settings

Configure how local devices receive IPv6:

```
IPv6 LAN Settings:

IPv6 Address Assignment:
  ☑ Assign IPv6 Addresses via SLAAC

DNS Assignment:
  ☑ Send DNS to clients via RDNSS
  DNS 1: 2001:4860:4860::8888
  DNS 2: 2001:4860:4860::8844

☑ Enable Router Advertisements
RA Interval: 200 seconds (default)
```

## Linksys Velop Configuration

For Velop mesh systems, use the Linksys app or web interface:

**Via Linksys App:**
1. Open the Linksys app
2. Go to **WiFi Settings** → **Advanced Settings**
3. Tap **IPv6**
4. Toggle to **On**
5. Select **Automatic** (recommended)

**Via Web Interface (`http://192.168.1.1`):**
1. Go to **Connectivity → Local Network**
2. Scroll down to **IPv6** section
3. Select **Automatic** or **DHCPv6**

## Step 5: Verify IPv6 Status

Check the Status page on Linksys admin:

```
Status → Local Network

Expected IPv6 info:
  Connection Type: DHCPv6
  IPv6 Address: 2001:xxxx:xxxx:xxxx::1/64
  Gateway: fe80::xxxx
  Primary DNS: 2001:xxxx::1
```

From a connected laptop:

```bash
# macOS/Linux
ip -6 addr show scope global
ping6 -c 4 ipv6.google.com

# Check external IPv6
curl -6 https://ipv6.icanhazip.com
```

## Troubleshooting Linksys IPv6

**DHCPv6 not connecting:**
- Check if your ISP requires a specific DUID format
- Try SLAAC/Auto first — some ISPs use RA instead of DHCPv6
- Factory reset and reconfigure if IPv6 was previously set to a different mode

**LAN devices get fe80 only (no global IPv6):**
- Verify Router Advertisements are enabled
- Check if another device on LAN is also sending RAs (rogue RA)
- Disable and re-enable IPv6 on the router to trigger RA resend

**PPPoEv6 connection fails:**
- Verify ISP credentials support IPv6
- Check that the DSL modem is in bridge mode
- Contact ISP to confirm PPPoEv6 is enabled on the account

## Conclusion

Linksys routers support IPv6 through both the web admin panel and the Linksys mobile app. Select DHCPv6 or Auto depending on your ISP's method, enable Prefix Delegation for LAN device addressing, and verify operation using ping6 tests or the `test-ipv6.com` website.
