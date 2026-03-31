# How to Configure PRTG for IPv6 Network Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: PRTG, IPv6, Network Monitoring, SNMP, Window, Enterprise Monitoring

Description: Configure PRTG Network Monitor to discover, add, and monitor IPv6-addressed devices using SNMP and ping sensors over IPv6 transport.

---

PRTG Network Monitor supports IPv6 for device discovery, SNMP monitoring, and ping-based availability checks. Configuring PRTG for IPv6 monitoring involves adding IPv6 devices and configuring sensors to use IPv6 transport.

## PRTG IPv6 Prerequisites

```text
Requirements:
- PRTG server needs an IPv6 address (or IPv6 connectivity)
- Windows with IPv6 enabled (Control Panel > Network Adapters > IPv6)
- SNMP agents on target devices configured for IPv6
- DNS AAAA records for devices (optional but recommended)

Verify PRTG IPv6 support:
Setup > System Administration > Core & Probes
Check: "IPv6 available" shows Yes
```

## Adding IPv6 Devices to PRTG

```text
Method 1: Add by IPv6 Address
1. Devices > Add Device
2. IP Address/DNS Name: 2001:db8::router1
3. PRTG recognizes IPv6 addresses automatically
4. Set SNMP Version and Community/Credentials
5. Click OK

Method 2: Add by Hostname (AAAA record)
1. IP Address/DNS Name: router1.example.com
2. PRTG resolves AAAA record for IPv6 transport
3. Ensure DNS returns AAAA record for the hostname
```

## IPv6 Ping Sensor Configuration

```sql
Add Ping Sensor to IPv6 Device:
1. Right-click device > Add Sensor
2. Search for "Ping"
3. Select "Ping" sensor
4. Sensor Name: IPv6 Ping Check
5. Timeout: 10 seconds
6. Packet Size: 32 bytes
7. PRTG will ping the device's IPv6 address

For explicit IPv6 ping:
- Use "Ping IPv6" sensor if available
- Or configure channel with IPv6 flag
```

## SNMP Sensors over IPv6

```sql
SNMP Traffic Sensor for IPv6 Interface:
1. Device > Add Sensor > SNMP Traffic
2. Device: 2001:db8::switch1
3. SNMP Version: v2c or v3
4. Community: public (or SNMPv3 credentials)
5. Select Interfaces to monitor
6. Save

PRTG uses SNMP over IPv6 transport automatically
for devices with IPv6 addresses
```

## PRTG Auto-Discovery for IPv6

```text
Configure IPv6 Auto-Discovery:
1. Group > Add Group > Auto-Discovery
2. IP Range: 2001:db8:0:0::/64
   (PRTG supports /64 subnets for discovery)
3. Discovery Method: ICMP (ping), then SNMP
4. SNMP Settings: Community, version
5. Start Discovery

Note: IPv6 auto-discovery scans are slow due to the large address space
Prefer direct device addition or DNS-based discovery for IPv6
```

## PRTG Custom SNMP Sensor for IPv6 Stats

```xml
<!-- PRTG Custom SNMP OID Sensor template -->
<!-- For IPv6-specific statistics via IP-MIB -->

Sensor: SNMP Custom String OID
OID: 1.3.6.1.2.1.4.31.1.1.3.2
    (ipSystemStatsHCInReceives for IPv6)
Data Type: Integer (Counter)
Channel Name: IPv6 Packets Received

OID: 1.3.6.1.2.1.4.31.1.1.4.2
    (ipSystemStatsHCOutTransmits for IPv6)
Channel Name: IPv6 Packets Transmitted
```

## PRTG Notifications for IPv6 Devices

```text
Configure alerts for IPv6 device issues:
1. Setup > Notifications > Add Notification
2. Trigger: Sensor goes Down
3. Filter: Tags = "ipv6"
4. Action: Send Email / SMS

Tag all IPv6 devices:
- Device Properties > Tags: "ipv6", "core-network"
- Use tags to filter IPv6-specific dashboards and reports
```

## PRTG Remote Probe on IPv6 Network

```text
For monitoring remote IPv6 network segments:
1. Install PRTG Remote Probe on IPv6 segment
2. Probe connects to PRTG Core Server
   - Can use IPv4 or IPv6 for probe-to-core connection
3. Configure probe to monitor local IPv6 devices
4. In PRTG: Setup > Remote Probes > Add Probe

Remote Probe is particularly useful for:
- Monitoring IPv6-only network segments
- Reducing latency for geographically distributed IPv6 sites
```

## Troubleshooting PRTG IPv6 Issues

```text
Common issues:
1. "Unable to ping device" for IPv6 address
   - Verify PRTG server has IPv6 connectivity
   - Check Windows Firewall allows outbound ICMPv6
   - Test: ping -6 2001:db8::device from PRTG server

2. SNMP over IPv6 fails
   - Verify snmpget works: snmpget -v2c -c public udp6:[addr]:161 sysDescr.0
   - Check firewall on target device allows SNMP from PRTG IPv6 address

3. Sensor shows IPv4 instead of IPv6
   - Verify device DNS returns AAAA record, not A record
   - Or enter IPv6 address directly instead of hostname
```

PRTG's native IPv6 support allows monitoring IPv6-addressed devices with the same sensors as IPv4, with auto-discovery being less practical for IPv6 due to the large address space but individual device addition and DNS-based discovery working effectively.
