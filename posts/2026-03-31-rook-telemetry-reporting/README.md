# How to Configure Telemetry Reporting in Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Telemetry, Configuration, Monitoring

Description: Configure the Ceph telemetry module in Rook to control anonymous usage reporting and understand what data is shared with the Ceph community.

---

## Overview

Ceph includes a telemetry module that can optionally send anonymized cluster usage statistics to the Ceph project. This data helps the community understand real-world usage patterns and prioritize development. In Rook-managed clusters, you can enable, disable, or preview the telemetry data using the Ceph CLI.

## Check Telemetry Module Status

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph mgr module ls | grep telemetry
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph telemetry status
```

## Preview What Would Be Sent

Before enabling telemetry, preview the report to understand what data is collected:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph telemetry preview
```

The report includes anonymized cluster configuration, component versions, and usage statistics. No personally identifiable information or actual data content is included.

## Enable Telemetry

To opt in to telemetry reporting:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph telemetry on --license sharing-1-0
```

The `--license` flag acknowledges the data sharing agreement. Verify it is enabled:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph telemetry status
```

## Disable Telemetry

To opt out:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph telemetry off
```

## Configure Contact Information (Optional)

You can optionally provide contact details so the Ceph team can reach out about your cluster configuration:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/telemetry/contact "admin@example.com"

kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/telemetry/organization "My Company"
```

## Configure Telemetry Channels

Telemetry data is organized into channels. Control which channels are enabled:

```bash
# List available channels
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph telemetry show

# Enable specific channels
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/telemetry/channel_basic true

kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/telemetry/channel_crash true
```

Available channels include:

```text
channel_basic   - Basic cluster configuration and version info
channel_crash   - Crash reports (anonymized backtraces)
channel_device  - Device health and SMART data
channel_ident   - Optional contact information
```

## Force a Manual Report

To send a report immediately without waiting for the scheduled interval:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph telemetry send
```

## Summary

The Ceph telemetry module in Rook-managed clusters is opt-in and fully transparent. You can preview exactly what would be sent before enabling it, control which data channels are active, and disable it at any time. Opting in helps the Ceph community improve the software based on real-world deployment patterns.
