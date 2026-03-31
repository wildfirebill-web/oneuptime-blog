# How to Configure RGW Gateway Settings (Port, SecurePort, Instances) in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Object Storage, Kubernetes

Description: Learn how to configure Rook RGW gateway settings including port, securePort, and instance count to tune your S3-compatible object store deployment.

---

## Overview of RGW Gateway Configuration

The Ceph RADOS Gateway (RGW) is the S3-compatible HTTP endpoint for your Rook object store. Configuring gateway settings correctly affects how clients connect, how many concurrent requests can be handled, and whether TLS is terminated at the gateway or an upstream load balancer.

The `gateway` section of the `CephObjectStore` CRD controls these settings.

## Setting Port and SecurePort

The `port` field sets the HTTP port for unencrypted connections. The `securePort` enables HTTPS directly on RGW using a TLS certificate:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
  gateway:
    port: 80
    securePort: 443
    sslCertificateRef: rgw-tls-secret
    instances: 2
```

Set `securePort` only when you also provide `sslCertificateRef`. If you terminate TLS at an ingress controller, set only `port` and leave `securePort` unset.

## Configuring Multiple Instances for High Availability

The `instances` field controls how many RGW pods are deployed. Each instance runs independently and can handle requests:

```yaml
gateway:
  port: 80
  instances: 3
```

For production, use at least 2 instances for redundancy. With 3+ instances, rolling upgrades can proceed without downtime. The pods are distributed across nodes based on pod anti-affinity rules that Rook applies automatically.

## Setting Resource Requests and Limits

Tune gateway resource usage to match your workload:

```yaml
gateway:
  port: 80
  instances: 2
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "4"
      memory: "4Gi"
```

Inadequate memory limits can cause OOM kills under heavy load. Start with 2 GiB and profile your workload.

## Configuring Prioritization

For critical workloads, set a `priorityClassName` to protect RGW pods from eviction:

```yaml
gateway:
  port: 80
  instances: 2
  priorityClassName: system-cluster-critical
```

## Checking Gateway Status

After applying the CRD, verify the pods and service are running:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-rgw
kubectl -n rook-ceph get service rook-ceph-rgw-my-store
```

Test connectivity with a basic HTTP request:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  curl -s http://rook-ceph-rgw-my-store.rook-ceph.svc:80/
```

You should receive an RGW response (typically an empty XML document or an `AccessDenied` error for unauthenticated requests, both indicating the gateway is live).

## Updating Gateway Settings

Change instance count without downtime by editing the CRD:

```bash
kubectl -n rook-ceph patch cephobjectstore my-store \
  --type merge -p '{"spec":{"gateway":{"instances":3}}}'
```

Rook will reconcile and add the new pod while keeping existing ones running.

## Summary

RGW gateway settings in Rook control the HTTP/HTTPS ports, number of instances, and resource allocation for the S3-compatible object store endpoint. Use `port` for HTTP and `securePort` plus `sslCertificateRef` for HTTPS. Set `instances` to 2 or more for production availability. Tune resource requests and limits based on expected load. Changes to instance count are applied without downtime through Rook's reconciliation loop.
