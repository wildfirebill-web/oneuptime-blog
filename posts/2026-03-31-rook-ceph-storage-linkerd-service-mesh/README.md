# How to Use Ceph Storage with Linkerd Service Mesh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Linkerd, Service Mesh, Kubernetes, Storage, Networking

Description: Learn how to configure Rook Ceph storage to work correctly alongside Linkerd service mesh, including proxy exclusions and traffic configuration for CSI and RGW.

---

## Linkerd and Ceph Compatibility Considerations

Linkerd injects a lightweight Rust-based proxy into pods to handle service mesh features. While Linkerd is generally less intrusive than Istio, it still intercepts network traffic and can interfere with Ceph components in specific ways:

- Ceph's RADOS protocol is not HTTP-based and can behave unpredictably when proxied
- Linkerd proxies add latency to storage I/O
- CSI node plugins run as privileged daemonsets and should not have Linkerd injected

## Excluding the Rook Namespace from Linkerd Injection

Prevent Linkerd from injecting proxies into rook-ceph namespace pods:

```bash
kubectl annotate namespace rook-ceph linkerd.io/inject=disabled
```

Verify the annotation:

```bash
kubectl get namespace rook-ceph -o jsonpath='{.metadata.annotations}'
```

## Excluding Individual Pods

If Linkerd is already installed in the namespace, annotate specific pods to exclude them:

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    linkerd.io/inject: disabled
spec:
  containers:
  - name: ceph-osd
    image: rook/ceph:latest
```

For DaemonSets like CSI node plugins, add the annotation to the pod template:

```yaml
spec:
  template:
    metadata:
      annotations:
        linkerd.io/inject: disabled
```

## Using Linkerd for Application-to-RGW Traffic

While Ceph internals should bypass Linkerd, you can benefit from Linkerd features for application pods accessing Ceph RGW (S3):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  annotations:
    linkerd.io/inject: enabled
```

Applications in the `my-app` namespace will have Linkerd-managed connections to the RGW service, gaining features like:
- Automatic retries on failed requests
- Timeout enforcement
- Traffic metrics and tracing

## Configuring Service Profiles for RGW

Define a ServiceProfile to enable per-route metrics and retry policies for S3 access:

```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: rook-ceph-rgw-my-store.rook-ceph.svc.cluster.local
  namespace: my-app
spec:
  routes:
  - name: GET /
    condition:
      method: GET
      pathRegex: /.*
    isRetryable: true
    timeout: 30s
  - name: PUT /
    condition:
      method: PUT
      pathRegex: /.*
    isRetryable: false
    timeout: 120s
```

## Verifying Linkerd is Not Proxying Ceph Traffic

Check that rook-ceph pods do not have the Linkerd proxy container:

```bash
kubectl -n rook-ceph get pods -o jsonpath=\
  '{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
```

Pods should not show `linkerd-proxy` in their container list.

## Monitoring RGW Traffic Through Linkerd

For applications in the mesh accessing RGW:

```bash
# View live traffic stats
linkerd viz stat deployment -n my-app

# View route-level metrics for RGW calls
linkerd viz routes deployment/my-app -n my-app
```

## Summary

Linkerd and Ceph coexist well when you disable proxy injection for the rook-ceph namespace. Ceph's internal traffic bypasses the mesh entirely, avoiding protocol compatibility issues and latency overhead. Application pods in meshed namespaces can still access Ceph storage services, and you can leverage Linkerd's ServiceProfile feature to gain observability and retry policies specifically for application-to-RGW traffic.
