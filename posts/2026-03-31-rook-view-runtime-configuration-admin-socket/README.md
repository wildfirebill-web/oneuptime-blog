# How to View Runtime Configuration via Admin Socket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Admin Socket, Configuration, Runtime, Debug

Description: View the active runtime configuration of any Ceph daemon via the admin socket to verify settings, compare with defaults, and diagnose configuration drift.

---

## Overview

Ceph daemons can have their configuration changed at runtime without a restart. The admin socket provides commands to view the current active configuration, see which values differ from defaults, and inspect individual settings. This is essential for verifying that configuration changes were applied correctly.

## Viewing the Full Configuration

```bash
# Show all configuration for an OSD daemon
ceph daemon osd.0 config show

# Show full config for RGW
ceph daemon rgw.myzone config show

# Show full config for MON
ceph daemon mon.$(hostname) config show
```

The output is a JSON object with all configuration key-value pairs currently in effect.

## Getting a Specific Configuration Value

```bash
# Get a specific config key from a running daemon
ceph daemon osd.0 config get osd_max_backfills

# Get RGW-specific settings
ceph daemon rgw.myzone config get rgw_thread_pool_size
ceph daemon rgw.myzone config get d3n_l1_local_datacache_enabled
ceph daemon rgw.myzone config get d3n_l1_datacache_size

# Get network settings
ceph daemon mon.$(hostname) config get public_network
```

## Viewing Configuration Differences from Default

```bash
# Show only values that differ from compiled-in defaults
ceph daemon osd.0 config diff

# This is especially useful after applying changes to verify what was modified
ceph daemon rgw.myzone config diff | python3 -m json.tool
```

Example output:

```json
{
  "diff": {
    "changed": {
      "d3n_l1_local_datacache_enabled": "true",
      "d3n_l1_datacache_size": "10737418240",
      "rgw_thread_pool_size": "512"
    },
    "defaults": {
      "d3n_l1_local_datacache_enabled": "false",
      "d3n_l1_datacache_size": "0",
      "rgw_thread_pool_size": "100"
    }
  }
}
```

## Comparing Running Config with ceph.conf

```bash
# Export running config to a file
ceph daemon osd.0 config show > /tmp/osd0-running-config.json

# Compare with what is in the Ceph config store
ceph config show osd.0 > /tmp/osd0-stored-config.txt

# If values differ, the daemon may have been modified at runtime without persisting
```

## Checking All OSD Configs

```bash
# Loop through all OSD admin sockets
for i in $(ceph osd ls); do
    echo "=== OSD $i ==="
    ceph daemon osd.$i config get osd_max_backfills 2>/dev/null
done
```

## Viewing Ceph Log Levels at Runtime

```bash
# Check current debug log levels for an OSD
ceph daemon osd.0 config get debug_osd
ceph daemon osd.0 config get debug_ms

# Check RGW log levels
ceph daemon rgw.myzone config get debug_rgw
ceph daemon rgw.myzone config get log_to_stderr
```

## Practical Verification After Config Changes

```bash
# After applying a config change via ceph config set, verify it took effect
ceph config set osd osd_recovery_max_active 5
ceph daemon osd.0 config get osd_recovery_max_active
# Should return: 5
```

## Summary

The admin socket `config show`, `config get`, and `config diff` commands provide complete visibility into a running daemon's configuration. Use `config diff` to quickly see what has been customized from defaults. Verify configuration changes took effect immediately after applying them, and compare running config with the stored config to detect runtime drift that has not been persisted.
