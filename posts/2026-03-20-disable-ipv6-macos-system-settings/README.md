# How to Disable IPv6 on macOS via System Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, macOS, System Settings, Network Configuration, Disable IPv6

Description: Step-by-step guide to disabling IPv6 on macOS using the System Settings GUI for both Wi-Fi and Ethernet connections.

## Disable IPv6 via System Settings (macOS Ventura/Sonoma)

```sql
Steps:
1. Click Apple menu (top-left) → System Settings

2. In the left sidebar, click "Network"

3. Select your active connection:
   - Click "Wi-Fi" and click "Details..."
   - Or click "Ethernet" and click "Details..."

4. Click the "TCP/IP" tab

5. Find "Configure IPv6" dropdown

6. Change from "Automatically" to:
   - "Off" - Disables IPv6 completely on this interface
   - "Link-local only" - Keeps only link-local (fe80::) addressing

7. Click "OK"

8. The change takes effect immediately (no restart required)
```

## Disable IPv6 via System Preferences (macOS Monterey and earlier)

```sql
Steps:
1. Apple menu → System Preferences

2. Click "Network"

3. Select the network interface from the left list
   (Wi-Fi or Ethernet)

4. Click "Advanced..." button (bottom right)

5. Click "TCP/IP" tab

6. "Configure IPv6" → select "Off" or "Link-local only"

7. Click "OK"

8. Click "Apply" in the Network preferences window
```

## Disable on Multiple Interfaces

```text
For dual-stack machines, disable on all interfaces:

1. Repeat the steps above for:
   - Wi-Fi
   - Ethernet (if connected)
   - Any VPN adapters (Cisco VPN, etc.)
   - Thunderbolt Bridge (if present)

Note: Each interface has its own IPv6 setting
```

## Verify IPv6 is Disabled

After disabling via GUI, verify using Terminal:

```bash
# Check IPv6 addresses on Wi-Fi (en0)

ifconfig en0 | grep inet6

# With IPv6 disabled, you should only see (or nothing):
# inet6 fe80::1234:5678%en0 prefixlen 64 scopeid 0x4

# If "Off" was selected, no inet6 lines should appear
# If "Link-local only", only fe80:: should appear

# Test that global IPv6 is gone
ping6 2001:4860:4860::8888
# Should fail: "ping6: UDP connect: No route to host"
```

## Re-enable IPv6

```sql
Steps:
1. System Settings → Network → Details → TCP/IP

2. Configure IPv6 → select "Automatically"

3. Click OK

IPv6 is re-enabled and will acquire addresses via SLAAC or DHCPv6
```

## Summary

Disable IPv6 on macOS via **System Settings → Network → [Interface] Details → TCP/IP → Configure IPv6 → Off**. This takes effect immediately without a restart. For Monterey and earlier, the path is **System Preferences → Network → Advanced → TCP/IP**. To keep link-local-only addressing (needed for some local network features), choose "Link-local only" instead of "Off". Verify with `ifconfig en0 | grep inet6`.
