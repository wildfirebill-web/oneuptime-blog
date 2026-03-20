# How to Configure DHCPv6 Reservations for Static Assignment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, IPv6, Static IP, Reservations, DUID

Description: Learn how to configure DHCPv6 host reservations to assign fixed IPv6 addresses to specific clients based on their DUID or hardware address.

## Overview

DHCPv6 reservations allow you to pin a specific IPv6 address to a particular client. Unlike DHCPv4 where MAC addresses are used, DHCPv6 reservations are based on the client's **DUID** (DHCP Unique Identifier) or, in some implementations, the hardware address directly.

## Finding a Client's DUID

Before creating a reservation, you need the client's DUID:

```bash
# On the DHCPv6 client (Linux), find the DUID from the lease file

cat /var/lib/dhclient/dhclient6.leases | grep "dhcp6.client-id"

# Or from systemd-networkd
cat /var/lib/systemd/network/*.lease | grep DUID

# View the DUID stored by dhclient directly
cat /var/lib/dhclient/dhclient6.duid
```

## ISC Kea DHCPv6 Reservation Configuration

Kea supports reservations in its JSON configuration under the subnet or globally:

```json
// /etc/kea/kea-dhcp6.conf
{
  "Dhcp6": {
    "subnet6": [
      {
        "subnet": "2001:db8::/64",
        "pools": [
          { "pool": "2001:db8::100 - 2001:db8::1ff" }
        ],
        "reservations": [
          {
            // Reserve a specific address for a client identified by DUID
            "duid": "00:01:00:01:12:34:56:78:aa:bb:cc:dd:ee:ff",
            "ip-addresses": ["2001:db8::10"],
            "hostname": "webserver.example.com"
          },
          {
            // Reserve by hardware address (Kea-specific extension)
            "hw-address": "aa:bb:cc:dd:ee:ff",
            "ip-addresses": ["2001:db8::20"]
          }
        ]
      }
    ]
  }
}
```

## ISC DHCP (dhcpd) Reservation Configuration

For legacy ISC DHCP server:

```bash
# /etc/dhcp/dhcpd6.conf

# Define a host reservation using the client's DUID
host webserver {
    # DUID-LLT type (00:01) followed by hardware type and address
    host-identifier option dhcp6.client-id
        00:01:00:01:12:34:56:78:aa:bb:cc:dd:ee:ff;

    # The fixed IPv6 address to assign
    fixed-address6 2001:db8::10;
}
```

## Understanding DUID Types

| DUID Type | Code | Description |
|-----------|------|-------------|
| DUID-LLT | 00:01 | Link-layer address + timestamp |
| DUID-EN  | 00:02 | Enterprise number + identifier |
| DUID-LL  | 00:03 | Link-layer address only |
| DUID-UUID| 00:04 | UUID (RFC 6355) |

## Verifying a Reservation is Working

After restarting the DHCPv6 server, force the client to renew:

```bash
# Release and re-request the DHCPv6 lease on the client
sudo dhclient -6 -r eth0
sudo dhclient -6 eth0

# Verify the assigned address
ip -6 addr show dev eth0 | grep "2001:db8::10"
```

On Kea, check the lease database:

```bash
# Query Kea's lease database via REST API
curl -s -X POST http://localhost:8000/ \
  -H "Content-Type: application/json" \
  -d '{"command": "lease6-get-by-duid",
       "arguments": {"duid": "00:01:00:01:12:34:56:78:aa:bb:cc:dd:ee:ff"},
       "service": ["dhcp6"]}' | jq .
```

## Global vs Subnet Reservations

In Kea, reservations can be placed at the global level (for all subnets) or within a specific subnet. Global reservations require enabling `reservations-global`:

```json
{
  "Dhcp6": {
    "reservations-global": true,
    "reservations-in-subnet": false,
    "reservations": [
      {
        "duid": "00:01:00:01:12:34:56:78:aa:bb:cc:dd:ee:ff",
        "ip-addresses": ["2001:db8::10"]
      }
    ]
  }
}
```

## Summary

DHCPv6 reservations use the client's DUID to map a fixed IPv6 address. Both Kea and ISC DHCP support this, with Kea offering more flexibility including hardware address matching and global reservations. Always verify the DUID from the client lease file before creating a reservation.
