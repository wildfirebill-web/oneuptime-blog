# Understanding DHCPv6 IAID (Identity Association Identifier)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, IPv6, IAID, Networking, IA_NA, IA_PD, Identity Association

Description: Learn what a DHCPv6 IAID is, how it differs from a DUID, and how Identity Associations (IA_NA, IA_TA, IA_PD) work to assign IPv6 addresses and prefixes to clients.

---

In DHCPv6, an IAID (Identity Association Identifier) is a 32-bit number that uniquely identifies a collection of IPv6 addresses or prefixes assigned to a specific interface on a DHCPv6 client. While the DUID identifies the client device, the IAID identifies a specific interface or address group on that device.

---

## What is an Identity Association (IA)?

An Identity Association (IA) is the binding between a DHCPv6 client's interface and the addresses or prefixes assigned by the server. There are three IA types:

| IA Type | Name | Purpose |
|---------|------|---------|
| IA_NA | Identity Association for Non-temporary Addresses | Standard IPv6 address assignment |
| IA_TA | Identity Association for Temporary Addresses | Short-lived privacy addresses |
| IA_PD | Identity Association for Prefix Delegation | Delegating a prefix to a client router |

---

## DUID vs IAID

| Concept | Scope | Stability | Purpose |
|---------|-------|-----------|---------|
| DUID | Entire device | Persistent | Identifies the client/server |
| IAID | Per interface | Per-interface | Identifies an address group |

A client with two interfaces (eth0, eth1) has one DUID but two IAIDs (one per interface).

---

## How IAIDs Are Generated

### Linux (systemd-networkd)

systemd-networkd generates the IAID from the interface index by default:

```bash
# View interface index
ip link show eth0 | head -1
# 2: eth0: ...

# The IAID is typically the interface index (decimal to hex)
# eth0 with index 2 → IAID = 0x00000002
```

### Linux (dhclient)

dhclient generates IAIDs from the interface's hardware address:

```bash
# View dhclient leases with IAID
cat /var/lib/dhcp/dhclient6.leases | grep ia-na
```

### Windows

Windows generates IAIDs automatically. View them via:

```powershell
# View IAID in use
netsh interface ipv6 show addresses
# The "Dhcp" prefix addresses are associated with an IAID internally
```

---

## Configuring IAID in systemd-networkd

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
DHCP=ipv6

[DHCPv6]
# Pin the IAID value for this interface
IAID=1
```

### Why Pin the IAID?

If `systemd-networkd` regenerates the IAID (e.g., after hardware changes), the server sees it as a new client and issues a new address. Pinning the IAID ensures stable address assignments.

---

## Configuring IAID in dhclient

```text
# /etc/dhcp/dhclient6.conf
interface "eth0" {
    # IA_NA with IAID 1
    send ia-na 1;
    # IA_PD with IAID 2 (for prefix delegation)
    send ia-pd 2;
}

id-assoc na 1 {
};

id-assoc pd 2 {
    prefix-interface eth1 {
        sla-id 1;
        sla-len 8;
    };
};
```

---

## IA_NA Example — Non-Temporary Address

```text
# DHCPv6 message containing IA_NA
IA_NA:
  IAID: 0x00000001
  T1: 3600 seconds (renewal timer)
  T2: 5400 seconds (rebind timer)
  IA Address:
    Address: 2001:db8::10
    Preferred lifetime: 7200
    Valid lifetime: 14400
```

---

## IA_PD Example — Prefix Delegation

An IA_PD allows a client (e.g., a CPE router) to receive an entire prefix:

```text
IA_PD:
  IAID: 0x00000002
  T1: 3600
  T2: 5400
  IA Prefix:
    Prefix: 2001:db8:1::/48
    Preferred lifetime: 7200
    Valid lifetime: 14400
```

---

## Server-Side IAID Handling in Kea

Kea DHCPv6 automatically tracks IAID per client:

```json
{
  "Dhcp6": {
    "subnet6": [
      {
        "subnet": "2001:db8::/32",
        "pools": [
          { "pool": "2001:db8::100-2001:db8::200" }
        ],
        "pd-pools": [
          {
            "prefix": "2001:db8:1::",
            "prefix-len": 48,
            "delegated-len": 56
          }
        ]
      }
    ]
  }
}
```

---

## Troubleshooting IAID Issues

```bash
# Capture DHCPv6 traffic and inspect IAID values
sudo tcpdump -i eth0 -vv udp port 546 or udp port 547

# In Wireshark, filter:
# dhcpv6.iaaddr.iaid

# Check systemd-networkd IAID
journalctl -u systemd-networkd | grep -i iaid

# Force IAID regeneration by removing lease cache
rm /var/lib/systemd/network/*.lease
sudo systemctl restart systemd-networkd
```

---

## Best Practices

1. **Pin IAIDs in production** to ensure stable address assignments after reboots or hardware changes
2. **Use separate IAIDs** for each interface on multi-homed systems
3. **Use IA_PD** only on gateway/CPE devices that need prefix delegation
4. **Document DUID+IAID pairs** in your IPAM for server-side reservation management
5. **Test IAID persistence** by rebooting a client and verifying it receives the same address

---

## Conclusion

IAIDs are the per-interface half of DHCPv6 client identity — paired with the DUID to form the full client identifier for lease tracking. Understanding IAIDs is essential when configuring prefix delegation, multi-interface hosts, or stable address reservations.

---

*Monitor your IPv6 address assignments and infrastructure with [OneUptime](https://oneuptime.com).*
