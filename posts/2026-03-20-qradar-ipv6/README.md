# How to Configure IBM QRadar for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: QRadar, IPv6, SIEM, Security Analytics, IBM, Log Sources, Threat Detection

Description: Configure IBM QRadar SIEM to collect, parse, and analyze IPv6 network traffic logs including log source configuration, custom properties, and building block rules for IPv6.

## QRadar IPv6 Support Overview

QRadar supports IPv6 in multiple areas:
- Log source collection over IPv6 (syslog, JDBC, etc.)
- IPv6 address parsing in events and flows
- Network hierarchy definitions for IPv6 subnets
- Rules and building blocks using IPv6 addresses
- Flow collector (QFlow) capturing IPv6 traffic

## Configuring Log Sources via IPv6

```
QRadar Admin Console → Log Sources → Add Log Source

Log Source Type: Linux OS / Cisco ASA / etc.
Protocol: Syslog
Log Source Identifier: 2001:db8:device::1   ← IPv6 address

# For UDP syslog from IPv6 devices:
Admin → System Configuration → Firewall Access
Add: Allow UDP 514 from 2001:db8:device::/48

# Command line: verify QRadar listens on IPv6 for syslog
ss -6 -u -l -n | grep 514
# UNCONN ... :::514
```

## Network Hierarchy: IPv6 Subnets

```
QRadar Admin → Network Hierarchy → Add Network

Name: Corp_IPv6_Network
Network: 2001:db8:corp::/48
Group: Internal Networks

Name: DMZ_IPv6
Network: 2001:db8:dmz::/48
Group: DMZ

Name: Guest_IPv6
Network: 2001:db8:guest::/48
Group: Guest Networks

# This allows QRadar to classify traffic as:
# Local-Local, Local-Remote, Remote-Local, Remote-Remote
# based on IPv6 source/destination subnets
```

## Custom Event Properties for IPv6

```
QRadar Admin → Custom Event Properties → Add Property

# Extract IPv6 source from custom log format
Property Name: IPv6_Source_Address
Field Type: IP
Regex: SRC=([0-9a-fA-F:]+:[0-9a-fA-F:]+)
Group: 1
Enabled: Yes

# Extract IPv6 /64 prefix
Property Name: IPv6_Source_Prefix64
Field Type: Text
Regex: SRC=((?:[0-9a-fA-F]{0,4}:){4})
Group: 1

# IPv6 destination
Property Name: IPv6_Dest_Address
Field Type: IP
Regex: DST=([0-9a-fA-F:]+:[0-9a-fA-F:]+)
Group: 1
```

## Building Block Rules for IPv6

```
QRadar → Rules → Add Building Block Rule

# BB: IPv6 Internal Source
Name: BB:IPv6_Internal_Source
Test: when the source IP is contained in any of:
    - Corp_IPv6_Network
    - DMZ_IPv6

# BB: IPv6 External Source
Name: BB:IPv6_External_Source
Test: when the source IP is NOT contained in:
    - Corp_IPv6_Network
    - DMZ_IPv6
    - Guest_IPv6
    - AND source IP is an IPv6 address

# BB: IPv6 Link-Local Observed
Name: BB:IPv6_LinkLocal_In_Logs
Test: when source IP matches pattern "fe80:*"
Note: Link-local should not appear in routed logs
```

## Detection Rules

```
# Rule: IPv6 Scanning Detection
Name: IPv6 Port Scan from External
Description: Detect external IPv6 host scanning multiple ports

Tests:
  AND the event(s) were detected by one or more of: Firewall Log Sources
  AND the source IP matches BB:IPv6_External_Source
  AND the source IP is the same across events
  AND the destination port count is greater than 20
  WITHIN 60 seconds

Response: Email security team, create offense

# Rule: IPv6 Tunnel Protocol Detected
Name: IPv6 Tunnel (Teredo/6to4) Traffic
Tests:
  AND the source IP matches pattern "2002:*" (6to4)
     OR source IP matches pattern "2001:0:*" (Teredo range)
  AND local network

Response: Low priority alert, track for volume
```

## AQL Queries for IPv6 Analysis

```sql
-- AQL: Find IPv6 traffic to external destinations
SELECT
    "sourceip",
    "destinationip",
    "destinationport",
    "protocolid",
    COUNT(*) as event_count
FROM events
WHERE LOGSOURCETYPENAME(devicetype) = 'Linux OS'
  AND "sourceip" LIKE '2001:db8:%'
  AND "destinationip" NOT LIKE '2001:db8:%'
  AND "destinationip" LIKE '%:%'  -- IPv6 pattern
LAST 24 HOURS
GROUP BY "sourceip", "destinationip", "destinationport", "protocolid"
ORDER BY event_count DESC

-- AQL: IPv6 flows — top talkers
SELECT
    sourceip,
    destinationip,
    SUM(sourcebytes) as bytes_out,
    SUM(destinationbytes) as bytes_in,
    COUNT(*) as flows
FROM flows
WHERE sourceip NOT LIKE '%.%'  -- Exclude IPv4
LAST 1 HOURS
GROUP BY sourceip, destinationip
ORDER BY bytes_out DESC
LIMIT 20

-- AQL: Find NDP-related events
SELECT *
FROM events
WHERE QIDNAME(qid) ILIKE '%neighbor%'
   OR QIDNAME(qid) ILIKE '%ndp%'
   OR "message" ILIKE '%ICMPv6 Type 135%'
LAST 1 HOURS
```

## QFlow: IPv6 Flow Collection

```
# QFlow collector configuration for IPv6 support
# Admin → Data Collection → Flow Sources

Flow Source: Core_Switch_IPv6
Type: NetFlow v9 (supports IPv6)
Host: 2001:db8:core::1
Port: 2055

# Enable IPv6 in flow parsing
# Admin → System Settings → QFlow Configuration
# Check: Enable IPv6 Flow Parsing = Yes

# Verify IPv6 flows are being collected
# In QRadar Network Activity:
# Filter: Source/Destination IP = 2001:db8:*
```

## Reporting

```
# Create scheduled report: IPv6 Security Summary
QRadar → Reports → Add Report

Report Template: Network Activity Overview
Filters:
  - IP Version: IPv6
  - Time Period: Last 24 Hours

Include sections:
  - Top IPv6 Source IPs
  - Top Destination Ports for IPv6
  - IPv6 Traffic by Network Group (using Network Hierarchy)
  - IPv6 Offense Count

Schedule: Daily at 08:00, Email to security-team@example.com
```

## Conclusion

QRadar IPv6 support spans log collection (IPv6 syslog), flow analysis (NetFlow v9/IPFIX), and detection rules. Define IPv6 subnets in Network Hierarchy to enable Local/Remote classification for IPv6 sources and destinations. Custom Event Properties extract IPv6 addresses from non-standard log formats using regex. Building Blocks encapsulate IPv6 subnet membership tests for reuse across multiple rules. AQL queries use `LIKE '%:%'` as a simple IPv6 filter when IP version metadata is unavailable. Enable IPv6 flow parsing in QFlow settings to include IPv6 traffic in the Network Activity view alongside IPv4.
