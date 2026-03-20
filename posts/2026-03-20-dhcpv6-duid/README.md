# Understanding DHCPv6 DUID (DHCP Unique Identifier)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, IPv6, DUID, Networking, Identity, IAID

Description: Learn what a DHCPv6 DUID is, the different DUID types (DUID-LLT, DUID-EN, DUID-LL, DUID-UUID), how they are generated, and how to manage them on Linux and Windows clients.

---

A DUID (DHCP Unique Identifier) is the DHCPv6 equivalent of a MAC address for DHCP purposes. It uniquely identifies a DHCPv6 client or server and is used by the server to track leases and enforce host-specific configuration. Unlike MAC addresses in DHCPv4, DUIDs persist across interface changes and reboots.

---

## DUID Types

DHCPv6 defines four DUID types, each with a different format:

### DUID-LLT (Type 1) - Link-Layer Address Plus Time

The most common type. Generated from a hardware address and the time the DUID was first created.

```text
Format: 2-byte type | 2-byte hardware type | 4-byte timestamp | N-byte link-layer address
Example: 00:01:00:01:28:4f:a1:b2:00:11:22:33:44:55
```

### DUID-EN (Type 2) - Vendor-Based Enterprise Number

Used by vendors and appliances. Based on the IANA Private Enterprise Number.

```text
Format: 2-byte type | 4-byte enterprise number | variable identifier
Example: 00:02:00:00:09:bf:...
```

### DUID-LL (Type 3) - Link-Layer Address Only

Similar to DUID-LLT but without the timestamp. Used when stable time is unavailable.

```text
Format: 2-byte type | 2-byte hardware type | N-byte link-layer address
Example: 00:03:00:01:00:11:22:33:44:55
```

### DUID-UUID (Type 4) - UUID-Based

Based on the system's UUID (RFC 6355). Common on UEFI systems.

```text
Format: 2-byte type | 16-byte UUID
Example: 00:04:00:01:02:03:04:05:06:07:08:09:0a:0b:0c:0d:0e:0f
```

---

## Viewing Your DUID on Linux

```bash
# dhclient stores DUID in lease file

cat /var/lib/dhcpv6/dhcp6c_duid

# Or in the dhclient lease file
cat /var/lib/dhcp/dhclient6.leases | grep duid

# systemd-networkd DUID
cat /var/lib/systemd/network/lldp/*.duid 2>/dev/null

# wide-dhcpv6 DUID file
xxd /var/lib/dhcpv6/dhcp6c_duid
```

### Using ip Command to Find MAC (basis for DUID-LL)

```bash
ip link show eth0 | grep "link/ether"
# link/ether 00:11:22:33:44:55 brd ff:ff:ff:ff:ff:ff
```

---

## Viewing Your DUID on Windows

```powershell
# Show DUID used by Windows DHCP client
ipconfig /all | Select-String "DHCP"

# More detailed DUID info via registry
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\services\TCPIP6\Parameters" | Select-Object Dhcpv6DUID

# PowerShell - decode DUID bytes
$duid = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\services\TCPIP6\Parameters").Dhcpv6DUID
[System.BitConverter]::ToString($duid)
```

---

## Configuring Static DUID on Linux (systemd-networkd)

You can pin a DUID in `systemd-networkd` to ensure consistent lease assignment:

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
DHCP=ipv6

[DHCPv6]
DUID=link-layer-time
# Or: DUID=link-layer
# Or: DUID=uuid
# Or: DUID=vendor
```

### Custom DUID via dhclient

```text
# /etc/dhcp/dhclient6.conf
interface "eth0" {
    send dhcp6.client-id 00:01:00:01:28:4f:a1:b2:00:11:22:33:44:55;
}
```

---

## Using DUID for Host Reservations on the Server

### ISC DHCP (dhcpd)

```text
# /etc/dhcp/dhcpd6.conf
host webserver01 {
    host-identifier option dhcp6.client-id 00:01:00:01:28:4f:a1:b2:00:11:22:33:44:55;
    fixed-address6 2001:db8::10;
}
```

### Kea DHCPv6

```json
{
  "Dhcp6": {
    "reservations": [
      {
        "duid": "00:01:00:01:28:4f:a1:b2:00:11:22:33:44:55",
        "ip-addresses": ["2001:db8::10"],
        "hostname": "webserver01.corp.example.com"
      }
    ]
  }
}
```

---

## DUID vs IAID

| Concept | Purpose | Scope | Stability |
|---------|---------|-------|-----------|
| DUID | Identifies the DHCPv6 client/server | Device-wide | Persistent across reboots |
| IAID | Identifies a specific interface/address pool | Per-interface | May change on interface rebind |

A full DHCPv6 client identifier is the combination of DUID + IAID.

---

## Best Practices

1. **Never change a DUID** on production systems - it will cause lease re-assignment
2. **Use DUID-LL** on virtual machines for portability (no timestamp dependency)
3. **Record DUIDs in your IPAM** for all servers with reserved addresses
4. **Test reservations** in staging before production deployment
5. **Use DUID-UUID** on UEFI systems for guaranteed uniqueness

---

## Conclusion

DUIDs are the cornerstone of DHCPv6 client identity. Understanding the four DUID types and how to view, pin, and use them for reservations is essential for managing IPv6 address assignments at scale.

---

*Manage and monitor your IPv6 network with [OneUptime](https://oneuptime.com) - full-stack observability with IPv6 support.*
