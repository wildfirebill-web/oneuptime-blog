# How to Use Sticky Mute for Health Checks in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Health, Monitoring, Maintenance

Description: Use the sticky mute feature in Ceph to prevent a health check from reactivating even if the condition resolves and recurs within the mute period.

---

Ceph's sticky mute feature is an advanced variant of health check muting that keeps the mute in place even if the underlying condition temporarily clears and then recurs. This is useful for intermittent issues that oscillate between healthy and warning states.

## Standard Mute vs Sticky Mute

**Standard mute**: The mute is removed when the health check clears. If the same condition returns, the health check becomes visible again.

**Sticky mute**: The mute persists even if the condition clears and returns. It only expires when the TTL expires or when manually removed with `unmute`.

## When to Use Sticky Mute

Use sticky mute for:
- Intermittent OSD connectivity issues that bounce between up and down
- Clock skew warnings on a node where NTP is being stabilized
- PG degradation that cycles during an ongoing recovery operation
- Any situation where you want to silence a check for the full duration of a maintenance window regardless of transient state changes

## Enable Sticky Mute

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health mute <HEALTH_CHECK_CODE> --sticky
```

Combine sticky with a TTL:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health mute OSD_DOWN 4h --sticky
```

Example - sticky mute for monitor clock skew during NTP configuration:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health mute MON_CLOCK_SKEW 2h --sticky
```

## Verify a Mute is Sticky

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health detail --format json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for mute in data.get('mutes', []):
    sticky = mute.get('sticky', False)
    ttl = mute.get('ttl', 'indefinite')
    print(f'{mute[\"code\"]}: sticky={sticky}, ttl={ttl}')
"
```

Sample output:

```text
OSD_DOWN: sticky=True, ttl=3h 47m
MON_CLOCK_SKEW: sticky=True, ttl=1h 52m
```

## Unmute a Sticky Mute

Sticky mutes are removed the same way as regular mutes:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health unmute OSD_DOWN
```

## Practical Example: OSD Replacement

During a disk replacement, the OSD will cycle up and down multiple times. A sticky mute prevents alert fatigue:

```bash
# Start disk replacement
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health mute OSD_DOWN 6h --sticky

# Proceed with disk replacement
# (osd.3 will go down, come up, go down again during replacement)

# After replacement is complete and OSD is stable
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health unmute OSD_DOWN
```

Without sticky, every time `osd.3` bounces, the `OSD_DOWN` warning would briefly reappear and trigger alerts.

## Sticky Mute vs Standard Mute Behavior

```text
Scenario: OSD goes down, recovers, goes down again

Standard mute:
  10:00 - OSD down   -> mute active (suppressed)
  10:15 - OSD up     -> mute REMOVED (condition cleared)
  10:20 - OSD down   -> WARNING VISIBLE AGAIN

Sticky mute (4h TTL):
  10:00 - OSD down   -> mute active (suppressed)
  10:15 - OSD up     -> mute remains active
  10:20 - OSD down   -> still suppressed
  14:00 - TTL expires -> mute removed
```

## Summary

Sticky mute (`--sticky` flag) in Ceph keeps a health check suppressed for the full mute duration even if the underlying condition oscillates between present and absent. Use it during maintenance operations where components will cycle through states, combining it with a TTL to ensure the mute eventually expires. Remove sticky mutes manually with `ceph health unmute` after maintenance completes.
