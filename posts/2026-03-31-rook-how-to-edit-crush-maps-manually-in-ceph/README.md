# How to Edit CRUSH Maps Manually in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CRUSH, Storage, Distributed Storage

Description: Learn how to manually extract, decompile, edit, and recompile Ceph CRUSH maps to customize data placement rules and bucket hierarchies.

---

## What Is the CRUSH Map

The CRUSH (Controlled Replication Under Scalable Hashing) map is the data structure Ceph uses to determine where data should be stored. It defines a hierarchy of buckets (rows, racks, hosts, OSDs) and rules for distributing replicas across them.

Manual edits to the CRUSH map let you:
- Define custom failure domains
- Pin specific pools to specific hardware
- Balance uneven OSD weights
- Configure stretch cluster topology

## Extracting the CRUSH Map

The CRUSH map is stored in compiled binary format inside the cluster. To edit it, you must extract and decompile it:

```bash
# Get the compiled CRUSH map
ceph osd getcrushmap -o /tmp/crushmap.bin

# Decompile to human-readable text
crushtool -d /tmp/crushmap.bin -o /tmp/crushmap.txt
```

Now open the text file to view and edit it:

```bash
cat /tmp/crushmap.txt
```

## Understanding CRUSH Map Structure

A CRUSH map has four main sections:

```text
# devices         - Lists all OSDs
# types           - Defines bucket types (osd, host, rack, etc.)
# buckets          - Describes the hierarchy
# rules            - Placement rules for pools
```

Example snippet:

```text
# devices
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class ssd

# types
type 0 osd
type 1 host
type 2 rack
type 3 row
type 4 room
type 5 datacenter
type 6 root

# buckets
host node1 {
    id -2
    alg straw2
    hash 0
    item osd.0 weight 1.000
    item osd.1 weight 1.000
}

root default {
    id -1
    alg straw2
    hash 0
    item node1 weight 2.000
}

# rules
rule replicated_rule {
    id 0
    type replicated
    min_size 1
    max_size 10
    step take default
    step chooseleaf firstn 0 type host
    step emit
}
```

## Editing the CRUSH Map

### Adding a New Host Bucket

```text
host node2 {
    id -5
    alg straw2
    hash 0
    item osd.3 weight 1.000
    item osd.4 weight 1.000
}
```

Then add the new host to the root:

```text
root default {
    id -1
    alg straw2
    hash 0
    item node1 weight 2.000
    item node2 weight 2.000
}
```

### Adding a Rack-Level Failure Domain

```text
rack rack1 {
    id -10
    alg straw2
    hash 0
    item node1 weight 2.000
    item node2 weight 2.000
}

root default {
    id -1
    alg straw2
    hash 0
    item rack1 weight 4.000
}
```

### Creating a Custom Placement Rule

```text
rule ssd_rule {
    id 5
    type replicated
    min_size 1
    max_size 10
    step take ssd_root
    step chooseleaf firstn 0 type host
    step emit
}
```

## Recompiling and Injecting the CRUSH Map

After editing, compile and inject:

```bash
# Compile the modified CRUSH map
crushtool -c /tmp/crushmap.txt -o /tmp/crushmap_new.bin

# Test the map before applying
crushtool -i /tmp/crushmap_new.bin --test --show-statistics --rule 0 --num-rep 3 --min-x 1 --max-x 1000

# Inject the new CRUSH map
ceph osd setcrushmap -i /tmp/crushmap_new.bin
```

## Verifying the Changes

```bash
# View the current CRUSH map in readable form
ceph osd crush tree

# Check that rules look correct
ceph osd crush rule list
ceph osd crush rule dump <rule-name>
```

## Safety Tips

- Always test the compiled map with `crushtool --test` before injecting
- Keep a backup of the original binary CRUSH map before editing
- Monitor the cluster after injection: `watch -n 5 ceph -s`
- Avoid editing the CRUSH map while the cluster is already degraded

## Summary

Manually editing the Ceph CRUSH map involves extracting the binary map, decompiling it to text, editing the bucket hierarchy and rules, recompiling, and injecting the new map back into the cluster. This process lets you precisely control data placement, define custom failure domains, and optimize storage topology - but should always be tested and validated before applying to a production cluster.
