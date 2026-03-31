# How to Troubleshoot Ceph Manager Module Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Troubleshooting, Ceph Manager, Debugging

Description: Learn how to diagnose and fix common Ceph Manager module errors including failed starts, configuration issues, and Python dependency problems.

---

Ceph Manager module errors can leave features like monitoring integrations or dashboard access unavailable. This guide covers systematic approaches to diagnosing and resolving module failures, from health warnings to Python exceptions.

## Identifying Module Failures

Check cluster health for module-related warnings:

```bash
ceph health detail
```

Example output indicating a failed module:

```text
HEALTH_WARN Module 'influx' has failed: No module named 'influxdb'
```

List module states to identify which are failing:

```bash
ceph mgr module ls --format json | python3 -m json.tool
```

## Reading Manager Logs

Manager module errors appear in the Ceph log:

```bash
ceph log last 50 | grep -i "module\|mgr\|error\|exception"
```

On manager nodes, check the daemon log directly:

```bash
tail -f /var/log/ceph/ceph-mgr.*.log
```

Filter for Python tracebacks:

```bash
grep -A 10 "Traceback" /var/log/ceph/ceph-mgr.*.log
```

## Checking Python Dependencies

A common module failure cause is missing Python packages. The `can_run` field in the module list shows this:

```bash
ceph mgr module ls --format json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for m in data.get('disabled_modules', []):
    if not m.get('can_run', True):
        print(f\"{m['name']}: {m.get('error_string', 'unknown error')}\")"
```

Install missing packages on the manager node:

```bash
pip3 install influxdb requests
```

Or in Rook environments, add packages to the manager image by building a custom container image.

## Restarting a Failed Module

Disable and re-enable a module to restart it:

```bash
ceph mgr module disable prometheus
ceph mgr module enable prometheus
```

## Forcing a Manager Failover

If the active manager is stuck with a failed module, fail over to a standby:

```bash
ceph mgr fail
```

The standby manager takes over, and modules re-initialize fresh.

## Validating Module Configuration

Missing or incorrect module configuration is another common failure cause. Check current settings:

```bash
ceph config dump | grep mgr
```

Reset a misconfigured option to its default:

```bash
ceph config rm mgr mgr/influx/hostname
```

## Debugging with Increased Log Level

Increase the log level for the mgr to get verbose module output:

```bash
ceph tell mgr.* injectargs --debug-mgr 10
```

Then watch logs for detailed module activity:

```bash
tail -f /var/log/ceph/ceph-mgr.*.log | grep -i "module_name"
```

Reset log level after debugging:

```bash
ceph tell mgr.* injectargs --debug-mgr 1
```

## Summary

Troubleshooting Ceph Manager module errors starts with `ceph health detail` to identify the failing module, followed by log inspection with `ceph log last` or the manager daemon log for Python tracebacks. Missing Python dependencies, incorrect configuration, and network connectivity issues to external services are the most common root causes, and forcing a manager failover with `ceph mgr fail` is a quick recovery option.
