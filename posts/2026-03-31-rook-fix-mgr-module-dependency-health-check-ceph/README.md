# How to Fix MGR_MODULE_DEPENDENCY Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Manager, Module, Dependency

Description: Learn how to resolve the MGR_MODULE_DEPENDENCY health warning in Ceph when a manager module has unmet dependencies preventing it from loading correctly.

---

## Understanding MGR_MODULE_DEPENDENCY

`MGR_MODULE_DEPENDENCY` fires when a Ceph MGR module cannot load because one of its dependencies is missing or incompatible. This could mean a required Python library is absent, a required sibling module is disabled, or the module requires a specific Ceph version feature that is not available.

Check current health:

```bash
ceph health detail
```

Example output:

```text
HEALTH_WARN mgr module dependency not met
[WRN] MGR_MODULE_DEPENDENCY: Module balancer has failed dependency on module stats
    Module 'balancer' depends on 'stats' which is not enabled
```

## Listing Module Status

Check the status of all MGR modules:

```bash
ceph mgr module ls
```

This shows enabled, disabled, and always-on modules. Look for modules that are enabled but failing:

```bash
ceph mgr module ls | python3 -m json.tool | grep -A5 enabled_modules
```

## Identifying the Dependency Chain

Some modules have explicit dependencies. Check the module's requirements:

```bash
ceph mgr metadata | python3 -m json.tool
```

Common dependency relationships:
- `balancer` depends on `stats` being enabled
- `pg_autoscaler` may depend on `prometheus` for feedback
- `dashboard` depends on `restful` and `telemetry`

## Enabling Missing Dependencies

Enable the missing dependency module:

```bash
# Example: enable the stats module that balancer needs
ceph mgr module enable stats

# Enable restful for dashboard dependency
ceph mgr module enable restful
```

Verify after enabling:

```bash
ceph mgr module ls | grep -E "stats|restful"
```

## Fixing Python Library Dependencies

Some modules require Python libraries installed on the MGR host. In Rook, the MGR runs in a container - missing libraries should not occur unless using a custom image.

Check for import errors in MGR logs:

```bash
kubectl -n rook-ceph logs <mgr-pod> | grep -i "ImportError\|ModuleNotFoundError"
```

If a library is missing, switch to the official Ceph container image:

```bash
kubectl -n rook-ceph patch cephcluster rook-ceph --type=merge \
  -p '{"spec":{"cephVersion":{"image":"quay.io/ceph/ceph:v18.2.0"}}}'
```

## Disabling the Dependent Module

If the dependency cannot be satisfied (e.g., the required module is deprecated), disable the dependent module:

```bash
ceph mgr module disable balancer
```

Verify cluster health improves:

```bash
ceph health detail
```

## Re-enabling in the Correct Order

If multiple modules have a dependency chain, enable them in order:

```bash
# Enable base modules first
ceph mgr module enable stats
ceph mgr module enable iostat

# Then enable dependent modules
ceph mgr module enable balancer
ceph mgr module enable pg_autoscaler
```

## Checking Module Errors After Enabling

After enabling dependencies, check for remaining errors:

```bash
ceph mgr module ls
ceph tell mgr* mgr_status
```

Any module that failed to load will appear with a failed status.

## Summary

`MGR_MODULE_DEPENDENCY` fires when a MGR module's required dependencies are not met. Identify the failing module and its dependencies using `ceph mgr module ls`, then enable missing dependency modules in the correct order. If a Python library is missing, ensure you are using the official Ceph container image. If the required module is deprecated, disable the dependent module to clear the health warning.
