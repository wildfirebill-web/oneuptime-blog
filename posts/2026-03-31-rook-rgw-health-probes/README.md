# How to Set Up Health Probes for RGW in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Kubernetes, Observability

Description: Learn how to configure liveness and readiness health probes for Rook RGW gateway pods to enable proper traffic routing and automatic pod recovery.

---

## Why Health Probes Matter for RGW

Kubernetes uses liveness and readiness probes to determine whether a pod is healthy and ready to serve traffic:

- **Liveness probe**: Restarts a pod if it becomes unresponsive. RGW can enter a hung state if metadata operations back up, making liveness probes critical.
- **Readiness probe**: Removes a pod from the service endpoints when it is not ready. During startup or high load, RGW may not be able to serve S3 requests immediately.

Without health probes, a hung or starting RGW pod continues receiving traffic, causing request timeouts for clients.

## Default Probe Configuration

Rook configures basic health probes for RGW by default. You can customize these through the `CephObjectStore` CRD's `healthCheck` section:

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
  healthCheck:
    bucket:
      disabled: false
      interval: 60s
  gateway:
    port: 80
    instances: 2
```

The `healthCheck.bucket` setting controls how frequently Rook checks the bucket health of the object store endpoint.

## Customizing Readiness and Liveness Probes

Override the probe settings in the gateway spec:

```yaml
gateway:
  port: 80
  instances: 2
  livenessProbe:
    disabled: false
    probe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
      successThreshold: 1
  readinessProbe:
    disabled: false
    probe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1
```

## Understanding the RGW Health Endpoint

RGW responds to unauthenticated HTTP `GET /` requests with an XML response or a 403/405. Both indicate the daemon is alive. The probe checks for a non-500 HTTP status code:

```bash
# Test the RGW health endpoint manually
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  curl -s -o /dev/null -w "%{http_code}" \
  http://rook-ceph-rgw-my-store.rook-ceph.svc:80/
```

A response of `200`, `403`, or `405` means RGW is alive. `000` or `500` indicates a problem.

## Disabling Probes for Debugging

If you need to keep a misbehaving pod running for diagnostics:

```yaml
gateway:
  livenessProbe:
    disabled: true
  readinessProbe:
    disabled: true
```

Remember to re-enable probes after debugging.

## Monitoring Probe Results

Watch probe events on RGW pods:

```bash
kubectl -n rook-ceph describe pod -l app=rook-ceph-rgw | grep -A5 "Liveness\|Readiness"
```

If probes are failing repeatedly:

```bash
kubectl -n rook-ceph get events --field-selector reason=Unhealthy | grep rgw
```

## Checking Object Store Health via Rook

Rook also exposes object store health through the CRD status:

```bash
kubectl -n rook-ceph get cephobjectstore my-store -o jsonpath='{.status}'
```

The `.status.conditions` array shows if the bucket health check is passing.

## Summary

Health probes for RGW in Rook are configured through the `gateway.livenessProbe`, `gateway.readinessProbe`, and `healthCheck.bucket` fields in the CephObjectStore CRD. The liveness probe restarts hung RGW pods, while the readiness probe removes not-ready pods from service rotation. The RGW `GET /` endpoint returns a non-500 response when the daemon is alive, making it a reliable probe target. Tune `initialDelaySeconds` to allow RGW time to initialize before probes begin.
