# How to Configure the Crash Module in Ceph Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Crash Reporting, Debugging, Monitoring

Description: Learn how to configure the Ceph Manager crash module to collect, archive, and manage daemon crash reports for debugging cluster failures.

---

The Ceph Manager crash module collects crash reports from all Ceph daemons (OSDs, monitors, MDSes, etc.) and stores them in the cluster for review and analysis. This is essential for post-mortem debugging after unexpected daemon crashes.

## Enabling the Crash Module

Enable the crash module:

```bash
ceph mgr module enable crash
```

By default, crash data is retained for a configurable period before being pruned.

## Viewing Crash Reports

List all stored crash reports:

```bash
ceph crash ls
```

Example output:

```
ID                                           ENTITY   NEW
2026-03-31_09:22:11.abc123_osd.5            osd.5    *
2026-03-31_08:10:05.def456_mon.a            mon.a
```

The asterisk (`*`) marks crashes that have not been acknowledged.

## Examining a Crash Report

View the full backtrace and details for a specific crash:

```bash
ceph crash info 2026-03-31_09:22:11.abc123_osd.5
```

This returns a JSON object with the crash timestamp, version, backtrace, and entity name.

## Archiving Crashes

After reviewing a crash, mark it as acknowledged to prevent it from triggering health warnings:

```bash
ceph crash archive 2026-03-31_09:22:11.abc123_osd.5
```

Archive all unacknowledged crashes at once:

```bash
ceph crash archive-all
```

## Viewing Crash Stats

Get a summary of crash frequency by module:

```bash
ceph crash stat
```

Example output:

```
8 crashes recorded
1 crashes are new
```

## Configuring Retention

Set how many days crash reports are retained before automatic deletion:

```bash
ceph config set mgr mgr/crash/warn_recent_interval 86400
ceph config set mgr mgr/crash/retain_interval 31536000
```

- `warn_recent_interval` - interval in seconds to include crashes in the health warning
- `retain_interval` - total retention period before pruning

## Pruning Old Crashes

Manually remove old crash reports to free space:

```bash
ceph crash prune 90
```

This removes crash reports older than 90 days.

## Integration with Telemetry

The crash module feeds into the telemetry module's crash channel. When telemetry is enabled, anonymized crash backtraces are shared with the Ceph project:

```bash
ceph config set mgr mgr/telemetry/channel_crash true
```

## Summary

The Ceph Manager crash module centralizes crash report collection from all cluster daemons, making post-incident analysis straightforward. Operators can list, inspect, archive, and prune crash reports using the `ceph crash` command family, and configure retention intervals to balance storage use against diagnostic history.
