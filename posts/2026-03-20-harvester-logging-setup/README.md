# How to Configure Harvester Logging

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Logging, Kubernetes, Fluentbit, Elasticsearch, Loki, SUSE Rancher

Description: Learn how to enable and configure Harvester's centralized logging system to collect VM audit logs, system logs, and Kubernetes component logs for compliance and troubleshooting.

---

Harvester's logging system collects logs from the underlying Kubernetes components, VM audit events, and Harvester system components. Logs can be shipped to Elasticsearch, Loki, Splunk, or other backends.

---

## Step 1: Enable Logging in Harvester

In the Harvester UI:

1. Go to **Advanced > Monitoring & Logging > Logging**
2. Click **Enable Logging**

Or via Helm:

```bash
helm install rancher-logging \
  rancher-charts/rancher-logging \
  --namespace cattle-logging-system \
  --create-namespace
```

---

## Step 2: Configure a Log Output to Elasticsearch

```yaml
# clusteroutput-elasticsearch.yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: harvester-elasticsearch
  namespace: cattle-logging-system
spec:
  elasticsearch:
    host: elasticsearch.logging.example.com
    port: 9200
    scheme: https
    index_name: harvester-logs
    user: elastic
    password:
      valueFrom:
        secretKeyRef:
          name: elastic-creds
          key: password
    buffer:
      flush_interval: 30s
      chunk_limit_size: 8MB
```

---

## Step 3: Configure a Flow for Harvester Logs

```yaml
# clusterflow-harvester.yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: harvester-system-logs
  namespace: cattle-logging-system
spec:
  filters:
    - record_transformer:
        records:
          harvester_cluster: "harvester-prod"
          environment: "production"
  match:
    - select:
        namespaces:
          - harvester-system
          - longhorn-system
          - kube-system
  globalOutputRefs:
    - harvester-elasticsearch
```

---

## Step 4: Collect VM Audit Logs

Enable VM audit log collection via Harvester's event API:

```yaml
# Collect VM lifecycle events (create, delete, migrate)
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: vm-audit-logs
  namespace: cattle-logging-system
spec:
  filters:
    - grep:
        regexp:
          - key: $.kubernetes.namespace_name
            pattern: default
  match:
    - select:
        labels:
          app.kubernetes.io/part-of: harvester
  globalOutputRefs:
    - harvester-elasticsearch
```

---

## Step 5: Configure Log Rotation

```yaml
# Set log retention in Longhorn-backed Elasticsearch
# Via Elasticsearch ILM policy
PUT _ilm/policy/harvester-logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "10gb",
            "max_age": "7d"
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## Option: Ship to Loki Instead of Elasticsearch

```yaml
# clusteroutput-loki.yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: harvester-loki
  namespace: cattle-logging-system
spec:
  loki:
    url: http://loki.monitoring.svc.cluster.local:3100
    labels:
      cluster: harvester-prod
    buffer:
      flush_interval: 10s
```

---

## Best Practices

- Enable VM audit logging for compliance — track who created, deleted, or migrated VMs.
- Ship logs to a system outside the Harvester cluster for disaster recovery coverage.
- Add the `harvester_cluster` label to all log records for easy filtering when aggregating multiple Harvester clusters.
- Set log retention based on your compliance requirements — many regulations require 90-365 days.
