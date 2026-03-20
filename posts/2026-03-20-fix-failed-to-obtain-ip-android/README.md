# How to Fix 'Failed to Obtain IP Address' on Android WiFi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Android, WiFi, DHCP, IP Address, Troubleshooting, Mobile

Description: Learn how to fix the 'Failed to obtain IP address' error on Android WiFi by troubleshooting DHCP issues, router configuration, and Android network settings.

## What Causes "Failed to Obtain IP Address"?

Android shows this error when it cannot complete the DHCP process after connecting to a WiFi network. Common causes:
- DHCP server pool exhausted
- Router firewall blocking DHCP broadcast
- Incorrect security settings (wrong password causing auth loop)
- Android's DHCP client failing to get a response
- MAC address filtering blocking the device
- IP address conflict

## Step 1: Basic Fixes on Android

**Forget and Reconnect:**
1. Settings → WiFi → Long-press the network
2. Forget network
3. Reconnect and enter the password carefully

**Toggle WiFi:**
```text
Settings → WiFi → Toggle OFF (10 seconds) → Toggle ON
```

**Airplane Mode Toggle:**
```text
Settings → Enable Airplane Mode (5 seconds) → Disable
```

## Step 2: Set a Static IP on Android

If DHCP consistently fails, set a static IP:

1. Settings → WiFi → Long-press the network → Modify Network
2. Show advanced options
3. IP settings: Change **DHCP** to **Static**
4. Enter:
   - **IP address**: 192.168.1.150 (unused address)
   - **Gateway**: 192.168.1.1 (your router IP)
   - **Network prefix length**: 24
   - **DNS 1**: 8.8.8.8
   - **DNS 2**: 8.8.4.4
5. Save

## Step 3: Check Router-Side DHCP Settings

On the router:
```text
# Log into router (usually 192.168.1.1 or 192.168.0.1)

# Check DHCP settings:
# - Pool range: should have available addresses
# - Max clients: should not be exceeded
# - MAC filtering: Android device MAC should not be blocked
```

**Check DHCP pool capacity:**
```bash
# On a Linux-based router
cat /var/lib/misc/dnsmasq.leases | wc -l    # Current leases
grep "dhcp-range" /etc/dnsmasq.conf         # Max pool size

# If pool is 50 addresses but 50 devices are connected, expand:
dhcp-range=192.168.1.50,192.168.1.250,12h
```

## Step 4: Check Android MAC Randomization

Android 10+ uses random MAC addresses by default, which can cause issues:
- Router may block unknown MACs
- DHCP reservations won't work

```sql
Settings → WiFi → Select network → Privacy
Change from "Use randomized MAC" to "Use device MAC"
```

Or on older Android:
```text
Developer Options → WiFi → Disable MAC randomization
```

## Step 5: Clear Android Network Settings

```text
Settings → General Management → Reset → Reset Network Settings
```

This resets all WiFi, Bluetooth, and mobile network settings. You'll need to reconnect to all WiFi networks.

## Step 6: Check Android DNS Settings

Some routers block DNS queries from unknown devices:

1. Settings → WiFi → Long-press network → Modify
2. Show advanced → Change DNS to Custom
3. Enter DNS 1: 8.8.8.8, DNS 2: 1.1.1.1

Or use Private DNS:
```text
Settings → Network & Internet → Advanced → Private DNS
Enter: dns.google
```

## Step 7: Router Debugging

On the router, capture DHCP traffic while Android is connecting:

```bash
# On Linux-based router
sudo tcpdump -i wlan0 port 67 or port 68 -n

# Look for:
# DHCP DISCOVER from Android MAC (should appear when connecting)
# DHCP OFFER from router (should appear in response)
# DHCP REQUEST from Android
# DHCP ACK from router

# If DISCOVER appears but no OFFER: router DHCP is misconfigured
# If OFFER appears but no ACK: Android rejected the offer (wrong subnet?)
```

## Conclusion

"Failed to obtain IP address" on Android is most commonly resolved by setting a static IP or expanding the router's DHCP pool. Check for MAC randomization (Android 10+) that may be triggering MAC-based filtering on the router. If a static IP works but DHCP fails, the router's DHCP service has an issue - capture packets with `tcpdump port 67 or 68` on the router to identify where the DORA sequence breaks down.
