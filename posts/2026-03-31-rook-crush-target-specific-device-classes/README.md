# How to Target Specific Device Classes in CRUSH Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Device Class, Storage

Description: Learn how to write Ceph CRUSH rules that target specific device classes (HDD, SSD, NVMe) to implement tiered storage pools within a mixed-media cluster.

---

## Why Target Device Classes in CRUSH Rules

In a mixed-media Ceph cluster containing HDDs, SSDs, and NVMe drives, you need CRUSH rules that direct specific pools to specific drive types. Without device class targeting, CRUSH distributes data across all available OSDs regardless of drive type, making tiered storage impossible.

Device class targeting uses Ceph's shadow CRUSH hierarchies - virtual subtrees containing only the OSDs of a given class - to restrict placement to matching drives.

## How Shadow Hierarchies Work

When you assign a device class to OSDs, Ceph automatically creates a shadow CRUSH hierarchy for that class. For example, with `hdd` OSDs in `rack-a` and `ssd` OSDs in `rack-b`, Ceph creates:

```text
default (root)
  rack-a
    node-01
      osd.0 (hdd)
      osd.1 (hdd)
  rack-b
    node-02
      osd.2 (ssd)
      osd.3 (ssd)

~hdd (shadow root - contains only HDD OSDs)
  rack-a~hdd
    node-01~hdd
      osd.0
      osd.1

~ssd (shadow root - contains only SSD OSDs)
  rack-b~ssd
    node-02~ssd
      osd.2
      osd.3
```

## Creating Device-Class-Targeted Rules

```bash
# Create rules targeting each device class
ceph osd crush rule create-replicated hdd-rule default host hdd
ceph osd crush rule create-replicated ssd-rule default host ssd
ceph osd crush rule create-replicated nvme-rule default host nvme

# List rules to verify
ceph osd crush rule ls
ceph osd crush rule dump ssd-rule
```

The `create-replicated` shorthand accepts:
- rule name
- root bucket
- failure domain type
- device class (optional)

## Applying Rules to Pools

```bash
# Create a pool on SSD storage
ceph osd pool create hot-data 128 128 replicated ssd-rule

# Create a pool on HDD storage
ceph osd pool create cold-archive 64 64 replicated hdd-rule

# Move an existing pool to a different device class
ceph osd pool set hot-data crush_rule ssd-rule

# Verify the rule on a pool
ceph osd pool get hot-data crush_rule
```

## Creating Class-Targeted Rules Manually

For erasure coded pools or multi-level rules, add class targeting via the CRUSH map:

```bash
ceph osd getcrushmap -o crush.bin
crushtool -d crush.bin -o crush.txt
```

Edit `crush.txt` to add a class-filtered rule:

```text
rule nvme-rack-rule {
    id 7
    type replicated
    step take default class nvme
    step chooseleaf firstn 0 type rack
    step emit
}
```

```bash
crushtool -c crush.txt -o crush-new.bin
ceph osd setcrushmap -i crush-new.bin
```

## Verifying Data Lands on the Correct Drives

```bash
# Map an object to see which OSDs handle it
ceph osd map ssd-pool testobject

# Check the class of each OSD in the result
for osd in 2 3 5; do
  ceph osd tree | grep "osd.$osd"
done

# Confirm object placement uses only SSD OSDs
ceph osd dump | grep "ssd-pool" | grep crush_rule
```

## Rook Configuration for Device Classes

In Rook, set `deviceClass` directly in the pool spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: nvme-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  deviceClass: nvme
  replicated:
    size: 3
```

## Summary

CRUSH device class targeting enables true tiered storage within a single Ceph cluster. Assign classes to OSDs, then create rules using `ceph osd crush rule create-replicated` with the device class argument, or use the `step take default class <classname>` directive in manual CRUSH map edits. In Rook, the `deviceClass` field in pool specs handles this automatically, creating the appropriate shadow hierarchy rule behind the scenes.
