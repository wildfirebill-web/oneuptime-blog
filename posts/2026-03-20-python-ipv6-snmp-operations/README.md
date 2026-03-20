# How to Use Python for IPv6 SNMP Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv6, SNMP, Network Management, Pysnmp, Monitoring

Description: Use Python with pysnmp to perform SNMP queries over IPv6 to monitor and manage network devices.

## SNMP and IPv6

SNMP (Simple Network Management Protocol) works over IPv6 using the same operations (GET, GETNEXT, GETBULK, SET) but with IPv6 transport. Most managed network devices support SNMP over both IPv4 and IPv6.

```bash
pip install pysnmp
```

## Basic SNMP GET over IPv6

Query a network device using SNMPv2c over IPv6:

```python
from pysnmp.hlapi import (
    getCmd, SnmpEngine, CommunityData,
    UdpTransportTarget, Udp6TransportTarget,
    ContextData, ObjectType, ObjectIdentity
)

def snmp_get_ipv6(
    host: str,
    community: str,
    oid: str,
    port: int = 161
) -> str:
    """
    Perform an SNMP GET over IPv6.
    host: IPv6 address of the target device
    """
    # Use Udp6TransportTarget for IPv6
    transport = Udp6TransportTarget((host, port), timeout=5, retries=3)

    error_indication, error_status, error_index, var_binds = next(
        getCmd(
            SnmpEngine(),
            CommunityData(community, mpModel=1),  # SNMPv2c
            transport,
            ContextData(),
            ObjectType(ObjectIdentity(oid))
        )
    )

    if error_indication:
        raise RuntimeError(f"SNMP error: {error_indication}")
    if error_status:
        raise RuntimeError(f"PDU error: {error_status}")

    for var_bind in var_binds:
        return str(var_bind[1])

# Query sysDescr from a router over IPv6

sysDescr = snmp_get_ipv6(
    host="2001:db8:router::1",
    community="public",
    oid="1.3.6.1.2.1.1.1.0"
)
print(f"System Description: {sysDescr}")
```

## SNMP WALK over IPv6 (getBulk)

Walk an OID tree using GETBULK for efficiency:

```python
from pysnmp.hlapi import (
    bulkCmd, SnmpEngine, CommunityData,
    Udp6TransportTarget, ContextData,
    ObjectType, ObjectIdentity
)

def snmp_walk_ipv6(host: str, community: str, oid: str) -> list[tuple]:
    """Walk an OID subtree via SNMP GETBULK over IPv6."""
    results = []

    for error_indication, error_status, error_index, var_binds in bulkCmd(
        SnmpEngine(),
        CommunityData(community),
        Udp6TransportTarget((host, 161)),
        ContextData(),
        0,  # nonRepeaters
        10, # maxRepetitions
        ObjectType(ObjectIdentity(oid)),
        lexicographicMode=False
    ):
        if error_indication:
            break
        if error_status:
            break
        for var_bind in var_binds:
            results.append((str(var_bind[0]), str(var_bind[1])))

    return results

# Walk interface table
interfaces = snmp_walk_ipv6(
    host="2001:db8::1",
    community="public",
    oid="1.3.6.1.2.1.2.2.1"  # ifTable
)
for oid, value in interfaces[:10]:
    print(f"  {oid} = {value}")
```

## SNMPv3 over IPv6

SNMPv3 with authentication and privacy over IPv6:

```python
from pysnmp.hlapi import (
    getCmd, SnmpEngine, UsmUserData,
    Udp6TransportTarget, ContextData,
    ObjectType, ObjectIdentity,
    usmHMACMD5AuthProtocol, usmDESPrivProtocol
)

def snmp_v3_get_ipv6(
    host: str,
    username: str,
    auth_key: str,
    priv_key: str,
    oid: str
) -> str:
    """SNMPv3 GET over IPv6 with auth and encryption."""
    error_indication, error_status, error_index, var_binds = next(
        getCmd(
            SnmpEngine(),
            UsmUserData(
                username,
                authKey=auth_key,
                privKey=priv_key,
                authProtocol=usmHMACMD5AuthProtocol,
                privProtocol=usmDESPrivProtocol
            ),
            Udp6TransportTarget(("2001:db8::1", 161)),
            ContextData(),
            ObjectType(ObjectIdentity(oid))
        )
    )

    if error_indication or error_status:
        raise RuntimeError(f"SNMP v3 error: {error_indication or error_status}")

    return str(var_binds[0][1])
```

## Monitoring IPv6 Interface Statistics

```python
# Interface MIB OIDs
IF_MIB = {
    "ifDescr":            "1.3.6.1.2.1.2.2.1.2",
    "ifOperStatus":       "1.3.6.1.2.1.2.2.1.8",
    "ifInOctets":         "1.3.6.1.2.1.2.2.1.10",
    "ifOutOctets":        "1.3.6.1.2.1.2.2.1.16",
}

def get_interface_stats(host: str, community: str, if_index: int) -> dict:
    """Get interface statistics from a network device via IPv6 SNMP."""
    stats = {}
    for name, base_oid in IF_MIB.items():
        oid = f"{base_oid}.{if_index}"
        try:
            stats[name] = snmp_get_ipv6(host, community, oid)
        except Exception as e:
            stats[name] = f"Error: {e}"
    return stats

# stats = get_interface_stats("2001:db8::1", "public", 1)
# print(stats)
```

## Bulk Network Discovery via SNMP over IPv6

```python
import asyncio
from typing import Optional

async def probe_snmp_host(host: str, community: str) -> Optional[str]:
    """Check if a host responds to SNMP over IPv6."""
    loop = asyncio.get_event_loop()
    try:
        result = await asyncio.wait_for(
            loop.run_in_executor(
                None,
                lambda: snmp_get_ipv6(host, community, "1.3.6.1.2.1.1.1.0")
            ),
            timeout=5.0
        )
        return result
    except Exception:
        return None

async def discover_snmp_hosts(hosts: list[str], community: str) -> dict:
    """Discover which hosts respond to SNMP over IPv6."""
    tasks = {host: probe_snmp_host(host, community) for host in hosts}
    results = await asyncio.gather(*tasks.values())
    return {host: result for host, result in zip(hosts, results) if result}
```

## Conclusion

Python's pysnmp library supports SNMP over IPv6 using `Udp6TransportTarget`. The API is identical to IPv4 SNMP - only the transport target class changes. SNMP over IPv6 is essential for network monitoring of IPv6-only devices and for management systems that need to reach devices with IPv6 addresses in dual-stack networks.
