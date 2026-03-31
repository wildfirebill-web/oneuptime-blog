# How to Troubleshoot Ceph Dashboard Access Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Dashboard, Troubleshooting, Kubernetes

Description: Learn how to diagnose and fix common Ceph dashboard access problems including SSL errors, authentication failures, and network connectivity issues.

---

## Common Ceph Dashboard Access Problems

The Ceph dashboard is a critical management interface, but accessing it can sometimes be tricky in Kubernetes environments managed by Rook. This guide walks through the most common issues and how to resolve them.

## Verify the Dashboard Service is Running

Start by checking whether the dashboard service and pods are healthy:

```bash
kubectl -n rook-ceph get svc rook-ceph-mgr-dashboard
kubectl -n rook-ceph get pods -l app=rook-ceph-mgr
```

If the MGR pod is not in `Running` state, check its logs:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-mgr --tail=100
```

## Check Dashboard is Enabled in the CephCluster

Ensure the dashboard is enabled in your CephCluster spec:

```yaml
spec:
  dashboard:
    enabled: true
    ssl: true
    port: 8443
```

Apply changes after editing:

```bash
kubectl -n rook-ceph apply -f cluster.yaml
```

## Retrieve the Default Admin Password

If you can reach the dashboard but cannot log in, retrieve the generated admin password:

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath='{.data.password}' | base64 --decode
```

The default username is `admin`.

## Fix SSL Certificate Errors

Many browsers block self-signed certificates. You can either import the certificate or disable SSL for testing:

```yaml
spec:
  dashboard:
    enabled: true
    ssl: false
```

For production, configure a proper TLS secret:

```yaml
spec:
  dashboard:
    enabled: true
    ssl: true
    urlPrefix: /ceph-dashboard
```

Then set up an Ingress with your certificate manager.

## Check Network Policies

If your cluster uses network policies, ensure the dashboard port (default 8443 or 7000) is allowed:

```bash
kubectl -n rook-ceph get networkpolicies
```

Test reachability by running a temporary debug pod:

```bash
kubectl -n rook-ceph run debug --image=curlimages/curl --rm -it -- \
  curl -k https://rook-ceph-mgr-dashboard:8443
```

## Verify MGR Module is Active

The dashboard is a MGR module. Confirm it is active:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module ls | grep dashboard
```

If it shows as disabled, enable it:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module enable dashboard
```

## Reset Dashboard User Password

If you need to reset credentials manually:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph dashboard ac-user-set-password admin <new-password>
```

## Summary

Ceph dashboard access issues typically fall into three categories: pod/service health, SSL certificate handling, and authentication problems. Systematically verifying each layer - from pod status to MGR module state to network connectivity - allows you to isolate and fix the issue quickly. Always check MGR pod logs first as they reveal most configuration and startup errors.
