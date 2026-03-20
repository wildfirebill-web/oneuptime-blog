# How to Use SNMP over IPv6 with Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Python, SNMP, Network Monitoring, Pysnmp

Description: Poll network devices over IPv6 using SNMP with Python's PySNMP library, query IPv6-specific MIBs, and build IPv6 network monitoring tools.

## Install PySNMP

```bash
pip install pysnmp pysnmp-mibs
```

## SNMP GET over IPv6

```python
from pysnmp.hlapi import *

def snmp_get_ipv6(host: str, oid: str,
                  community: str = "public",
                  port: int = 161) -> str | None:
    """
    SNMP GET request to an IPv6 device.
    host: IPv6 address (without brackets)
    """
    error_indication, error_status, error_index, var_binds = next(
        getCmd(
            SnmpEngine(),
            CommunityData(community, mpModel=1),   # mpModel=1 = SNMPv2c
            UdpTransportTarget(
                (host, port),
                timeout=5,
                retries=2,
            ),
            ContextData(),
            ObjectType(ObjectIdentity(oid)),
        )
    )

    if error_indication:
        print(f"SNMP error: {error_indication}")
        return None
    if error_status:
        print(f"SNMP error status: {error_status.prettyPrint()}")
        return None

    for var_bind in var_binds:
        return str(var_bind[1])

# Query sysDescr from an IPv6 device

host = "2001:db8::router"
result = snmp_get_ipv6(host, "1.3.6.1.2.1.1.1.0")
print(f"sysDescr: {result}")
```

## SNMP WALK over IPv6

```python
from pysnmp.hlapi import *

def snmp_walk_ipv6(host: str, oid: str,
                   community: str = "public",
                   port: int = 161) -> list[tuple[str, str]]:
    """SNMP WALK to enumerate a subtree from an IPv6 device."""
    results = []

    for error_indication, error_status, error_index, var_binds in nextCmd(
        SnmpEngine(),
        CommunityData(community),
        UdpTransportTarget((host, port), timeout=5, retries=2),
        ContextData(),
        ObjectType(ObjectIdentity(oid)),
        lexicographicMode=False,   # Stop at end of subtree
    ):
        if error_indication or error_status:
            break
        for var_bind in var_binds:
            oid_str = str(var_bind[0])
            val_str = str(var_bind[1])
            results.append((oid_str, val_str))

    return results

# Walk interface table
host = "2001:db8::switch1"
ifaces = snmp_walk_ipv6(host, "1.3.6.1.2.1.2.2.1")  # ifTable
for oid, val in ifaces[:10]:
    print(f"  {oid}: {val}")
```

## Query IPv6-Specific MIBs

```python
from pysnmp.hlapi import *

# IPv6 MIB OIDs (RFC 2465 / IP-MIB)
IPV6_MIBS = {
    # IP-MIB ipv6IfTable
    "ipv6IfDescr":           "1.3.6.1.2.1.55.1.5.1.2",
    "ipv6IfPhysAddress":     "1.3.6.1.2.1.55.1.5.1.3",
    "ipv6IfOperStatus":      "1.3.6.1.2.1.55.1.5.1.10",

    # IP-MIB ipAddressTable (RFC 4293)
    "ipAddressType":         "1.3.6.1.2.1.4.34.1.4",
    "ipAddressPrefix":       "1.3.6.1.2.1.4.34.1.5",

    # IPv6 address table (older RFC 2465)
    "ipv6AddrAddress":       "1.3.6.1.2.1.55.1.8.1.1",
    "ipv6AddrPfxLength":     "1.3.6.1.2.1.55.1.8.1.2",

    # BGP4+ for IPv6 (Cisco-specific)
    "cbgpPeer2RemoteAs":     "1.3.6.1.4.1.9.9.187.1.2.5.1.11",
}

def get_ipv6_interfaces(host: str, community: str = "public") -> list[dict]:
    """Get IPv6 interface information from a network device."""
    results = []

    # Walk ipv6AddrAddress to find all IPv6 addresses
    addresses = snmp_walk_ipv6(host, "1.3.6.1.2.1.55.1.8.1.1", community)
    prefix_lens = snmp_walk_ipv6(host, "1.3.6.1.2.1.55.1.8.1.2", community)

    for (addr_oid, addr_val), (pl_oid, pl_val) in zip(addresses, prefix_lens):
        results.append({
            "oid": addr_oid,
            "address": addr_val,
            "prefixlen": pl_val,
        })

    return results

# Get IPv6 addresses from router
interfaces = get_ipv6_interfaces("2001:db8::router")
for iface in interfaces:
    print(f"  {iface['address']}/{iface['prefixlen']}")
```

## SNMPv3 over IPv6 (Secure)

```python
from pysnmp.hlapi import *

def snmp_get_v3_ipv6(host: str, oid: str,
                     username: str,
                     auth_key: str,
                     priv_key: str,
                     port: int = 161) -> str | None:
    """SNMPv3 with authentication and encryption over IPv6."""
    error_indication, error_status, error_index, var_binds = next(
        getCmd(
            SnmpEngine(),
            UsmUserData(
                username,
                authKey=auth_key,
                privKey=priv_key,
                authProtocol=usmHMACSHAAuthProtocol,
                privProtocol=usmAesCfb128Protocol,
            ),
            UdpTransportTarget(
                (host, port),
                timeout=10,
                retries=2,
            ),
            ContextData(),
            ObjectType(ObjectIdentity(oid)),
        )
    )

    if error_indication:
        return f"Error: {error_indication}"
    if error_status:
        return f"Error: {error_status.prettyPrint()}"

    for var_bind in var_binds:
        return str(var_bind[1])

# Secure SNMP query over IPv6
result = snmp_get_v3_ipv6(
    host="2001:db8::secure-router",
    oid="1.3.6.1.2.1.1.1.0",
    username="snmpv3user",
    auth_key="authpassphrase123",
    priv_key="privpassphrase123",
)
print(f"sysDescr: {result}")
```

## Bulk Device Polling

```python
import concurrent.futures
from pysnmp.hlapi import *

DEVICES = [
    "2001:db8::router1",
    "2001:db8::switch1",
    "2001:db8::switch2",
]

SYS_OIDS = {
    "sysDescr":   "1.3.6.1.2.1.1.1.0",
    "sysName":    "1.3.6.1.2.1.1.5.0",
    "sysUpTime":  "1.3.6.1.2.1.1.3.0",
}

def poll_device(host: str) -> dict:
    """Poll a single device for system info."""
    result = {"host": host}
    for name, oid in SYS_OIDS.items():
        result[name] = snmp_get_ipv6(host, oid)
    return result

# Poll all devices in parallel
with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    futures = {executor.submit(poll_device, dev): dev for dev in DEVICES}
    for future in concurrent.futures.as_completed(futures):
        data = future.result()
        print(f"\nDevice: {data['host']}")
        print(f"  Name:    {data.get('sysName')}")
        print(f"  UpTime:  {data.get('sysUpTime')}")
```

## Conclusion

PySNMP's `UdpTransportTarget((host, port))` accepts IPv6 addresses directly - pass the bare IPv6 address without brackets. Use SNMPv2c (`CommunityData`) for backward compatibility or SNMPv3 (`UsmUserData`) with SHA/AES for secure polling over IPv6. Query IPv6-specific device information via RFC 2465 MIBs (`1.3.6.1.2.1.55.*` for ipv6IfTable and ipv6AddrTable) and RFC 4293 (`1.3.6.1.2.1.4.34.*` for ipAddressTable). Use `nextCmd` for WALK operations and `concurrent.futures.ThreadPoolExecutor` to poll multiple IPv6 devices in parallel. Prefer SNMPv3 in production - community strings in SNMPv2c are transmitted in cleartext.
