# How IPv6 Privacy Extensions Work on Android

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Privacy Extensions, Android, Mobile, SLAAC, Security

Description: A guide to understanding and verifying IPv6 privacy extensions on Android devices, including address generation behavior, per-network randomization, and privacy implications of Android's IPv6 implementation.

Android has included IPv6 privacy extensions support since Android 4.0 (Ice Cream Sandwich). Modern Android (8.0+) generates a new random interface ID each time the device connects to a network, providing strong privacy protection against cross-network tracking.

## Android's IPv6 Privacy Implementation

Android uses a stricter privacy model than RFC 8981:

```
Android 7.0 and earlier:
  - Enabled RFC 4941 privacy extensions
  - Generated temporary addresses that rotate periodically
  - Used consistent EUI-64 or random address per network

Android 8.0 (Oreo) and later:
  - New random interface ID generated on EACH network connection
  - Even reconnecting to the same Wi-Fi generates a new address
  - No persistent address across connections (maximum privacy)
  - Each network gets a completely different random interface ID
```

## Checking IPv6 Addresses on Android

```bash
# Via ADB (Android Debug Bridge) — requires USB debugging enabled

# Connect phone via USB and enable USB debugging
adb shell ip -6 addr show

# Filter for global addresses
adb shell ip -6 addr show | grep "scope global"

# Example output:
# inet6 2001:db8:a:b:1234:5678:9abc:def0/64 scope global dynamic
#   valid_lft 2591985sec preferred_lft 604785sec

# Note: No EUI-64 pattern (no ff:fe in the interface ID)
# The address is random and will change on next network connection
```

## Verifying Privacy on Android

```bash
# Step 1: Note current IPv6 address
adb shell ip -6 addr show wlan0 | grep "scope global"
# Write down the address

# Step 2: Disconnect from Wi-Fi and reconnect
adb shell svc wifi disable
sleep 5
adb shell svc wifi enable
sleep 15

# Step 3: Check the new IPv6 address
adb shell ip -6 addr show wlan0 | grep "scope global"
# Should be a completely different address

# Verify with a public IPv6 check
adb shell curl -6 https://ipv6.icanhazip.com
```

## Android IPv6 Configuration Per Network

```bash
# Each Wi-Fi SSID gets a different random interface ID
# This prevents tracking across different networks

# Check current Wi-Fi connection
adb shell dumpsys wifi | grep -i "current network\|SSID\|ipv6"

# Check all interfaces
adb shell ifconfig | grep -A 3 "wlan\|rmnet"

# Check routing table for IPv6
adb shell ip -6 route show
```

## Privacy Addresses in Android Apps

For Android app developers who work with IPv6:

```java
// Java/Kotlin: Get device's current IPv6 address
import java.net.NetworkInterface;
import java.net.InetAddress;
import java.net.Inet6Address;
import java.util.Enumeration;

public static String getIPv6Address() {
    try {
        Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
        while (interfaces.hasMoreElements()) {
            NetworkInterface iface = interfaces.nextElement();
            if (!iface.getName().startsWith("wlan") && !iface.getName().startsWith("eth")) {
                continue;
            }
            Enumeration<InetAddress> addresses = iface.getInetAddresses();
            while (addresses.hasMoreElements()) {
                InetAddress addr = addresses.nextElement();
                if (addr instanceof Inet6Address && !addr.isLoopbackAddress()
                        && !addr.isLinkLocalAddress()) {
                    return addr.getHostAddress();
                }
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```

```kotlin
// Kotlin: Check if an address is a temporary privacy address
fun isPrivacyAddress(address: Inet6Address): Boolean {
    val bytes = address.address
    // Check if interface ID matches EUI-64 pattern (ff:fe in bytes 11-12)
    // If NOT EUI-64, likely a privacy address
    return !(bytes[11] == 0xFF.toByte() && bytes[12] == 0xFE.toByte())
}
```

## IPv6 Privacy on Mobile Networks (LTE/5G)

```bash
# On cellular connections, IPv6 address assignment varies by carrier:
# - Many carriers use DHCPv6 Prefix Delegation
# - The /64 prefix changes per session
# - Interface ID may be random or device-specific

# Check cellular interface
adb shell ip -6 addr show rmnet_data0

# Carrier-assigned prefix (will be different each time data session starts)
# Example: 2600:1700:carrier::device-id/64

# For cellular, the carrier assigns the full /64 per session,
# so cross-session tracking is limited by the changing prefix
```

## Checking IPv6 Connectivity and Privacy

```bash
# Full IPv6 connectivity test via ADB
adb shell ping6 -c 3 2001:4860:4860::8888

# DNS AAAA record resolution
adb shell nslookup -type=AAAA google.com

# Check if the device prefers IPv6 over IPv4 (Happy Eyeballs)
adb shell ping6 -c 3 ipv6.google.com

# Verify no EUI-64 address is present
# EUI-64 contains ff:fe in position 11-12 of the interface ID
adb shell ip -6 addr show | grep "ff:fe"
# This should return NOTHING on Android 8.0+
```

## Privacy Limitations on Android

```bash
# Known limitations:
# 1. Some Android OEMs may modify IPv6 behavior
# 2. VPN apps may use their own IPv6 assignment (potentially less private)
# 3. On some carriers, IPv6 prefix is static per SIM/device

# Check if VPN is changing IPv6 behavior
adb shell ip -6 addr show tun0   # OpenVPN
adb shell ip -6 addr show wg0    # WireGuard
# VPN interface IPv6 address may be stable (assigned by VPN server)

# For enterprise Android (work profile), MDM may configure static IPv6
adb shell ip -6 addr show | grep -E "scope global"
```

Android's strong IPv6 privacy model — generating a new random interface ID for each network connection — provides better privacy than the RFC 8981 temporary address model. Users do not need to configure anything; privacy is enabled by default from Android 8.0 onward. The address visible to websites and services changes with every Wi-Fi reconnection, making persistent cross-network tracking via IPv6 address infeasible.
