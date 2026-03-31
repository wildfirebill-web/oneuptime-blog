# How to Create CRUSH Rules for Replicated Pools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Storage, Rule

Description: Learn how to create and apply Ceph CRUSH rules for replicated pools to control data placement across specific failure domains and device classes.

---

## What are CRUSH Rules for Replicated Pools

CRUSH rules for replicated pools define how Ceph selects OSDs when storing replicated data. A rule specifies the starting point in the topology (usually the root bucket), the type of failure domain for spreading replicas (host, rack, datacenter), and optionally filters replicas to a specific device class.

Each replicated pool has exactly one CRUSH rule. Choosing the right rule determines your cluster's resilience and performance characteristics.

## Viewing Existing CRUSH Rules

```bash
# List all CRUSH rules
ceph osd crush rule ls

# Get details on all rules
ceph osd crush rule dump

# Get details on a specific rule
ceph osd crush rule dump replicated_rule
```

Example rule output:

```json
{
    "rule_id": 0,
    "rule_name": "replicated_rule",
    "type": 1,
    "steps": [
        {"op": "take", "item_name": "default"},
        {"op": "chooseleaf_firstn", "num": 0, "type": "host"},
        {"op": "emit"}
    ]
}
```

## Creating a Basic Replicated Rule

The simplest way to create a rule is with the `create-replicated` shorthand:

```bash
# Create a rule spreading replicas across different hosts
ceph osd crush rule create-replicated host-rule default host

# Create a rule spreading replicas across different racks
ceph osd crush rule create-replicated rack-rule default rack

# Create a rule using only SSD OSDs, separated by host
ceph osd crush rule create-replicated ssd-host-rule default host ssd
```

Parameters:
- Rule name
- Root bucket to start from (`default` in most clusters)
- Failure domain type
- Optional device class

## Creating Advanced Rules via CRUSH Map

For more complex placement scenarios, edit the CRUSH map directly:

```bash
ceph osd getcrushmap -o crush.bin
crushtool -d crush.bin -o crush.txt
```

Add a custom rule to `crush.txt`:

```text
rule datacenter-replicated {
    id 5
    type replicated
    step take default
    step choose firstn 3 type datacenter
    step chooseleaf firstn 1 type host
    step emit
}
```

This rule places one replica in each of 3 different datacenters, choosing one host within each.

```bash
crushtool -c crush.txt -o crush-new.bin
ceph osd setcrushmap -i crush-new.bin
```

## Applying a Rule to a Pool

```bash
# Set the CRUSH rule on an existing pool
ceph osd pool set mypool crush_rule rack-rule

# Specify the rule when creating a new pool
ceph osd pool create mypool 64 64 replicated rack-rule

# Verify the rule assignment
ceph osd pool get mypool crush_rule
```

## Testing Rule Behavior

Before applying, test how a rule distributes data:

```bash
# Test the rule and show OSD selection statistics
crushtool -i crush.bin --test \
  --rule 1 \
  --num-rep 3 \
  --min-x 0 \
  --max-x 1000 \
  --show-statistics

# Verify rule works with current OSD tree
ceph osd map mypool testobject
```

## Deleting a Rule

```bash
# Remove an unused CRUSH rule
ceph osd crush rule rm old-rule

# Note: rules in use by pools cannot be deleted until the pool is changed
ceph osd pool get mypool crush_rule
```

## Summary

CRUSH rules for replicated pools control which OSDs receive each replica and across which failure domains they are spread. Use `ceph osd crush rule create-replicated` for common patterns, or edit the CRUSH map directly for advanced multi-level placement. Always test rules with `crushtool --test` before applying them to production pools to verify the distribution behavior.
