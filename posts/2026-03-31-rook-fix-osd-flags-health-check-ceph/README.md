# How to Fix OSD_FLAGS Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, OSD, Flag, Configuration

Description: Learn how to resolve the OSD_FLAGS health warning in Ceph when individual OSDs have flags set that alter their behavior outside normal cluster operations.

---

## Understanding OSD_FLAGS

While `OSDMAP_FLAGS` covers cluster-wide flags, `OSD_FLAGS` fires when individual OSDs have flags set that affect their specific behavior. These per-OSD flags can disable scrubbing, prevent the OSD from being marked in or out, or set it to a special state. This warning often appears after manual interventions or failed maintenance operations.

Check current health:

```bash
ceph health detail
```

Example output:

```text
HEALTH_WARN 2 osd(s) have {noup} flag(s) set
[WRN] OSD_FLAGS: 2 osd(s) have {noup} flag(s) set
    osd.3 has flags noup
    osd.7 has flags noup,nodown
```

## Checking Per-OSD Flags

List flags for all OSDs:

```bash
ceph osd dump | grep -E "^osd\." | grep -v "{}$"
```

Or check a specific OSD:

```bash
ceph osd dump | grep "^osd.3"
```

The flags field after the OSD entry shows per-OSD flags.

## Common Per-OSD Flags

| Flag | Effect |
|------|--------|
| `noup` | OSD cannot be brought up |
| `nodown` | OSD cannot be marked down |
| `noin` | OSD cannot be marked in |
| `noout` | OSD cannot be marked out |
| `noscrub` | Scrubbing disabled for this OSD |
| `nodeep-scrub` | Deep scrubbing disabled |

## Clearing Per-OSD Flags

Unset a flag from a specific OSD:

```bash
# Unset noup from osd.3
ceph osd rm-flag 3 noup

# Unset multiple flags
ceph osd rm-flag 3 noup nodown

# Or use the older syntax
ceph osd unset-flag osd.3 noup
```

For multiple OSDs with the same flag, clear them in bulk:

```bash
for osd in 3 7 11; do
  ceph osd rm-flag $osd noup
done
```

Verify flags are cleared:

```bash
ceph osd dump | grep -E "^osd\.(3|7)" | grep -v "{}$"
```

## When OSD Cannot Come Up After Unsetting noup

If an OSD has `noup` set because it was intentionally prevented from starting:

```bash
# Check why the OSD was prevented from starting
ceph osd find 3
journalctl -u ceph-osd@3 | tail -50

# Only unset after resolving the underlying issue
ceph osd rm-flag 3 noup
```

## Bulk Flag Check Script

Use this script to find all OSDs with non-empty flags:

```bash
#!/bin/bash
ceph osd dump | awk '/^osd\./ {
  match($0, /flags ([^,]+)/, arr)
  if (arr[1] != "" && arr[1] != "{}") {
    print "OSD " $1 " has flags: " arr[1]
  }
}'
```

## In Rook Deployments

Rook rarely sets per-OSD flags directly, but they can be set during manual troubleshooting. Check the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- ceph osd dump | grep flags
```

If flags were set via the toolbox during debugging, unset them when done:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- ceph osd rm-flag <osd-id> <flag>
```

## Summary

`OSD_FLAGS` warns that individual OSDs have flags set that modify their behavior. Identify affected OSDs with `ceph osd dump`, then use `ceph osd rm-flag` to clear unwanted flags. Always investigate why a flag was set before removing it - per-OSD flags are often set to prevent a problematic OSD from rejoining prematurely. Only clear flags after confirming the underlying issue is resolved.
