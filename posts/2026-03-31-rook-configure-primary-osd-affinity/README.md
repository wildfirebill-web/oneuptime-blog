# How to Configure Primary OSD Selection and Primary Affinity

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Performance, Storage

Description: Configure Ceph primary OSD selection and primary affinity values to control which OSDs handle client reads and writes for performance optimization.

---

## What is Primary OSD Selection

In a Ceph replicated pool, one OSD in each placement group is designated the "primary." The primary OSD handles all client reads and writes, then replicates to the replica OSDs. By default, CRUSH selects the primary OSD using the same algorithm as replicas - typically the first OSD in the PG's acting set.

**Primary affinity** is a per-OSD value (0.0 to 1.0) that biases whether that OSD is selected as a primary. A value of 1.0 (default) means full eligibility; 0.0 means the OSD is never selected as a primary.

## Why Adjust Primary Affinity

Common use cases include:

- **Mixed-speed deployments** - set higher affinity for faster drives (SSD/NVMe) to serve reads, lower for HDDs
- **Hardware differences** - reduce primary load on older or slower nodes
- **Network asymmetry** - direct primaries toward nodes with better client-facing network connectivity
- **Rolling maintenance** - temporarily reduce primary affinity before taking a node down

## Checking Current Primary Affinity Values

```bash
# Show all OSD primary affinity values
ceph osd tree | head -5  # affinity shown in PRI-AFF column

# Show detailed OSD info including primary affinity
ceph osd dump | grep -E "^osd\.[0-9]+" | awk '{print $1, $NF}'

# Check specific OSD
ceph osd dump | grep "osd.0" | grep primary_affinity
```

## Setting Primary Affinity

```bash
# Set full primary affinity (default - OSD is eligible to be primary)
ceph osd primary-affinity osd.0 1.0

# Reduce primary affinity (OSD will be primary less often)
ceph osd primary-affinity osd.1 0.5

# Disable primary selection for an OSD (never primary)
ceph osd primary-affinity osd.2 0.0

# Verify the change
ceph osd tree | grep osd.2
```

The `PRI-AFF` column in `ceph osd tree` shows the current value.

## Biasing Primaries Toward SSD OSDs

In a cluster with mixed HDD and SSD OSDs:

```bash
# Get SSD OSD IDs
ssd_osds=$(ceph osd crush class ls-osd ssd)

# Set high primary affinity for SSDs
for osd in $ssd_osds; do
  ceph osd primary-affinity osd.$osd 1.0
done

# Get HDD OSD IDs
hdd_osds=$(ceph osd crush class ls-osd hdd)

# Reduce primary affinity for HDDs
for osd in $hdd_osds; do
  ceph osd primary-affinity osd.$osd 0.3
done

# Verify the distribution of primaries
ceph pg dump | grep primary | awk '{print $14}' | sort | uniq -c | sort -nr | head -20
```

## Read Affinity (Balancing Reads Across Replicas)

In addition to primary affinity, Ceph supports read affinity to direct reads to replica OSDs:

```bash
# Enable read balancing (distributes reads to any replica, not just primary)
ceph osd pool set mypool read_balance_score 1

# Check read balance configuration
ceph osd pool get mypool read_balance_score
```

## Monitoring Primary Distribution

After adjusting primary affinity, verify the primaries are distributed as expected:

```bash
# Count how many PGs each OSD is primary for
ceph pg dump pgs | awk '{print $14}' | tr ',' '\n' | \
  awk 'NR==1 || (NR%3==1)' | sort | uniq -c | sort -nr | head -20

# Check utilization to see if read load has shifted
ceph osd stats | sort -k 3 -nr | head -10
```

## Summary

Primary affinity values (0.0 to 1.0) control the likelihood of an OSD being selected as the primary for placement groups. Set higher values on faster or better-connected OSDs and lower values on HDDs or nodes undergoing maintenance. Use `ceph osd primary-affinity` to adjust individual OSDs and verify results with `ceph osd tree` to see the PRI-AFF column. This is a lightweight way to bias read/write load without changing the CRUSH map.
