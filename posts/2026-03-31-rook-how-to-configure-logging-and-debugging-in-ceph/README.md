# How to Configure Logging and Debugging in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Logging, Debugging, Troubleshooting

Description: Configure Ceph log levels, log file locations, and debug subsystems to capture detailed diagnostic information for troubleshooting cluster issues.

---

## Ceph Logging Architecture

Ceph uses a subsystem-based logging model. Each component (OSD, monitor, RGW, MDS, etc.) has independent log levels that can be tuned at runtime without restarting daemons.

Log messages have levels 0-20:
- 0: Fatal/critical errors
- 1: Important warnings and errors
- 5: Moderate detail
- 10: High verbosity
- 20: Maximum verbosity (very performance-impacting)

## Viewing Current Log Levels

```bash
# View log levels for all daemons
ceph config dump | grep log

# View for a specific daemon type
ceph config get osd log_level

# View runtime log level via admin socket
ceph daemon osd.0 log get-level
```

## Setting Global Log Levels

```bash
# Set global log level (applies to all Ceph daemons)
ceph config set global log_to_file true
ceph config set global log_level 1

# Set file log path
ceph config set global log_file /var/log/ceph/ceph.log
```

## Setting Per-Component Log Levels

```bash
# Increase OSD log level for debugging
ceph config set osd log_level 5

# Set monitor logging
ceph config set mon log_level 2

# Increase RGW logging
ceph config set client.rgw log_level 5

# Temporarily debug a single OSD
ceph config set osd.3 log_level 10
```

## Runtime Log Level Changes (No Restart Required)

Ceph allows changing log levels at runtime via the admin socket:

```bash
# Increase log level on a running OSD
ceph daemon osd.0 log set-level 10

# Or via the config set command (applied live)
ceph tell osd.0 injectargs --debug-osd 10

# Restore to normal
ceph daemon osd.0 log set-level 1
```

## Subsystem-Specific Debugging

Ceph has fine-grained subsystem debug flags for deep troubleshooting:

```bash
# Enable CRUSH debugging
ceph tell osd.0 injectargs --debug-crush 10

# Enable objecter debugging
ceph tell osd.0 injectargs --debug-objecter 10

# Enable RGW debugging
ceph tell client.rgw.myrgw injectargs --debug-rgw 10

# Enable OSD PG debugging
ceph tell osd.0 injectargs --debug-osd 10 --debug-pg 10
```

Available debug subsystems include:

```text
debug-osd       - OSD operations
debug-pg        - Placement group operations
debug-crush     - CRUSH mapping
debug-mon       - Monitor operations
debug-ms        - Messaging layer
debug-mds       - CephFS MDS
debug-rgw       - RGW object gateway
debug-client    - Client operations
debug-bluestore - BlueStore storage backend
debug-journaler - Journal operations
debug-paxos     - Monitor consensus
```

## Log File Configuration

```bash
# Set log file path for all daemons
ceph config set global log_file /var/log/ceph/ceph-$type.$id.log

# Enable log rotation (default: true)
ceph config set global log_max_new 0  # Disable rotation
ceph config set global log_max_recent 1000  # Keep last 1000 lines in memory

# Maximum log file size before rotation (in MiB)
ceph config set global log_max_new 100
```

## Ceph Log Output Destinations

Logs can be sent to multiple destinations simultaneously:

```bash
# Log to file
ceph config set global log_to_file true

# Log to syslog
ceph config set global log_to_syslog false

# Log to stderr
ceph config set global log_to_stderr false

# Log to journald (systemd)
ceph config set global log_to_journald true
```

## Viewing Cluster Logs

```bash
# View recent cluster log entries
ceph log last 50

# Filter by channel
ceph log last 50 cluster
ceph log last 50 audit
ceph log last 50 rgw

# Watch logs in real time
ceph -w

# Watch specific channel
ceph -w --watch-channel=cluster
```

## Debug Logging for Specific Operations

For a targeted debugging session:

```bash
# Start debugging before reproducing the issue
ceph tell osd.* injectargs --debug-osd 10

# Reproduce the issue
# ... (perform the operation that is failing)

# Collect logs
journalctl -u "ceph-osd@*" --since "5 minutes ago" > /tmp/osd-debug.log

# Restore normal log level
ceph tell osd.* injectargs --debug-osd 1
```

## Ceph Logging in Rook/Kubernetes

In Rook, logs are available via kubectl:

```bash
# View OSD logs
kubectl -n rook-ceph logs -l app=rook-ceph-osd --tail=100

# Follow logs for a specific OSD pod
kubectl -n rook-ceph logs -f rook-ceph-osd-0-<pod-id>

# View monitor logs
kubectl -n rook-ceph logs -l app=rook-ceph-mon --tail=100

# Increase log level via Rook toolbox
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd log_level 5
```

## Summary

Ceph logging is configured per-subsystem and can be changed at runtime without daemon restarts. Use `ceph config set <daemon-type> log_level <0-20>` for global adjustments, or `ceph tell <daemon> injectargs --debug-<subsystem> <level>` for targeted debugging. For deep issue investigation, enable specific subsystem debug flags (e.g., `--debug-osd 10 --debug-pg 10`) while reproducing the problem, then collect the logs and restore normal log levels. In Rook/Kubernetes, access logs via `kubectl logs` and adjust levels via the toolbox pod.
