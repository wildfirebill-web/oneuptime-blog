# How to Understand Ceph Stretch Mode Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Stretch Mode, Architecture, High Availability

Description: Understand Ceph stretch mode architecture including dual-site replication, arbiter monitors, and how it protects against site-level failures.

---

## What is Ceph Stretch Mode?

Ceph stretch mode is a configuration that enables a Ceph cluster to span two geographic data centers (sites) while maintaining data availability even if one entire site fails. It provides active-active replication across both sites with an arbiter monitor as a tiebreaker.

## Core Components

A stretch mode deployment requires three logical zones:

- **Site A** - Primary data center with OSDs and monitors
- **Site B** - Secondary data center with OSDs and monitors
- **Arbiter site** - A lightweight third location hosting only a monitor (no OSDs)

This three-zone design ensures quorum can always be achieved by two of the three sites.

## CRUSH Topology

Stretch mode relies on a specific CRUSH hierarchy. Each OSD must be placed in a CRUSH bucket that corresponds to its site:

```bash
ceph osd crush add-bucket site-a datacenter
ceph osd crush add-bucket site-b datacenter
ceph osd crush move site-a root=default
ceph osd crush move site-b root=default
```

OSDs in each site are then moved under their respective datacenter bucket:

```bash
ceph osd crush move osd.0 host=host-a1 datacenter=site-a
ceph osd crush move osd.1 host=host-a2 datacenter=site-a
ceph osd crush move osd.2 host=host-b1 datacenter=site-b
ceph osd crush move osd.3 host=host-b2 datacenter=site-b
```

## Replication Rules

Stretch mode uses a special CRUSH rule that places data copies evenly across sites:

```bash
ceph osd crush rule create-replicated stretch-rule default datacenter osd
```

With `min_size=2` and `size=4`, each pool stores two copies per site, so reads/writes can proceed from either site independently.

## Monitor Quorum in Stretch Mode

Monitors are distributed as follows:

| Site | Monitors |
|------|----------|
| Site A | mon-a1, mon-a2 |
| Site B | mon-b1, mon-b2 |
| Arbiter | mon-arbiter |

With five monitors total, quorum requires three. If site A fails, site B and the arbiter still form quorum. If the arbiter is unreachable, sites A and B together still have four monitors for quorum.

```bash
ceph mon dump
```

## Write and Read Behavior

In normal operation, writes succeed when both sites acknowledge. When one site is down, Ceph enters a degraded but available state and writes only go to the surviving site. Once the failed site comes back, Ceph replays the missing writes.

Check current stretch mode status:

```bash
ceph osd dump | grep stretch
```

## Network Requirements

Stretch mode requires reliable inter-site connectivity. High latency (above 10ms RTT) between sites increases write latency proportionally. Configure a dedicated inter-site network and use traffic engineering to minimize packet loss.

```bash
# Check inter-site latency
ping -c 100 site-b-monitor-ip | tail -1
```

## Summary

Ceph stretch mode provides site-level fault tolerance through a dual-datacenter layout with an arbiter monitor. The CRUSH topology enforces even data distribution across sites, and the five-monitor quorum design ensures availability even with a complete site outage. Understanding this architecture is essential before attempting to enable or troubleshoot stretch mode.
