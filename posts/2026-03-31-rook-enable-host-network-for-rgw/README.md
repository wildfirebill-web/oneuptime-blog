# How to Enable Host Network for RGW in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Networking, Object Storage

Description: Enable host networking for the Ceph RGW daemon in Rook to reduce latency and improve throughput for S3-compatible workloads.

---

## What Is Host Networking for RGW

By default, Rook runs the RADOS Gateway (RGW) pods using the Kubernetes pod network. This introduces a layer of NAT and overlay networking overhead. When you enable host networking for RGW, each pod binds directly to the node's network interface, making the gateway reachable on the host IP at the configured port.

This is useful when:
- You want to expose the S3 endpoint directly without a LoadBalancer or Ingress.
- You need the lowest possible network latency for object storage operations.
- You are running Rook on bare-metal nodes where pod network overhead is noticeable.

## Prerequisites

- A running Rook-Ceph cluster (v1.10+).
- Nodes that do not already have a process listening on the RGW port (default 80).
- Appropriate privileges to modify the `CephObjectStore` CRD.

## Configuring Host Network in the CephObjectStore

Edit your `CephObjectStore` manifest and add the `hostNetwork` field under the `gateway` section:

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
    instances: 3
    hostNetwork: true
```

Apply the manifest:

```bash
kubectl apply -f object-store.yaml
```

Rook will restart the RGW pods. Once they are running, each pod will use the host network namespace.

## Verifying the Configuration

Check that the RGW pods show `hostNetwork: true` and that they are bound to the node IP:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-rgw -o wide
```

Each pod should show the node IP in the IP column, not a pod-network CIDR address.

You can also inspect the pod spec:

```bash
kubectl -n rook-ceph get pod <rgw-pod-name> -o jsonpath='{.spec.hostNetwork}'
```

Expected output:

```bash
true
```

## Accessing the RGW Endpoint

With host networking enabled, clients can reach the gateway directly on the node IP and port. For example, if the node IP is `192.168.1.10` and the port is `80`, the S3 endpoint is:

```bash
http://192.168.1.10:80
```

You can create a Kubernetes Service of type `ExternalName` or simply configure your application to use the node IP list.

## Considerations and Trade-offs

- **Port conflicts**: Ensure nothing else is listening on port 80 (or your chosen port) on each node that hosts an RGW pod.
- **Scheduling**: Use `affinity` or `tolerations` in the `CephObjectStore` spec to control which nodes RGW pods land on.
- **TLS**: For HTTPS, set `securePort` and provide a `sslCertificateRef` alongside `hostNetwork: true`.

Disabling host network reverts to pod-network mode. Update the field to `false` and re-apply to do so.

## Summary

Enabling host networking for Ceph RGW in Rook is a single-field change in the `CephObjectStore` manifest. It removes overlay network overhead and makes the S3 endpoint directly accessible on the node IP. Verify the change by confirming pods use node IPs and test access from a client before rolling this to production.
