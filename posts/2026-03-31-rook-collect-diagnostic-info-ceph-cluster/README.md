# How to Collect Diagnostic Information from a Ceph Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Diagnostic, Troubleshooting, Debug

Description: Collect comprehensive diagnostic data from a Ceph cluster including logs, crash dumps, perf counters, and cluster state for effective support escalation.

---

When troubleshooting Ceph issues or escalating to the community or a vendor, having a complete diagnostic bundle is essential. This guide shows how to gather everything needed in one pass.

## Cluster State Snapshot

Start with a full state capture:

```bash
#!/bin/bash
OUTDIR="/tmp/ceph-diag-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTDIR"

ceph status          > "$OUTDIR/status.txt"
ceph health detail   > "$OUTDIR/health_detail.txt"
ceph osd dump        > "$OUTDIR/osd_dump.txt"
ceph osd tree        > "$OUTDIR/osd_tree.txt"
ceph mon dump        > "$OUTDIR/mon_dump.txt"
ceph pg dump         > "$OUTDIR/pg_dump.txt"
ceph df detail       > "$OUTDIR/df_detail.txt"
ceph osd perf        > "$OUTDIR/osd_perf.txt"
ceph log last 500    > "$OUTDIR/cluster_log.txt"
```

## OSD Daemon Diagnostics

For each OSD, collect runtime state:

```bash
for OSD in $(ceph osd ls); do
  ceph daemon "osd.$OSD" status       > "$OUTDIR/osd_${OSD}_status.txt" 2>&1
  ceph daemon "osd.$OSD" perf dump    > "$OUTDIR/osd_${OSD}_perf.txt" 2>&1
  ceph daemon "osd.$OSD" config show  > "$OUTDIR/osd_${OSD}_config.txt" 2>&1
done
```

## Crash Dumps

Ceph stores crash data in the cluster:

```bash
ceph crash ls          > "$OUTDIR/crash_list.txt"
ceph crash stat        > "$OUTDIR/crash_stat.txt"

# Archive each crash
for CRASH in $(ceph crash ls --format json | python3 -c "import sys,json; [print(c['crash_id']) for c in json.load(sys.stdin)]"); do
  ceph crash info "$CRASH" > "$OUTDIR/crash_${CRASH}.txt"
done
```

## System and Host Information

Collect OS-level details from each Ceph host:

```bash
uname -a              > "$OUTDIR/uname.txt"
df -h                 > "$OUTDIR/df.txt"
free -h               > "$OUTDIR/memory.txt"
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT > "$OUTDIR/lsblk.txt"
cat /proc/cpuinfo     > "$OUTDIR/cpuinfo.txt"
ip addr               > "$OUTDIR/ip_addr.txt"
```

## Log Collection

Collect recent systemd logs for Ceph services:

```bash
journalctl -u "ceph-mon@*" --since "24 hours ago" > "$OUTDIR/mon_journal.txt"
journalctl -u "ceph-osd@*" --since "24 hours ago" > "$OUTDIR/osd_journal.txt"
journalctl -u "ceph-mgr@*" --since "24 hours ago" > "$OUTDIR/mgr_journal.txt"
```

## Package and Version Info

```bash
ceph version                       > "$OUTDIR/ceph_version.txt"
rpm -qa | grep ceph                > "$OUTDIR/ceph_packages.txt" 2>/dev/null
dpkg -l | grep ceph                >> "$OUTDIR/ceph_packages.txt" 2>/dev/null
```

## Archive the Bundle

```bash
tar -czf "/tmp/ceph-diag-$(date +%Y%m%d).tar.gz" -C /tmp "$(basename $OUTDIR)"
echo "Diagnostic bundle created at /tmp/ceph-diag-$(date +%Y%m%d).tar.gz"
```

## Summary

A thorough Ceph diagnostic bundle includes cluster state, per-OSD daemon data, crash dumps, OS-level information, and recent journal logs. Collecting all of this upfront saves significant back-and-forth when working with the Ceph community or a support team. Automate this collection script so it can be triggered quickly during an incident.
