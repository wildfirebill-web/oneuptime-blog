# How to Create Change Management Processes for Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Change Management, Kubernetes, Storage

Description: Learn how to establish structured change management processes for Ceph clusters to reduce risk, improve reliability, and maintain audit trails for storage changes.

---

## Why Change Management Matters for Ceph

Ceph clusters are stateful, shared infrastructure. An unplanned configuration change - such as modifying replication size or CRUSH rules - can cause data unavailability or performance degradation. A formal change management process gives teams a structured way to review, approve, and roll back changes.

## Define Change Categories

Not all changes carry the same risk. Categorize changes before applying them:

- **Low risk**: Updating dashboard credentials, adjusting alert thresholds
- **Medium risk**: Adding OSDs, creating new pools, modifying RGW user quotas
- **High risk**: Modifying CRUSH maps, changing replication factors, upgrading Ceph versions

Document these categories in a shared wiki or runbook so all operators agree on what triggers a full review process.

## Create a Change Request Template

Store change requests in version control (e.g., a Git repository) using a standard template:

```yaml
# change-request-001.yaml
id: CR-001
date: "2026-03-31"
author: "nawazdhandala"
category: medium
summary: "Add 3 new OSDs to ceph-cluster-prod"
cluster: ceph-cluster-prod
namespace: rook-ceph
pre_checks:
  - "ceph health OK"
  - "all OSDs up and in"
  - "pg status active+clean"
rollback_plan: "Remove OSD entries from CephCluster spec and run osd remove"
approval_required: true
approved_by: ""
```

Committing change requests to Git gives you a built-in audit trail.

## Pre-Change Health Checks

Always verify cluster health before making changes. Create a checklist script:

```bash
#!/bin/bash
# pre-change-check.sh
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  echo '=== Ceph Health ==='
  ceph health detail
  echo '=== OSD Status ==='
  ceph osd stat
  echo '=== PG Status ==='
  ceph pg stat
  echo '=== Mon Quorum ==='
  ceph quorum_status --format json | jq '.quorum_names'
"
```

Only proceed with the change if all checks pass.

## Apply Changes via GitOps

Use Rook's CRDs with a GitOps workflow (e.g., ArgoCD or Flux) so changes are code-reviewed before reaching the cluster:

```yaml
# cephcluster-patch.yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    nodes:
      - name: "worker-node-4"
        devices:
          - name: "sdb"
```

Submit this as a pull request. Require at least one reviewer with Ceph expertise before merging.

## Post-Change Validation

After a change, validate the cluster has returned to a healthy state:

```bash
# Watch cluster recovery
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph -w --format json

# Confirm PGs are active+clean
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg stat
```

Document the outcome in the change request record and close it once stability is confirmed.

## Track Changes in a Register

Maintain a simple CSV or database of all completed changes:

```bash
# Sample change register format
echo "CR-001,2026-03-31,medium,Add 3 OSDs,success,nawazdhandala" >> change-register.csv
```

This register helps with incident investigation, capacity planning, and audits.

## Summary

Creating change management processes for Ceph involves categorizing risk, using standardized templates stored in Git, performing pre- and post-change health checks, and maintaining a change register. By coupling these processes with a GitOps workflow, teams reduce the chance of cluster disruptions and maintain a clear audit trail for every storage change.
