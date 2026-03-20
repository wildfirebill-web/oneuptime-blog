# How to Configure Splunk for IPv6 Log Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Splunk, Log Analysis, SIEM, Network Security

Description: Configure Splunk to receive IPv6 syslog data, extract IPv6 address fields, and build searches and dashboards for IPv6 traffic analysis and security monitoring.

## Introduction

Splunk can receive logs from IPv6-addressed sources via UDP/TCP syslog inputs, and its search language (SPL) supports regex extraction and comparison for IPv6 addresses. This guide covers Splunk Universal Forwarder IPv6 configuration, field extraction, and SPL queries for IPv6 analysis.

## Step 1: Configure Splunk Inputs for IPv6

```ini
# $SPLUNK_HOME/etc/system/local/inputs.conf

# Listen on all IPv6 interfaces for syslog

[udp://:::]
connection_host = dns
sourcetype = syslog
index = network
disabled = false

[tcp://:::]
connection_host = dns
sourcetype = syslog
index = network
disabled = false

# Alternate: specify exact IPv6 address
[udp://[2001:db8::10]:514]
connection_host = ip
sourcetype = syslog
index = network
```

## Step 2: Configure Universal Forwarder to Send over IPv6

```ini
# $SPLUNK_HOME/etc/system/local/outputs.conf on the forwarder

[tcpout]
defaultGroup = indexers

[tcpout:indexers]
# Send to Splunk indexer via IPv6
server = [2001:db8::indexer]:9997
```

## Step 3: Field Extraction for IPv6 Addresses

```ini
# $SPLUNK_HOME/etc/system/local/transforms.conf

[extract-nginx-ipv6]
REGEX = ^(?<client_ip>[0-9a-fA-F:\.]+) - (?<user>\S+) \[(?<http_time>[^\]]+)\] "(?<http_method>\S+) (?<uri>\S+)
SOURCE_KEY = _raw
FORMAT = client_ip::$1 user::$2 http_time::$3 http_method::$4 uri::$5

[extract-sshd-ipv6]
REGEX = from (?<src_ip>[0-9a-fA-F:\.]{3,45}) port (?<src_port>\d+)
SOURCE_KEY = _raw
```

```ini
# props.conf
[source::/var/log/nginx/access.log]
TRANSFORMS-nginx = extract-nginx-ipv6

[syslog]
TRANSFORMS-sshd = extract-sshd-ipv6
```

## Step 4: SPL Queries for IPv6 Analysis

```splunk
# Find all events with IPv6 source addresses
index=network | regex client_ip="[0-9a-fA-F:]{3,39}:[0-9a-fA-F]{0,4}"

# Top IPv6 source IPs
index=network | regex client_ip=":"
| top client_ip limit=20

# IPv6 traffic volume over time
index=network | regex client_ip=":"
| timechart count span=1h by client_ip limit=10

# Find specific IPv6 subnet (prefix match)
index=network
| where match(client_ip, "^2001:db8:")
| stats count by client_ip

# IPv6 vs IPv4 traffic split
index=network
| eval ip_version=if(match(client_ip, ":"), "IPv6", "IPv4")
| timechart count by ip_version

# Failed SSH logins from IPv6
index=os sourcetype=linux_secure "Failed password"
| rex "from (?<src_ip>[0-9a-fA-F:\.]+) port"
| where match(src_ip, ":")
| stats count by src_ip
| sort -count
| head 20
```

## Step 5: IPv6 Security Dashboard Searches

```splunk
# Brute force detection: IPv6 sources with > 10 failed logins in 5min
index=os sourcetype=linux_secure "Failed password"
| rex "from (?<src_ip>[0-9a-fA-F:\.]+) port"
| where match(src_ip, ":")
| bucket _time span=5m
| stats count by _time, src_ip
| where count > 10
| sort -count

# New IPv6 sources never seen before (requires history baseline)
index=network earliest=-1h latest=now | regex client_ip=":"
| stats min(_time) as first_seen by client_ip
| where first_seen > relative_time(now(), "-1h")

# IPv6 scanning behavior (many destination ports from one source)
index=firewall | regex src_ip=":"
| stats dc(dest_port) as unique_ports by src_ip
| where unique_ports > 50
| sort -unique_ports
```

## Step 6: IPv6 CIDR Lookup Table

```csv
# $SPLUNK_HOME/etc/apps/network/lookups/ipv6_subnets.csv
network,description,category
2001:db8::/32,Documentation prefix,documentation
fe80::/10,Link-local,link_local
fc00::/7,Unique local,ula
::1/128,Loopback,loopback
2001::/32,Teredo,tunnel
2002::/16,6to4,tunnel
```

```splunk
# Enrich logs with subnet information
| inputlookup ipv6_subnets.csv
| eval prefix_bits=tonumber(split(network,"/")[1])
```

## Conclusion

Splunk handles IPv6 log collection by binding syslog inputs to `[:::]` (all IPv6 interfaces) and configuring Universal Forwarders to send to IPv6 indexers. Field extractions using REGEX transforms in `transforms.conf` pull IPv6 addresses from various log formats. SPL's `match()`, `where`, and `regex` commands enable flexible IPv6 filtering, and `timechart` provides time-series analysis by IPv6 source - all without requiring native CIDR support, unlike Elasticsearch's `ip` field type.
