# How to Understand What Data Ceph Telemetry Collects

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Telemetry, Privacy, Data Collection, Storage

Description: Understand exactly what data the Ceph telemetry module collects by channel, how it is anonymized, and how to inspect the payload before opting in.

---

## Overview of Telemetry Data Categories

Ceph telemetry collects data in distinct channels. Each channel covers a specific category of information. Before opting in, you can preview each channel's payload to understand what is shared.

Preview all telemetry data:

```bash
ceph telemetry show
```

Preview a specific channel:

```bash
ceph telemetry show basic
ceph telemetry show crash
ceph telemetry show device
```

## Basic Channel - What Is Collected

The `basic` channel includes anonymized cluster topology and configuration:

- Ceph version and release
- Number of monitors, OSDs, and MDSs
- OSD and pool configurations (without hostnames or IPs)
- Cluster ID (a UUID, not tied to any personally identifiable information)
- Enabled features and flags
- CRUSH map structure (anonymized)

Example of basic telemetry output:

```bash
ceph telemetry show basic | python3 -m json.tool | head -50
```

## Crash Channel - What Is Collected

The `crash` channel sends anonymized crash reports:

- Crash timestamp and Ceph version
- Stack trace from the crash
- Daemon type (OSD, MON, MGR) that crashed
- No hostname, IP address, or cluster-specific data

View current crash reports:

```bash
ceph crash ls
ceph crash info <crash-id>
```

## Device Channel - What Is Collected

The `device` channel shares SMART health data predictions:

- Drive model and firmware version
- SMART attributes used for health scoring
- Predicted failure probability
- No serial numbers or host associations

Check device health data:

```bash
ceph device ls
ceph device info <device-id>
```

## What Is NOT Collected

Ceph telemetry explicitly excludes:
- Hostnames and IP addresses
- Object names, bucket names, or user data
- Ceph user account names
- RGW, CephFS, or RBD client data
- Any data stored in the cluster

## Inspecting the Full Telemetry Report

Generate the complete JSON report that would be sent:

```bash
ceph telemetry show --format json > /tmp/telemetry-preview.json
cat /tmp/telemetry-preview.json | python3 -m json.tool | less
```

Count items by section:

```bash
ceph telemetry show --format json | python3 -c "
import sys, json
d = json.load(sys.stdin)
for k, v in d.items():
    print(f'{k}: {type(v).__name__}')
"
```

## Rook Toolbox Commands

Access telemetry inspection from the toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph telemetry show basic
```

## Summary

Ceph telemetry collects only anonymized cluster structure and health data - no user data, hostnames, or personally identifiable information is ever sent. The `basic`, `crash`, and `device` channels each cover distinct aspects of cluster metadata. Running `ceph telemetry show` before opting in lets you verify the payload is acceptable for your organization's data policies.
