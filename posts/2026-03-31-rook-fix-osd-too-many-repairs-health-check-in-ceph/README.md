# How to Fix OSD_TOO_MANY_REPAIRS Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, OSD, Health Check, Data Integrity

Description: Learn how to resolve OSD_TOO_MANY_REPAIRS in Ceph, a warning triggered when an OSD has performed an unusually high number of read repairs during scrubbing.

---

## What Is OSD_TOO_MANY_REPAIRS?

`OSD_TOO_MANY_REPAIRS` is a Ceph health warning that fires when an OSD has repaired more objects during scrubbing than a configured threshold (`osd_scrub_max_preemptions`). A "repair" during scrubbing means the OSD detected that a stored copy was different from the authoritative copy and overwrote it.

While a small number of repairs is normal (e.g., from race conditions during writes), a high repair count suggests the OSD disk may have silent data corruption - bit rot - where data changes on disk without triggering an error at the filesystem level.

## Checking the Alert

```bash
ceph health detail
```

Example:

```text
[WRN] OSD_TOO_MANY_REPAIRS: Too many repairs on 1 OSD(s)
    osd.7 had 23 read repairs
```

Check OSD repair statistics:

```bash
ceph tell osd.7 dump_repair_stats
```

Check the scrub history for the OSD's PGs:

```bash
ceph pg dump | grep -E "^[0-9]" | awk '$18 > 0 {print $1, $18, "repairs"}'
```

## Understanding the Risk

High repair rates mean data is silently changing on disk. This is a symptom of:

- Failing disk hardware with undetected sector errors
- Filesystem-level corruption (ext4/xfs journal issues)
- Memory corruption causing bad writes
- Firmware bugs in the storage controller

## Remediation Steps

### Step 1 - Run a Full Deep Scrub

Force a complete deep scrub on all PGs hosted by the OSD to get a baseline:

```bash
ceph osd deep-scrub osd.7
```

Watch for repair counts:

```bash
ceph pg dump | grep repair
```

### Step 2 - Check Disk SMART Data

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "smartctl -x /dev/sdd | grep -E 'Reallocated|Uncorrectable|Pending'"
```

High values in `Reallocated_Sector_Ct` or `Offline_Uncorrectable` confirm disk degradation.

### Step 3 - Check Memory ECC

```bash
dmidecode --type 17 | grep -E "Type|Size|ECC"
```

### Step 4 - Replace the OSD

If hardware issues are confirmed, proactively replace the OSD before it fails:

```bash
ceph osd out osd.7
```

Wait for rebalance to complete:

```bash
watch ceph -s
```

Then remove:

```bash
ceph osd down osd.7
ceph osd rm osd.7
ceph osd crush remove osd.7
ceph auth del osd.7
```

In Rook, trigger reprovisioning:

```bash
kubectl -n rook-ceph rollout restart deploy/rook-ceph-operator
```

## Adjusting the Repair Warning Threshold

If you want to adjust when this alert fires:

```bash
ceph config set osd osd_scrub_auto_repair_num_errors 5
```

## Summary

`OSD_TOO_MANY_REPAIRS` warns that an OSD is repairing data too frequently during scrubs, indicating silent disk corruption (bit rot). Investigate with SMART data and memory checks, and replace the OSD if hardware degradation is confirmed. This alert exists specifically to catch failing disks before they cause unrecoverable data loss.
