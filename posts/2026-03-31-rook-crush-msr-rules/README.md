# How to Use CRUSH MSR Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Rule, Storage

Description: Learn how to use Ceph CRUSH Multi-Step Replication (MSR) rules to implement complex multi-level placement strategies for advanced topology requirements.

---

## What are CRUSH MSR Rules

CRUSH Multi-Step Replication (MSR) rules allow you to define complex placement policies that go beyond single-level chooseleaf operations. MSR rules use multiple `choose` and `chooseleaf` steps to build a placement path through multiple hierarchy levels, enabling sophisticated strategies like "2 replicas in datacenter A and 1 replica in datacenter B" or "primary on SSD, secondaries on HDD."

MSR rules are type `replicated` rules but use multiple steps to achieve multi-level selection.

## Basic MSR Rule Structure

An MSR rule in the CRUSH map uses a sequence of `choose` and `chooseleaf` steps:

```text
rule msr-two-plus-one {
    id 10
    type replicated
    step take default
    step choose firstn 2 type datacenter
    step chooseleaf firstn 1 type host
    step emit
    step take default
    step choose firstn 1 type datacenter
    step chooseleaf firstn 1 type host
    step emit
}
```

Wait - the above uses multiple `emit` steps which is not standard. Standard MSR uses a more sequential approach:

```text
rule dc-spread-rule {
    id 10
    type replicated
    step take default
    step choose firstn 2 type datacenter
    step chooseleaf firstn 2 type host
    step emit
}
```

## Step-by-Step Rule Construction

Each step in a CRUSH rule narrows the selection:

```bash
# Export and decompile the CRUSH map
ceph osd getcrushmap -o crush.bin
crushtool -d crush.bin -o crush.txt
```

Add to `crush.txt`:

```text
rule primary-ssd-secondary-hdd {
    id 11
    type replicated
    step take default class ssd
    step chooseleaf firstn 1 type host
    step emit
    step take default class hdd
    step chooseleaf firstn 2 type host
    step emit
}
```

This places the primary replica on an SSD OSD and two secondary replicas on HDD OSDs.

```bash
crushtool -c crush.txt -o crush-new.bin

# Test the rule behavior
crushtool -i crush-new.bin --test \
  --rule 11 --num-rep 3 \
  --min-x 0 --max-x 100 \
  --show-statistics
```

## MSR for Stretched Clusters

A common use case is a 2+1 stretched cluster rule:

```text
rule stretched-2-1 {
    id 12
    type replicated
    # Place 2 replicas in primary datacenter
    step take dc-primary
    step chooseleaf firstn 2 type host
    step emit
    # Place 1 replica in secondary datacenter
    step take dc-secondary
    step chooseleaf firstn 1 type host
    step emit
}
```

## Applying MSR Rules to Pools

```bash
crushtool -c crush.txt -o crush-new.bin
ceph osd setcrushmap -i crush-new.bin

# Apply to a pool
ceph osd pool set critical-pool crush_rule stretched-2-1

# Verify rule assignment
ceph osd pool get critical-pool crush_rule

# Check PG placement
ceph osd map critical-pool testobject
```

## Testing MSR Rule Distribution

```bash
# Simulate placement and review statistics
crushtool -i crush-new.bin \
  --test --rule 12 \
  --num-rep 3 \
  --min-x 0 --max-x 10000 \
  --show-statistics 2>&1 | tail -20

# Verify each replica lands in the correct datacenter
# The output shows which OSDs handle each PG
```

## Summary

CRUSH MSR rules enable sophisticated multi-level placement strategies by using multiple `take...chooseleaf...emit` sequences within a single rule. Common use cases include placing replicas across different device classes (SSD primary, HDD secondaries) or across specific CRUSH roots (primary datacenter plus secondary datacenter). Always test MSR rules with `crushtool --test` before applying them to production pools to verify the intended distribution behavior.
