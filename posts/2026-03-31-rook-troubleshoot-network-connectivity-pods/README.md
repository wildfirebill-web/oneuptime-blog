# How to Troubleshoot Network Connectivity Between Rook-Ceph Pods

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Networking, Troubleshooting, Kubernetes, Connectivity

Description: Diagnose and resolve network connectivity issues between Rook-Ceph pods including monitor connectivity, OSD communication, and client access failures.

---

## Overview

Network connectivity issues between Rook-Ceph components can manifest as slow I/O, cluster health warnings, or complete service unavailability. This guide covers systematic troubleshooting of inter-pod communication in Rook-Ceph clusters.

## Step 1: Check Cluster Health

Start with the overall cluster health to identify the scope of issues:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph -s

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail
```

Common network-related warnings:
- `MON_DOWN` - monitors cannot communicate
- `OSD_DOWN` - OSDs unreachable
- `SLOW_OPS` - operations timing out

## Step 2: Test Pod-to-Pod Connectivity

Verify basic connectivity between Ceph components:

```bash
# Get monitor IPs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon dump

# Test TCP connectivity from tools pod to a monitor
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  nc -zv <mon-ip> 6789
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  nc -zv <mon-ip> 3300
```

## Step 3: Inspect Pod Network Configuration

Check that pods have the expected IPs and network interfaces:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon -o wide
kubectl -n rook-ceph get pods -l app=rook-ceph-osd -o wide

# Check network interfaces inside a pod
kubectl -n rook-ceph exec -it <osd-pod> -- ip addr show
kubectl -n rook-ceph exec -it <osd-pod> -- ip route show
```

## Step 4: Check CNI and Network Policies

Verify no network policies are blocking Ceph traffic:

```bash
kubectl -n rook-ceph get networkpolicies
kubectl get networkpolicies -A | grep -v "NAMESPACE"
```

Test with a simple ping:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ping -c4 <osd-pod-ip>
```

## Step 5: Diagnose DNS Resolution

Ceph services depend on DNS for component discovery:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  nslookup rook-ceph-mon-a.rook-ceph.svc.cluster.local

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  curl -v http://rook-ceph-mgr.rook-ceph.svc.cluster.local:9283/metrics
```

## Step 6: Check Service Endpoints

Verify Kubernetes services have healthy endpoints:

```bash
kubectl -n rook-ceph get endpoints
kubectl -n rook-ceph describe svc rook-ceph-mon-a
kubectl -n rook-ceph describe svc rook-ceph-mgr
```

## Step 7: Review Pod Logs for Network Errors

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-mon --tail=100 | \
  grep -E "connection|timeout|refused|unreachable"

kubectl -n rook-ceph logs -l app=rook-ceph-osd --tail=100 | \
  grep -E "connection|timeout|refused"
```

## Step 8: Packet Capture for Deep Analysis

For complex issues, capture packets inside a pod:

```bash
kubectl -n rook-ceph exec -it <osd-pod> -- \
  tcpdump -i eth0 -w /tmp/ceph-capture.pcap port 6800
```

Copy and analyze:

```bash
kubectl cp rook-ceph/<osd-pod>:/tmp/ceph-capture.pcap ./capture.pcap
wireshark capture.pcap
```

## Summary

Troubleshooting Rook-Ceph network connectivity requires checking cluster health, testing TCP connectivity between component IPs, verifying DNS resolution, and inspecting network policies. Start with `ceph -s` to identify the scope, then drill down into specific component connectivity using `nc`, `ping`, and pod logs.
