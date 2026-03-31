# How to Check Ceph Client Connection Count

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Monitoring, Client, Connection, Network

Description: Monitor the number of active client connections to Ceph OSDs, monitors, and RGW to detect connection saturation and diagnose client-side issues.

---

## Why Monitor Client Connections?

Too many simultaneous client connections can exhaust file descriptor limits on Ceph daemons, leading to refused connections or degraded performance. Tracking connection counts helps you detect connection leaks, identify noisy tenants, and plan capacity for large client deployments.

## Checking Connections to Monitors

See how many clients are connected to each monitor:

```bash
ceph tell mon.* sessions
```

Or for a specific monitor:

```bash
ceph daemon mon.mon1 sessions
```

## Checking Connections to OSDs

Count active connections per OSD:

```bash
ceph tell osd.* perf dump | grep -A2 "\"sessions_add\""
```

For a specific OSD via the admin socket:

```bash
ceph daemon osd.0 perf dump | python3 -c "
import json, sys
data = json.load(sys.stdin)
sessions = data.get('osd', {}).get('numpg', 0)
print('PGs:', sessions)
ms = data.get('ms', {})
print('Sessions dispatched:', ms.get('sessions_add', {}).get('val', 0))
"
```

## Checking MGR Client Sessions

```bash
ceph daemon mgr.$(ceph mgr stat | python3 -c "import json,sys; print(json.load(sys.stdin)['active_name'])") sessions
```

## OS-Level Connection Check

On individual OSD or MON hosts, use `ss` to count established connections:

```bash
# Count connections to OSD port (6800-7300 range)
ss -tn state established '( dport >= 6800 and dport <= 7300 )' | wc -l

# Show which clients are connected
ss -tnp state established '( dport >= 6800 and dport <= 7300 )'
```

## Checking RGW Client Connections

For RGW (Beast frontend), check active HTTP connections:

```bash
ceph daemon client.rgw.myzone perf dump | python3 -c "
import json, sys
data = json.load(sys.stdin)
rgw = data.get('rgw', {})
print('Active requests:', rgw.get('req', {}).get('val', 0))
"
```

Or use `ss` against the RGW port:

```bash
ss -tn state established 'dport = :7480' | wc -l
```

## File Descriptor Limits

If connection counts are high, verify the file descriptor limits on Ceph daemons:

```bash
# Get the PID of osd.0
OSD_PID=$(systemctl show ceph-osd@0.service -p MainPID | cut -d= -f2)
cat /proc/$OSD_PID/limits | grep "open files"
```

If near the limit, increase it in `/etc/systemd/system/ceph-osd@.service.d/override.conf`:

```ini
[Service]
LimitNOFILE=1048576
```

## Summary

Monitor Ceph client connection counts using `ceph tell mon.* sessions` for monitor connections, `ss` for OS-level socket counts per daemon, and RGW-specific perf counters for HTTP session tracking. Keep an eye on file descriptor limits when client counts are high to prevent connection refusals.
