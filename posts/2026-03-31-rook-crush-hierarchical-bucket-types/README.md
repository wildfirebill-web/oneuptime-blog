# How to Create Hierarchical Bucket Types in CRUSH Maps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Storage, Configuration

Description: Learn how to define and use hierarchical bucket types in Ceph CRUSH maps to model physical topology and enforce multi-level failure domain rules.

---

## What are CRUSH Bucket Types

In Ceph's CRUSH algorithm, a "bucket" is any node in the topology hierarchy that contains other buckets or OSD devices. Bucket types define the levels of that hierarchy - for example, host, rack, row, datacenter, and root. Each type is assigned a numeric ID, and the hierarchy must progress from lower IDs (closer to OSDs) to higher IDs (closer to the root).

The default Ceph bucket type hierarchy is:

```text
type 0  osd
type 1  host
type 2  chassis
type 3  rack
type 4  row
type 5  pdu
type 6  pod
type 7  room
type 8  datacenter
type 9  zone
type 10 region
type 11 root
```

## Adding Custom Bucket Types

You can add custom hierarchy levels by modifying the CRUSH map:

```bash
# Export and decompile the current CRUSH map
ceph osd getcrushmap -o crush.bin
crushtool -d crush.bin -o crush.txt
```

Edit `crush.txt` to add custom types:

```text
# types - add custom levels
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 zone
type 10 region
type 11 root
type 12 floor          # custom: physical floor in a building
type 13 building       # custom: building in a campus
```

```bash
# Recompile and apply
crushtool -c crush.txt -o crush-new.bin
ceph osd setcrushmap -i crush-new.bin
```

## Creating Bucket Instances

Once types are defined, create bucket instances of each type:

```bash
# Create host-level buckets
ceph osd crush add-bucket node-01 host
ceph osd crush add-bucket node-02 host

# Create rack-level buckets
ceph osd crush add-bucket rack-a rack
ceph osd crush add-bucket rack-b rack

# Create datacenter-level buckets
ceph osd crush add-bucket dc-east datacenter
ceph osd crush add-bucket dc-west datacenter

# Create a root
ceph osd crush add-bucket default root
```

## Building the Hierarchy

Connect buckets to their parents:

```bash
# Place hosts in racks
ceph osd crush move node-01 rack=rack-a
ceph osd crush move node-02 rack=rack-b

# Place racks in datacenters
ceph osd crush move rack-a datacenter=dc-east
ceph osd crush move rack-b datacenter=dc-west

# Place datacenters under root
ceph osd crush move dc-east root=default
ceph osd crush move dc-west root=default

# Verify the resulting hierarchy
ceph osd tree
```

Expected tree output:

```text
ID  CLASS WEIGHT  TYPE NAME         STATUS REWEIGHT PRI-AFF
-1        10.000  root default
-5         5.000      datacenter dc-east
-3         5.000          rack rack-a
-2         5.000              host node-01
 0  hdd    1.000                  osd.0   up  1.00000  1.00000
```

## Writing Rules That Use Multiple Levels

Define a CRUSH rule that spans the datacenter level for maximum failure isolation:

```text
rule replicated_dc {
    id 2
    type replicated
    step take default
    step chooseleaf firstn 0 type datacenter
    step emit
}
```

```bash
# Apply the rule to a pool
ceph osd pool set critical-pool crush_rule replicated_dc
```

This ensures each replica lands in a different datacenter.

## Summary

CRUSH bucket types define the levels of your storage topology hierarchy. Default types cover most deployments, but you can add custom types for unique environments. Create bucket instances with `ceph osd crush add-bucket`, build the hierarchy with `ceph osd crush move`, and write CRUSH rules that use specific bucket type levels to achieve the desired failure domain isolation.
