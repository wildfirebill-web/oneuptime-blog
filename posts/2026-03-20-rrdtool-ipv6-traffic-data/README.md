# How to Configure RRDtool for IPv6 Traffic Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RRDtool, IPv6, Traffic Data, SNMP, Network Monitoring, Round-Robin Database

Description: Use RRDtool to collect, store, and graph IPv6 traffic data from network devices via SNMP, creating time-series databases for IPv6 interface statistics.

---

RRDtool (Round-Robin Database tool) provides time-series storage and graphing. Building custom IPv6 monitoring with RRDtool involves collecting SNMP data from IPv6 devices and storing it in RRD databases for graphing.

## Installing RRDtool

```bash
# Ubuntu/Debian

sudo apt install rrdtool librrds-perl -y

# Verify installation
rrdtool --version

# Python bindings
sudo apt install python3-rrdtool -y
# Or: pip install rrdtool
```

## Creating RRD Database for IPv6 Traffic

```bash
# Create RRD for IPv6 interface traffic
# Store: input/output bytes, with 5-min intervals
rrdtool create /var/lib/rrd/ipv6_eth0.rrd \
  --step 300 \
  DS:in_bytes:COUNTER:600:0:U \
  DS:out_bytes:COUNTER:600:0:U \
  DS:in_packets:COUNTER:600:0:U \
  DS:out_packets:COUNTER:600:0:U \
  RRA:AVERAGE:0.5:1:576 \
  RRA:AVERAGE:0.5:12:336 \
  RRA:AVERAGE:0.5:288:365 \
  RRA:MAX:0.5:1:576

# Verify RRD creation
rrdtool info /var/lib/rrd/ipv6_eth0.rrd
```

## Collecting IPv6 Traffic Data via SNMP

```bash
#!/bin/bash
# /usr/local/bin/collect_ipv6_traffic.sh

DEVICE="2001:db8::router1"
COMMUNITY="public"
RRD="/var/lib/rrd/ipv6_eth0.rrd"
IF_INDEX=2  # Interface index for eth0

# Get 64-bit counters (ifHCInOctets, ifHCOutOctets)
IN_BYTES=$(snmpget -v2c -c $COMMUNITY \
  udp6:[${DEVICE}]:161 \
  .1.3.6.1.2.1.31.1.1.1.6.${IF_INDEX} 2>/dev/null | \
  awk '{print $NF}')

OUT_BYTES=$(snmpget -v2c -c $COMMUNITY \
  udp6:[${DEVICE}]:161 \
  .1.3.6.1.2.1.31.1.1.1.10.${IF_INDEX} 2>/dev/null | \
  awk '{print $NF}')

IN_PKTS=$(snmpget -v2c -c $COMMUNITY \
  udp6:[${DEVICE}]:161 \
  .1.3.6.1.2.1.2.2.1.11.${IF_INDEX} 2>/dev/null | \
  awk '{print $NF}')

OUT_PKTS=$(snmpget -v2c -c $COMMUNITY \
  udp6:[${DEVICE}]:161 \
  .1.3.6.1.2.1.2.2.1.17.${IF_INDEX} 2>/dev/null | \
  awk '{print $NF}')

# Update RRD
if [ -n "$IN_BYTES" ] && [ -n "$OUT_BYTES" ]; then
  rrdtool update $RRD \
    "N:${IN_BYTES}:${OUT_BYTES}:${IN_PKTS}:${OUT_PKTS}"
  echo "$(date): Updated RRD: in=${IN_BYTES} out=${OUT_BYTES}"
else
  echo "$(date): ERROR - Could not reach ${DEVICE} via SNMP"
fi
```

```bash
# Make executable and schedule
chmod +x /usr/local/bin/collect_ipv6_traffic.sh

# Add to crontab (every 5 minutes)
echo "*/5 * * * * /usr/local/bin/collect_ipv6_traffic.sh >> /var/log/ipv6_collect.log 2>&1" \
  | sudo crontab -u root -
```

## Generating Graphs from IPv6 RRD Data

```bash
#!/bin/bash
# /usr/local/bin/graph_ipv6_traffic.sh

RRD="/var/lib/rrd/ipv6_eth0.rrd"
GRAPH_DIR="/var/www/html/rrd"
mkdir -p $GRAPH_DIR

# Daily graph
rrdtool graph "${GRAPH_DIR}/ipv6_daily.png" \
  --title "IPv6 Traffic - Daily" \
  --start "now-1d" \
  --end "now" \
  --vertical-label "bits/s" \
  --width 800 \
  --height 300 \
  DEF:in_bytes=${RRD}:in_bytes:AVERAGE \
  DEF:out_bytes=${RRD}:out_bytes:AVERAGE \
  CDEF:in_bits=in_bytes,8,* \
  CDEF:out_bits=out_bytes,8,* \
  AREA:in_bits#00CF00:"In  " \
  LINE1:out_bits#0000CF:"Out" \
  GPRINT:in_bits:LAST:" Last\: %6.2lf %sbps\n" \
  GPRINT:out_bits:LAST:" Last\: %6.2lf %sbps\n"

echo "Graph saved to ${GRAPH_DIR}/ipv6_daily.png"
```

## Python-Based IPv6 RRD Collection

```python
#!/usr/bin/env python3
# collect_ipv6_rrd.py

import subprocess
import rrdtool
from datetime import datetime

def snmp_get_ipv6(host, community, oid):
    """Get SNMP value from IPv6 device."""
    cmd = ['snmpget', '-v2c', '-c', community,
           f'udp6:[{host}]:161', oid]
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode == 0:
        return result.stdout.split()[-1]
    return None

# Collect data
device = '2001:db8::router1'
in_bytes = snmp_get_ipv6(device, 'public', '.1.3.6.1.2.1.31.1.1.1.6.2')
out_bytes = snmp_get_ipv6(device, 'public', '.1.3.6.1.2.1.31.1.1.1.10.2')

if in_bytes and out_bytes:
    rrdtool.update('/var/lib/rrd/ipv6_eth0.rrd',
                   f'N:{in_bytes}:{out_bytes}:0:0')
    print(f"Updated at {datetime.now()}")
```

RRDtool with IPv6 SNMP collection requires only specifying the device address in `udp6:[address]:161` format for snmpget commands, while the RRD database structure and graphing commands remain identical to IPv4 deployments.
