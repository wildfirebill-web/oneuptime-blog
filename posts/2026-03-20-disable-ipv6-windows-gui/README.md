# How to Disable IPv6 on Windows via GUI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Windows, GUI, Network Configuration, Disable IPv6

Description: Step-by-step guide to disabling IPv6 on Windows using the graphical user interface through Network Adapter Properties and Network and Sharing Center.

## Method 1: Network Adapter Properties

This is the most direct GUI method to disable IPv6 per adapter:

```sql
Steps:
1. Press Win + R, type: ncpa.cpl, press Enter
   (Opens Network Connections)

2. Right-click the network adapter
   (e.g., "Ethernet", "Wi-Fi", "Local Area Connection")

3. Select Properties

4. In the "This connection uses the following items:" list,
   find "Internet Protocol Version 6 (TCP/IPv6)"

5. UNCHECK the checkbox next to it

6. Click OK

7. No restart required - IPv6 is disabled immediately on that adapter
```

## Method 2: Network and Sharing Center

```text
Steps:
1. Click Start → Settings → Network & Internet
   OR right-click the network icon in the system tray

2. Click "Network and Sharing Center"

3. Click "Change adapter settings" in the left panel

4. Right-click the adapter → Properties

5. Uncheck "Internet Protocol Version 6 (TCP/IPv6)"

6. Click OK
```

## Method 3: Windows Settings (Modern UI)

On Windows 10/11 with the new Settings app:

```text
Steps:
1. Start → Settings → Network & Internet

2. Click on your connection type (Ethernet or Wi-Fi)

3. Click on the connected network name

4. Scroll down to find network adapter properties
   Note: In newer Windows 11, direct IPv6 disable may
   require the classic Control Panel method above

5. For IPv6 toggle, use the classic ncpa.cpl method
```

## Verifying IPv6 is Disabled via GUI

```text
Steps:
1. Open Command Prompt (Win + R → cmd)

2. Run:
   ipconfig /all

3. Look at the disabled adapter's output
   - You should NOT see "IPv6 Address" entries
   - You may still see "Link-local IPv6 Address : fe80::..."
     if only the binding was disabled but not the protocol

4. To confirm no IPv6 traffic:
   ping -6 google.com
   Should fail with: "Ping request could not find host"
```

## What GUI Disable Does vs Registry Disable

| Method | Effect | Restart Needed? |
|--------|--------|-----------------|
| Uncheck adapter binding | Disables IPv6 on that adapter only | No |
| Registry DisabledComponents=0xFF | Disables all IPv6 system-wide | Yes |
| Both methods combined | Most complete disable | Yes (registry) |

## Re-enabling IPv6 via GUI

```text
Steps:
1. Open ncpa.cpl (Win + R → ncpa.cpl)

2. Right-click adapter → Properties

3. CHECK "Internet Protocol Version 6 (TCP/IPv6)"

4. Click OK

IPv6 is re-enabled immediately. No restart required.
```

## Important Considerations

```text
Microsoft notes:
- Disabling IPv6 is NOT recommended as Windows components
  rely on IPv6 internally (HomeGroup, DirectAccess, etc.)

- Even with IPv6 disabled at the adapter level, the loopback
  adapter (::1) and some internal IPv6 communication may remain

- For enterprise environments, use Group Policy to manage
  IPv6 settings consistently across multiple machines

- If troubleshooting IPv6 issues, consider using
  "Diagnose" option rather than disabling entirely
```

## Summary

Disable IPv6 on Windows via GUI by opening **Network Connections** (`ncpa.cpl`), right-clicking the adapter, selecting Properties, and unchecking **Internet Protocol Version 6 (TCP/IPv6)**. This takes effect immediately without a restart and is per-adapter. For system-wide disable including loopback tunnels, use the registry method (`DisabledComponents=0xFF`). Re-enable by checking the checkbox again.
