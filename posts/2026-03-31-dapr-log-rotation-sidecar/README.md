# How to Configure Log Rotation for Dapr Sidecar

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Logging, Log Rotation, Kubernetes, Operation

Description: Manage Dapr sidecar log file growth on Kubernetes nodes using container runtime log rotation settings and node-level configuration.

---

In Kubernetes, container logs are written to files on the node filesystem. Without log rotation, these files grow unbounded and can fill node disk space. This guide covers configuring log rotation for Dapr sidecar containers at the container runtime and node levels.

## How Kubernetes Manages Container Logs

Kubernetes uses the container runtime (containerd or Docker) to write logs to files on the node at `/var/log/pods/`. The kubelet is responsible for log rotation at the pod level.

Dapr sidecar logs are rotated the same way as any other container - through kubelet configuration.

## Configuring Kubelet Log Rotation

The kubelet handles log rotation for all containers including daprd. Configure it in the kubelet config file:

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
containerLogMaxSize: "50Mi"
containerLogMaxFiles: 5
```

Apply the config and restart the kubelet:

```bash
sudo systemctl restart kubelet
```

This limits each container log file to 50 MiB and keeps up to 5 rotated files per container.

## Setting Log Rotation in Managed Kubernetes

### AKS

```bash
# Set via AKS node pool configuration
az aks nodepool update \
  --resource-group myRG \
  --cluster-name myCluster \
  --name agentpool \
  --kubelet-config kubelet-config.json
```

With `kubelet-config.json`:

```json
{
  "containerLogMaxSizeMB": 50,
  "containerLogMaxFiles": 5
}
```

### EKS

Use a custom launch template with kubelet arguments:

```bash
# In the user data script
/etc/eks/bootstrap.sh my-cluster --kubelet-extra-args \
  '--container-log-max-size=50Mi --container-log-max-files=5'
```

## Reducing Log Volume at the Source

The most effective way to manage Dapr sidecar log size is to reduce what gets logged:

```yaml
# Production settings - reduce log volume
metadata:
  annotations:
    dapr.io/log-level: "warn"
    dapr.io/enable-api-logging: "false"
    dapr.io/log-as-json: "true"
```

At `warn` level with API logging disabled, a typical Dapr sidecar produces less than 1 MB per day under normal operation.

## Monitoring Node Disk Usage

Monitor disk usage on Kubernetes nodes to catch issues before they occur:

```bash
# Check node disk usage
kubectl describe node worker-node-1 | grep -A5 "Allocated resources"

# Check log directory size
du -sh /var/log/pods/default_order-service*/daprd/
```

Prometheus alert for node disk usage:

```yaml
- alert: NodeDiskPressure
  expr: |
    node_filesystem_avail_bytes{mountpoint="/", fstype!="tmpfs"}
      / node_filesystem_size_bytes{mountpoint="/"} < 0.15
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Node {{ $labels.node }} disk space below 15%"
```

## Centralizing Logs Before Rotation

The best practice is to ship logs to a centralized backend (Elasticsearch, Loki, CloudWatch) before rotation discards them. Use FluentD or Fluent Bit as a DaemonSet:

```yaml
# Fluent Bit container log tail config
[INPUT]
    Name              tail
    Path              /var/log/containers/*daprd*.log
    multiline.parser  docker, cri
    Tag               kube.*
    Mem_Buf_Limit     5MB
    Skip_Long_Lines   On
```

## Summary

Dapr sidecar log rotation in Kubernetes is managed by kubelet's `containerLogMaxSize` and `containerLogMaxFiles` settings, not by Dapr itself. Set these limits to prevent disk pressure on nodes. Reduce log volume at the source by using `warn` log level in production. Always ship logs to a centralized backend before they are rotated away so you retain history for incident investigation.
