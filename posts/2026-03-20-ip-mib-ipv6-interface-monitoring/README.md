# How to Use IP-MIB for IPv6 Interface Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IP-MIB, SNMP, IPv6, Network Monitoring, RFC 4293, OIDs, MIB

Description: Use the IP-MIB (RFC 4293) to monitor IPv6 addresses, routing table entries, and IP statistics on network devices and servers via SNMP.

---

The IP-MIB (RFC 4293) unifies IPv4 and IPv6 management in a single MIB structure, providing the `ipAddressTable` for address information and `ipNetToPhysicalTable` for neighbor mapping across both IP versions.

## IP-MIB Structure for IPv6

```
IP-MIB Objects for IPv6:
- ipAddressTable        : All IPv4 and IPv6 addresses
- ipAddressEntry        : Per-address information
  - ipAddressAddrType   : Address type (ipv6=2, ipv6z=4)
  - ipAddressAddr       : The IPv6 address
  - ipAddressIfIndex    : Interface index
  - ipAddressType       : unicast/anycast/broadcast
  - ipAddressPrefix     : Link prefix (OID reference)
  - ipAddressOrigin     : manual/dhcp/linklayer/random/other
  - ipAddressStatus     : preferred/deprecated/invalid/inaccessible
- ipNetToPhysicalTable  : IPv6 neighbor table
- ipSystemStatsTable    : Per-IP version statistics
```

## Querying ipAddressTable

```bash
# Get all IPv6 addresses on device
snmpwalk -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipAddressTable

# Get IPv6 addresses only (type 2 = ipv6)
snmpwalk -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipAddressAddr | grep "Hex-STRING\|IpAddress"

# Get address origins
snmpwalk -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipAddressOrigin

# Results:
# 1 = other
# 2 = manual (static)
# 4 = dhcp
# 5 = linklayer (SLAAC)
# 6 = random

# Get address status
snmpwalk -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipAddressStatus

# Results:
# 1 = preferred
# 2 = deprecated
# 3 = invalid
# 4 = inaccessible
# 5 = unknown
```

## IPv6 System Statistics via ipSystemStatsTable

```bash
# Get IPv6 packet statistics
snmpwalk -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipSystemStatsTable

# Specific IPv6 stats (ipVersion = 2 for IPv6)
snmpget -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipSystemStatsHCInReceives.ipv6

snmpget -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipSystemStatsHCOutTransmits.ipv6

# In/Out discards
snmpget -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipSystemStatsInDiscards.ipv6

snmpget -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipSystemStatsOutDiscards.ipv6
```

## IPv6 Neighbor Table (ipNetToPhysicalTable)

```bash
# Get IPv6 neighbor table
snmpwalk -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipNetToPhysicalTable

# Get neighbor MAC addresses
snmpwalk -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipNetToPhysicalPhysAddress

# Get neighbor state
snmpwalk -v2c -c public udp6:[2001:db8::device]:161 \
  IP-MIB::ipNetToPhysicalState

# State values:
# 1=reachable, 2=stale, 3=delay, 4=probe, 5=invalid, 6=unknown, 7=incomplete
```

## Python Script for IP-MIB IPv6 Monitoring

```python
#!/usr/bin/env python3
# ipv6_mib_monitor.py

from pysnmp.hlapi import *

def get_ipv6_addresses(host_ipv6, community='public'):
    """Retrieve IPv6 address table from device via SNMP."""
    ip_address_table_oid = '1.3.6.1.2.1.4.34.1'  # ipAddressTable

    addresses = []
    for (errorInd, errorStatus, errorIndex, varBinds) in nextCmd(
        SnmpEngine(),
        CommunityData(community, mpModel=1),
        Udp6TransportTarget((host_ipv6, 161)),
        ContextData(),
        ObjectType(ObjectIdentity(ip_address_table_oid)),
        lexicographicMode=False
    ):
        if errorInd:
            break
        for varBind in varBinds:
            oid_str = str(varBind[0])
            # Parse IPv6 addresses from OID
            if '1.3.6.1.2.1.4.34.1' in oid_str:
                addresses.append({
                    'oid': oid_str,
                    'value': str(varBind[1])
                })

    return addresses

if __name__ == '__main__':
    device = '2001:db8::router1'
    addrs = get_ipv6_addresses(device)
    for a in addrs[:10]:
        print(a)
```

## Comparing ipAddressTable with System Data

```bash
# Get addresses from SNMP
snmpwalk -v2c -c public udp6:[2001:db8::localhost]:161 \
  IP-MIB::ipAddressAddr > /tmp/snmp_ipv6_addrs.txt

# Get addresses directly from system
ip -6 addr show scope global | grep "inet6" > /tmp/system_ipv6_addrs.txt

# Compare to verify SNMP accuracy
diff /tmp/snmp_ipv6_addrs.txt /tmp/system_ipv6_addrs.txt
```

The IP-MIB's unified approach to IPv4 and IPv6 management through the `ipAddressTable` and `ipSystemStatsTable` simplifies monitoring by using the same OID structure for both protocols, with the address type field distinguishing IPv6 entries from IPv4.
