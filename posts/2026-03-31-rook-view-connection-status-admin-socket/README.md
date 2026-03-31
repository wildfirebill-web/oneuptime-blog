# How to View Connection Status via Admin Socket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Admin Socket, Connection, Network, Debug

Description: View active network connections, messenger state, and peer connectivity for Ceph daemons via the admin socket to diagnose network issues and monitor cluster communication.

---

## Overview

Ceph daemons communicate over TCP/IP using an internal messenger protocol. The admin socket exposes commands to inspect active connections, view peer states, and diagnose connectivity issues without using external network tools.

## Viewing Connections on an OSD

```bash
# Show active connections and messenger state
ceph daemon osd.0 connections

# View messenger statistics
ceph daemon osd.0 perf dump | python3 -m json.tool | grep -A5 '"ms"'
```

## Dumping Messenger Statistics

```bash
# Detailed messenger performance stats
ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
ms = data.get('AsyncMessenger::Worker-0', {})
if ms:
    print('send_bytes:', ms.get('send_bytes', {}).get('val', 0))
    print('recv_bytes:', ms.get('recv_bytes', {}).get('val', 0))
    print('send_messages:', ms.get('send_messages', {}).get('val', 0))
    print('recv_messages:', ms.get('recv_messages', {}).get('val', 0))
"
```

## Viewing Session State on MON

```bash
# View MON connection sessions
ceph daemon mon.$(hostname) sessions

# Show active subscriptions
ceph daemon mon.$(hostname) dump_watchers
```

## Checking OSD Peer Connections

```bash
# List connected peers for an OSD
ceph daemon osd.0 dump_watchers

# View operation tracking for connected clients
ceph daemon osd.0 dump_ops_in_flight | python3 -m json.tool
```

## Network Health via Admin Socket

```bash
# Check if OSD can see its peers
ceph daemon osd.0 config get cluster_addr
ceph daemon osd.0 config get public_addr

# Verify OSD network interfaces
ceph daemon osd.0 config get cluster_network
ceph daemon osd.0 config get public_network
```

## Connection Debugging Script

```bash
#!/bin/bash
# check-osd-connections.sh - verify all OSD connections are healthy

ERRORS=0
for osd in $(ceph osd ls); do
    STATUS=$(ceph daemon osd.$osd version 2>&1)
    if echo "$STATUS" | grep -q "version"; then
        echo "OSD $osd: CONNECTED ($(echo $STATUS | awk '{print $3}'))"
    else
        echo "OSD $osd: ERROR - $STATUS"
        ((ERRORS++))
    fi
done

echo ""
echo "Total OSDs: $(ceph osd ls | wc -l)"
echo "Connection errors: $ERRORS"
```

## Messenger Connection Counters

```bash
# Check for connection errors and resets
for i in 0 1 2; do
    echo "--- AsyncMessenger Worker $i ---"
    ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
ms = data.get(f'AsyncMessenger::Worker-$i', {})
for k in ['connection_ready', 'connection_rejected', 'send_messages', 'recv_messages']:
    v = ms.get(k, {})
    print(f'  {k}: {v.get(\"val\", 0)}')
" 2>/dev/null
done
```

## Detecting Connection Flapping

```bash
# Watch for connection reset messages in OSD log
journalctl -u ceph-osd@0 --no-pager | grep -E "reset|disconnect|lost connection" | tail -20

# Check messenger error counters
ceph daemon osd.0 perf dump | python3 -m json.tool | grep -i "error\|reset\|lost"
```

## Summary

The Ceph admin socket provides visibility into daemon network connections through the `connections`, `sessions`, and messenger perf counters. Use these to diagnose network partitions, identify flapping connections, and verify that all cluster members are communicating correctly. Combined with external network tools, admin socket connection inspection gives a complete picture of cluster network health.
