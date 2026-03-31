# How to Troubleshoot Stretch Mode Configuration Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Stretch Mode, Troubleshooting, Debugging

Description: Troubleshoot common Ceph stretch mode configuration issues including quorum failures, CRUSH misconfigurations, and PG imbalance.

---

## Common Stretch Mode Problems

Stretch mode configuration issues typically fall into three categories: quorum failures, CRUSH rule problems, and pool misconfiguration. This guide covers diagnostics for each.

## Diagnosing Quorum Issues

If the cluster cannot form quorum after enabling stretch mode, check monitor locations:

```bash
ceph mon dump
```

Every monitor must have a `crush_location` set. If any are missing:

```bash
ceph mon set-location mon-dc1a datacenter=dc1
ceph mon set-location mon-dc1b datacenter=dc1
ceph mon set-location mon-dc2a datacenter=dc2
ceph mon set-location mon-dc2b datacenter=dc2
ceph mon set-location mon-arbiter datacenter=arbiter
```

Check the tiebreaker monitor is correctly designated:

```bash
ceph quorum_status --format json-pretty | python3 -c "
import sys, json
d = json.load(sys.stdin)
print('tiebreaker_mon:', d.get('tiebreaker_mon', 'NOT SET'))
print('quorum:', d.get('quorum_names', []))
"
```

## Diagnosing CRUSH Rule Problems

If PGs are stuck in `unknown` or `incomplete` state, the CRUSH rule may be misconfigured:

```bash
ceph osd crush rule dump stretch_rule
```

Verify that the `step chooseleaf` uses `datacenter` type:

```bash
ceph osd crush rule dump stretch_rule | grep "step chooseleaf"
```

Test the CRUSH rule mapping:

```bash
ceph osd map <pool> <object>
```

If the OSD set returned does not include OSDs from both sites, the rule is wrong. Re-create it:

```bash
ceph osd crush rule rm stretch_rule
ceph osd crush rule create-replicated stretch_rule default datacenter osd
```

## Diagnosing Pool Misconfiguration

Pools must have size=4 and min_size=2 for stretch mode:

```bash
ceph osd pool ls detail | grep -E "size|min_size|crush"
```

Fix any pools with wrong settings:

```bash
for pool in $(ceph osd pool ls); do
  ceph osd pool set $pool size 4
  ceph osd pool set $pool min_size 2
  ceph osd pool set $pool crush_rule stretch_rule
done
```

## PGs Stuck in Incomplete State

If PGs are stuck `incomplete`, the cluster may have lost too many OSDs:

```bash
ceph pg dump stuck inactive
ceph pg repair <pgid>
```

Check which OSDs an affected PG uses:

```bash
ceph pg <pgid> query
```

## Viewing Stretch Mode Flags

Check which stretch flags are set in the OSD map:

```bash
ceph osd dump | grep -E "stretch|flags"
```

Expected output when stretch mode is correctly enabled:

```text
flags stretch_mode_enabled
```

## Disabling Stretch Mode for Debugging

If stretch mode must be disabled temporarily for debugging (not recommended in production):

```bash
ceph osd unset stretch_mode_enabled
```

Re-enable with:

```bash
ceph mon enable_stretch_mode mon-arbiter stretch_rule datacenter
```

## Summary

Troubleshooting Ceph stretch mode starts with verifying monitor CRUSH locations and tiebreaker designation, then inspecting CRUSH rules and pool configuration. Most issues stem from missing location labels or pools that were not updated to use the stretch rule. Systematic checks of each layer resolve the majority of stretch mode configuration problems.
