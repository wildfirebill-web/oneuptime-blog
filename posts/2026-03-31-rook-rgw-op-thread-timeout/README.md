# How to Set rgw_op_thread_timeout and rgw_op_thread_suicide_timeout

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Configuration, Timeout, Object Storage

Description: Configure rgw_op_thread_timeout and rgw_op_thread_suicide_timeout in Ceph RGW to control operation thread lifecycle and prevent hung threads.

---

Ceph RGW uses a thread pool to process S3 and Swift operations. Two timeout parameters control how long threads are allowed to run before being considered stuck and forcibly terminated.

## Understanding the Two Timeouts

- **`rgw_op_thread_timeout`** - Soft timeout (seconds). After this interval, the operation is flagged as timed out but the thread continues.
- **`rgw_op_thread_suicide_timeout`** - Hard timeout (seconds). After this interval, the RGW process kills itself to prevent indefinite hangs. Set to 0 to disable.

```bash
# View current settings
ceph config get client.rgw rgw_op_thread_timeout
ceph config get client.rgw rgw_op_thread_suicide_timeout
```

## Configuring the Timeouts

```bash
# Soft timeout - warn after 180 seconds (default is 600)
ceph config set client.rgw rgw_op_thread_timeout 180

# Hard suicide timeout - kill process after 240 seconds
ceph config set client.rgw rgw_op_thread_suicide_timeout 240

# Disable the suicide timeout (not recommended in production)
ceph config set client.rgw rgw_op_thread_suicide_timeout 0
```

The suicide timeout should always be greater than the soft timeout to give the system time to log the issue.

## When to Tune These Values

**Lower timeouts** are appropriate when:
- Your operations are expected to complete quickly (small object workloads)
- You want aggressive detection of stuck threads

**Higher timeouts** are appropriate when:
- Bulk upload or large object operations are common
- Network conditions are variable or latency is high

## Applying in Rook

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_op_thread_timeout = 300
    rgw_op_thread_suicide_timeout = 600
```

Apply the ConfigMap and restart:

```bash
kubectl apply -f rook-config-override.yaml
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Diagnosing Thread Timeout Issues

When a thread timeout triggers, check the RGW logs:

```bash
kubectl -n rook-ceph logs deploy/rook-ceph-rgw-my-store-a --tail=200 | grep -i "timeout\|suicide\|thread"
```

Look for lines like:

```bash
# Soft timeout hit
rgw: op thread 0x7f... timed out

# Suicide timeout triggered
rgw: rgw_op_thread_suicide_timeout exceeded, killing process
```

## Relationship with RADOS Timeouts

These timeouts interact with RADOS-level timeouts. If RADOS operations are slow (e.g., during OSD recovery), RGW threads will be held waiting. Ensure RADOS timeouts are lower than RGW thread timeouts:

```bash
ceph config set client.rgw rados_osd_op_timeout 120
```

## Summary

`rgw_op_thread_timeout` controls when an RGW operation thread is flagged as hung, while `rgw_op_thread_suicide_timeout` controls when the entire RGW process restarts. Set the soft timeout based on your expected operation duration and make the suicide timeout 1.5-2x the soft timeout. In Kubernetes/Rook deployments, RGW pod restarts are handled gracefully.
