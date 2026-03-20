# How to Configure Harvester Logging

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Logging, Fluentd, Fluentbit, Elasticsearch, Kubernetes, Observability, SUSE Rancher

Description: Learn how to configure centralized log collection in Harvester using the built-in logging stack, route logs to external destinations like Elasticsearch or Splunk, and create log filters for specific workloads.

---

Harvester includes a built-in logging stack based on Rancher Logging (Banzai Cloud Logging Operator). It collects logs from all nodes, VMs, and Kubernetes workloads and routes them to configurable outputs.

---

## Step 1: Enable Harvester Logging

```bash
# Check if logging is enabled
kubectl get pods -n cattle-logging-system

# Enable via Harvester UI:
# Advanced → Logging → Enable Logging
```

---

## Step 2: Deploy a Log Output to Elasticsearch

```yaml
# elasticsearch-output.yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: elasticsearch-output
  namespace: cattle-logging-system
spec:
  elasticsearch:
    host: elasticsearch.logging.svc.cluster.local
    port: 9200
    scheme: http
    index_name: harvester-logs
    type_name: fluentd
    buffer:
      timekey: 1m
      timekey_wait: 30s
      flush_interval: 10s
      retry_max_times: 3
```

```bash
kubectl apply -f elasticsearch-output.yaml
```

---

## Step 3: Create a ClusterFlow to Collect All Logs

```yaml
# cluster-flow.yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: all-logs
  namespace: cattle-logging-system
spec:
  match:
    - select: {}    # Match all logs

  filters:
    # Add cluster name to all logs
    - record_transformer:
        records:
          - cluster_name: harvester-production
            environment: production

    # Parse JSON logs if containers emit JSON
    - parser:
        remove_key_name_field: true
        parse:
          type: json

  globalOutputRefs:
    - elasticsearch-output
```

---

## Step 4: Filter Logs for Specific Namespaces

```yaml
# namespace-flow.yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: production-logs
  namespace: production
spec:
  match:
    - select:
        labels:
          app: my-app
    - exclude:
        labels:
          logging: disabled

  filters:
    - grep:
        regexp:
          - key: log
            pattern: /ERROR|WARN/

  localOutputRefs:
    - elasticsearch-output
```

---

## Step 5: Route Logs to Multiple Destinations

```yaml
# splunk-output.yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: splunk-output
  namespace: cattle-logging-system
spec:
  splunkHec:
    hec_host: splunk.example.com
    hec_port: 8088
    insecure_ssl: false
    hec_token:
      valueFrom:
        secretKeyRef:
          name: splunk-hec-token
          key: token
    index: harvester
```

```yaml
# multi-output-flow.yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: security-logs
  namespace: cattle-logging-system
spec:
  match:
    - select:
        namespaces:
          - kube-system
          - harvester-system

  globalOutputRefs:
    - elasticsearch-output
    - splunk-output    # Send to both destinations
```

---

## Step 6: Collect Node System Logs

Harvester also supports collecting host-level system logs (from `/var/log/` or journald):

```yaml
# host-log-flow.yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: host-systemd-logs
  namespace: cattle-logging-system
spec:
  match:
    - select:
        # Match systemd journal logs from nodes
        labels:
          component: systemd

  globalOutputRefs:
    - elasticsearch-output
```

---

## Step 7: Verify Log Collection

```bash
# Check that log collector pods are running
kubectl get pods -n cattle-logging-system

# View Fluentbit logs (log collector)
kubectl logs -n cattle-logging-system \
  $(kubectl get pod -n cattle-logging-system \
    -l app.kubernetes.io/name=fluentbit -o name | head -1) \
  | tail -50

# Check for output errors
kubectl logs -n cattle-logging-system \
  $(kubectl get pod -n cattle-logging-system \
    -l app.kubernetes.io/name=fluentd -o name | head -1) \
  | grep -i error
```

---

## Step 8: Adjust Buffer Settings for High-Volume Logging

```yaml
# Tune buffer settings for high log volume
spec:
  elasticsearch:
    buffer:
      timekey: 5m
      timekey_wait: 1m
      flush_interval: 30s
      flush_thread_count: 4
      chunk_limit_size: 8m
      total_limit_size: 512m
      retry_max_times: 10
      retry_exponential_backoff_base: 2
      retry_wait: 30s
```

---

## Best Practices

- Use `ClusterOutput` for sending logs to centralized systems (Elasticsearch, Splunk) and `Output` (namespace-scoped) for per-team log destinations.
- Add `record_transformer` to include cluster name, environment, and region in every log record — this is essential when aggregating logs from multiple Harvester clusters.
- Monitor Fluentd buffer size and tail it in Prometheus — a growing buffer indicates the output destination is too slow and logs may be dropped.
