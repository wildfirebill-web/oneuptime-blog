# How to Test Ceph HA by Simulating Component Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, High Availability, Testing, Chaos Engineering, OSD

Description: Simulate OSD, Monitor, and node failures in a Rook-managed Ceph cluster to validate HA configurations and recovery procedures before production incidents.

---

Testing failure scenarios before they happen in production is essential for validating HA configurations. Rook-managed Ceph clusters allow safe simulation of component failures in staging environments.

## Simulating an OSD Failure

Stop a single OSD pod to simulate a disk or OSD process failure:

```bash
# Identify the OSD deployment
kubectl -n rook-ceph get deploy | grep osd

# Scale down one OSD
kubectl -n rook-ceph scale deploy/rook-ceph-osd-0 --replicas=0
```

Observe cluster health response:

```bash
watch kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

Restore the OSD and watch rebalancing complete:

```bash
kubectl -n rook-ceph scale deploy/rook-ceph-osd-0 --replicas=1
```

## Simulating a Monitor Failure

Kill one monitor pod to test quorum resilience:

```bash
kubectl -n rook-ceph delete pod -l app=rook-ceph-mon,mon=b
```

Verify quorum is maintained with two remaining monitors:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph quorum_status --format json-pretty | python3 -m json.tool
```

Rook will automatically recreate the deleted monitor pod.

## Simulating a Full Node Failure

Cordon and drain a node to simulate a full node outage:

```bash
kubectl cordon node2
kubectl drain node2 --ignore-daemonsets --delete-emptydir-data
```

Monitor how Ceph responds and reassigns work:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

Uncordon the node when done:

```bash
kubectl uncordon node2
```

## Write Data During a Failure

Use an S3 benchmark tool to confirm writes succeed during a failure:

```bash
kubectl run s3bench --image=minio/mc --rm -it -- \
  sh -c "mc alias set mys3 http://rook-ceph-rgw-my-store.rook-ceph:80 ACCESS SECRET && \
         mc mb mys3/testbucket && \
         for i in \$(seq 1 100); do echo test | mc pipe mys3/testbucket/obj-\$i; done"
```

## Track Recovery Time

Record time from failure to full health:

```bash
# Note start time, then wait for HEALTH_OK
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail --format json | python3 -c \
  "import sys,json; h=json.load(sys.stdin); print(h['status'])"
```

## Summary

Regularly simulating OSD, monitor, and node failures in a staging cluster validates that Ceph HA configurations work as expected and helps teams practice recovery procedures. Scale down OSD deployments, delete monitor pods, and drain nodes incrementally to measure actual recovery times and identify configuration gaps.
