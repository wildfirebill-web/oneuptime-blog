# How to Monitor GPU Usage in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, GPU, Monitoring, Prometheus, Grafana, Dcgm

Description: Guide to setting up comprehensive GPU monitoring in Rancher using DCGM Exporter, Prometheus, and Grafana dashboards.

## Introduction

Monitoring GPU usage is critical for optimizing utilization, diagnosing performance issues, and planning capacity. Rancher combined with NVIDIA DCGM Exporter and Prometheus provides rich GPU observability.

## Key GPU Metrics to Monitor

- **GPU Utilization**: Percentage of time GPU was active
- **Memory Utilization**: GPU memory used vs available
- **Temperature**: GPU core and memory temperature
- **Power Usage**: Watt consumption
- **Clock Speeds**: GPU and memory clock frequencies
- **Error Counts**: ECC errors, NVLink errors

## Step 1: Enable DCGM Exporter

```yaml
# dcgm-exporter-values.yaml (part of GPU Operator)

dcgmExporter:
  enabled: true
  version: 3.2.5-3.1.8-ubuntu20.04
  
  # Metrics configuration
  config:
    name: dcgm-metrics-config
  
  serviceMonitor:
    enabled: true              # Auto-discover by Prometheus Operator
    interval: 30s
    honorLabels: false
    additionalLabels:
      release: prometheus      # Match your Prometheus stack label
```

## Step 2: Configure DCGM Metrics

```yaml
# dcgm-metrics-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dcgm-metrics-config
  namespace: gpu-operator
data:
  dcgm-metrics.csv: |
    # GPU Utilization
    DCGM_FI_DEV_GPU_UTIL,     gauge, GPU utilization (%).
    DCGM_FI_DEV_MEM_COPY_UTIL, gauge, Memory utilization (%).
    
    # Memory
    DCGM_FI_DEV_FB_FREE,      gauge, Framebuffer memory free (MiB).
    DCGM_FI_DEV_FB_USED,      gauge, Framebuffer memory used (MiB).
    
    # Temperature
    DCGM_FI_DEV_GPU_TEMP,     gauge, GPU temperature (C).
    DCGM_FI_DEV_MEMORY_TEMP,  gauge, Memory temperature (C).
    
    # Power
    DCGM_FI_DEV_POWER_USAGE,  gauge, Power draw (W).
    DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION, counter, Total energy (mJ).
    
    # Clocks
    DCGM_FI_DEV_SM_CLOCK,     gauge, SM clock frequency (MHz).
    DCGM_FI_DEV_MEM_CLOCK,    gauge, Memory clock frequency (MHz).
    
    # Errors
    DCGM_FI_DEV_ECC_DBE_VOL_TOTAL, counter, Total double-bit ECC errors.
```

## Step 3: Install Prometheus Stack

```bash
# Install kube-prometheus-stack via Rancher Apps
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack   --namespace cattle-monitoring-system   --create-namespace   --set grafana.enabled=true   --set prometheusOperator.enabled=true   --set defaultRules.rules.general=true
```

## Step 4: Create GPU Grafana Dashboard

```bash
# Import NVIDIA DCGM Exporter Dashboard (Grafana ID: 12239)
kubectl exec -n cattle-monitoring-system   $(kubectl get pods -n cattle-monitoring-system -l app.kubernetes.io/name=grafana -o name)   -- grafana-cli plugins install grafana-piechart-panel

# Or use Grafana provisioning
cat > /tmp/gpu-dashboard-provisioning.yaml << 'DASHEOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: gpu-dashboards
  namespace: cattle-monitoring-system
  labels:
    grafana_dashboard: "1"    # Grafana sidecar will pick this up
data:
  gpu-overview.json: |
    {
      "__inputs": [],
      "title": "NVIDIA GPU Overview",
      "uid": "nvidia-gpu-overview",
      "panels": [
        {
          "title": "GPU Utilization %",
          "type": "graph",
          "targets": [{"expr": "DCGM_FI_DEV_GPU_UTIL"}]
        },
        {
          "title": "GPU Memory Used (MiB)",
          "type": "graph",
          "targets": [{"expr": "DCGM_FI_DEV_FB_USED"}]
        },
        {
          "title": "GPU Temperature (C)",
          "type": "graph",
          "targets": [{"expr": "DCGM_FI_DEV_GPU_TEMP"}]
        }
      ]
    }
DASHEOF
kubectl apply -f /tmp/gpu-dashboard-provisioning.yaml
```

## Step 5: Configure GPU Alerts

```yaml
# gpu-prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: gpu.rules
    rules:
    - alert: GPUHighTemperature
      expr: DCGM_FI_DEV_GPU_TEMP > 85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "GPU temperature high on {{ $labels.instance }}"
        description: "GPU temp is {{ $value }}C (threshold: 85C)"
    
    - alert: GPUMemoryAlmostFull
      expr: |
        (DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)) > 0.95
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "GPU memory nearly exhausted"
        description: "GPU memory usage is {{ $value | humanizePercentage }}"
    
    - alert: GPUECCError
      expr: increase(DCGM_FI_DEV_ECC_DBE_VOL_TOTAL[1h]) > 0
      labels:
        severity: critical
      annotations:
        summary: "GPU double-bit ECC error detected"
        description: "Hardware error on GPU {{ $labels.gpu }}"
    
    - alert: GPUUnderutilized
      expr: |
        avg_over_time(DCGM_FI_DEV_GPU_UTIL[1h]) < 20
        and DCGM_FI_DEV_GPU_UTIL > 0
      for: 2h
      labels:
        severity: info
      annotations:
        summary: "GPU is underutilized"
        description: "Average GPU utilization is {{ $value }}% over 1 hour"
```

## Step 6: Query GPU Metrics

```bash
# Get GPU utilization for all pods
kubectl exec -n cattle-monitoring-system prometheus-0   -- promtool query instant 'DCGM_FI_DEV_GPU_UTIL'

# Check top GPU consumers
kubectl exec -n cattle-monitoring-system prometheus-0   -- promtool query instant 'topk(5, DCGM_FI_DEV_FB_USED)'

# GPU utilization over time
kubectl exec -n cattle-monitoring-system prometheus-0   -- promtool query range     --start=1h     --end=0m     --step=5m     'avg(DCGM_FI_DEV_GPU_UTIL) by (instance)'
```

## Conclusion

Comprehensive GPU monitoring with DCGM Exporter, Prometheus, and Grafana provides the visibility needed to optimize GPU resource usage, catch hardware issues early, and plan capacity. Configure alerts for temperature, memory pressure, and ECC errors to maintain healthy GPU infrastructure.
