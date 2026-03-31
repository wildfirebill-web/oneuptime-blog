# How to Test Ceph Release Candidates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Release Candidate, Testing, Upgrade

Description: Test Ceph release candidates in a staging environment by deploying RC packages, running functional tests, and reporting findings to the Ceph community before GA.

---

Testing Ceph release candidates (RCs) helps the community find regressions before they reach production users. You benefit too - testing RCs on a staging cluster gives you early familiarity with new features and behavior changes.

## Finding Release Candidate Packages

Ceph RC packages are published to the official repositories with an RC suffix:

```bash
# Check available RC versions
curl https://download.ceph.com/rpm-testing/ 2>/dev/null | grep -i "rc\|candidate"

# Ubuntu/Debian testing repo
# https://download.ceph.com/debian-testing/
```

For containerized deployments, RC images are on Quay.io:

```bash
# List available RC tags for Ceph container images
podman search quay.io/ceph/ceph --list-tags 2>/dev/null | grep -i "rc\|dev"
```

## Deploying with Rook

To test an RC with Rook, update the CephCluster spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19.1.0-rc1
    allowUnsupported: true
```

Note: `allowUnsupported: true` is required for pre-GA versions.

## Setting Up a Test Environment

Use a dedicated namespace or cluster for RC testing:

```bash
kubectl create namespace rook-ceph-test
# Deploy Rook operator in test namespace
helm install rook-ceph rook-release/rook-ceph -n rook-ceph-test \
  --set "image.tag=v1.17.0-alpha.0"
```

## Running Functional Tests

After deploying the RC, run a standard set of validation tests:

```bash
# Basic health
ceph status
ceph health detail

# OSD functionality
rados bench -p rbd 30 write --no-cleanup
rados bench -p rbd 30 seq
rados bench -p rbd 30 rand

# RBD operations
rbd create --size 1024 rbd/test-image
rbd bench rbd/test-image --io-type write --io-size 4096 --io-total 1G
rbd rm rbd/test-image
```

## Testing Upgrade Paths

A critical test is upgrading from the previous stable release to the RC:

```bash
# Start with stable version, then upgrade to RC
# Update CephCluster spec to point to RC image
kubectl edit cephcluster -n rook-ceph-test rook-ceph
# Change: image: quay.io/ceph/ceph:v18.2.2
# To:     image: quay.io/ceph/ceph:v19.1.0-rc1

# Watch upgrade progress
watch -n 10 ceph status
kubectl get pods -n rook-ceph-test -w
```

## Reporting Results

Report both positive results and regressions to the ceph-users mailing list and the Ceph tracker:

- Subject: `[RC Testing] Ceph 19.1.0-rc1 on Ubuntu 22.04 - PASS/FAIL`
- Include: version, OS, test steps, results, and any relevant log excerpts

## Summary

Testing Ceph RCs requires a staging environment, RC packages or container images, and a standard set of read/write and upgrade tests. Reporting results - whether a pass or a failure - to the mailing list and tracker is the most direct way to contribute to Ceph quality without writing code.
