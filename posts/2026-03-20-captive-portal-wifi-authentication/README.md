# How to Set Up Captive Portal Authentication on a WiFi Network

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Captive Portal, WiFi, Authentication, Nodogsplash, OpenWrt, IPv4, Guest Network

Description: Learn how to set up a captive portal on a WiFi network using nodogsplash or pfSense to require users to authenticate before gaining internet access.

---

A captive portal intercepts HTTP traffic from unauthenticated clients and redirects them to a login page before granting internet access. This is standard for guest WiFi networks.

## Architecture Overview

```text
WiFi Client → DHCP → HTTP request → Captive Portal (redirect) → Login page
                                                                     ↓ (authenticated)
                                                              Internet access
```

## Setting Up nodogsplash on OpenWrt

```bash
# Install nodogsplash

opkg update && opkg install nodogsplash

# Enable and start
/etc/init.d/nodogsplash enable
/etc/init.d/nodogsplash start
```

## Basic nodogsplash Configuration

```bash
# /etc/nodogsplash/nodogsplash.conf
GatewayInterface br-guest        # WiFi guest bridge interface
GatewayAddress 192.168.100.1     # Portal IP
GatewayPort 2050

# Authentication method: click-through (no password)
AuthenticateImmediately yes

# Custom splash page
SplashPage splash.html

# Session timeout (seconds)
ClientTimeout 7200

# Bandwidth limits per client (Kbit/s)
DownloadLimit 2048
UploadLimit 512
```

## Custom Splash Page

```html
<!-- /etc/nodogsplash/htdocs/splash.html -->
<!DOCTYPE html>
<html>
<head><title>Guest WiFi</title></head>
<body>
  <h2>Welcome to Guest WiFi</h2>
  <p>By connecting, you agree to our <a href="/terms">Terms of Service</a>.</p>
  <form method="get" action="$authaction">
    <input type="hidden" name="tok" value="$tok">
    <input type="hidden" name="redir" value="$redir">
    <button type="submit">Connect</button>
  </form>
</body>
</html>
```

## pfSense Captive Portal

```text
Services → Captive Portal → Add Zone
  Interface: GUESTNET
  Maximum concurrent connections: 100
  Idle timeout: 30 minutes
  Hard timeout: 480 minutes
  Authentication: Local User Manager / No Authentication (click-through)
  Logout popup window: ✓
  Redirect URL: https://www.example.com
```

## DHCP for Captive Portal Network

```bash
# Isolated DHCP pool for guest network
# /etc/dnsmasq.d/guest.conf
interface=br-guest
dhcp-range=192.168.100.50,192.168.100.200,1h
dhcp-option=3,192.168.100.1
dhcp-option=6,8.8.8.8
```

## Testing the Portal

```bash
# Connect to guest WiFi and verify redirect
curl -v http://example.com    # Should redirect to captive portal IP

# Check nodogsplash status
ndsctl status
ndsctl clients    # List authenticated clients
```

## Key Takeaways

- nodogsplash is a lightweight captive portal for OpenWrt; pfSense has a built-in captive portal for enterprise use.
- Always isolate the captive portal network with a separate VLAN and DHCP pool to prevent guest-to-LAN access.
- Use session timeouts and bandwidth limits to prevent resource abuse on guest networks.
- For password-protected portals, use RADIUS or local user database authentication instead of click-through.
