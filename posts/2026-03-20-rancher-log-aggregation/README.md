# How to Configure Log Aggregation Pipelines in Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Log Aggregation, Loki, Fluentd, Observability

Description: Build scalable log aggregation pipelines in Rancher using Fluentd, Fluent Bit, or Vector to collect, process, and forward logs to centralized storage.

## Introduction

Log aggregation centralizes logs from all your Kubernetes workloads for searching, alerting, and compliance. This guide covers building production log pipelines using Fluent Bit (lightweight collection), Fluentd (processing and routing), and the Rancher Logging operator for declarative pipeline management.

## Prerequisites

- Rancher-managed Kubernetes cluster
- Helm 3.x installed
- Loki, Elasticsearch, or another log storage backend
- kubectl access

## Step 1: Install the Rancher Logging Operator

```bash
# Install from Rancher Apps catalog or via Helm

helm repo add rancher-charts https://charts.rancher.io

helm install rancher-logging rancher-charts/rancher-logging \
  --namespace cattle-logging-system \
  --create-namespace \
  --wait
```

## Step 2: Configure ClusterFlow and ClusterOutput

```yaml
# cluster-output-loki.yaml - Send all cluster logs to Loki
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: loki-output
  namespace: cattle-logging-system
spec:
  loki:
    url: http://loki.observability.svc.cluster.local:3100
    configure_kubernetes_labels: true
    # Buffer configuration
    buffer:
      type: file
      path: /buffers/loki
      flush_mode: interval
      flush_interval: 10s
      retry_max_times: 5
    # Add cluster-level labels to all logs
    labels:
      cluster: rancher-production
      environment: production
---
# cluster-flow-all.yaml - Route all logs to Loki
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: all-logs-to-loki
  namespace: cattle-logging-system
spec:
  filters:
    # Parse JSON logs
    - parser:
        remove_key_name_field: true
        parse:
          type: json
    # Add Kubernetes metadata
    - kubernetes_metadata:
        skip_labels: true
  globalOutputRefs:
    - loki-output
```

## Step 3: Create Namespace-Specific Flows

```yaml
# production-flow.yaml - Production namespace log routing
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: production-logs
  namespace: production
spec:
  filters:
    # Extract important fields from structured logs
    - parser:
        key_name: message
        remove_key_name_field: false
        parse:
          type: json
          time_key: timestamp
          time_format: "%Y-%m-%dT%H:%M:%S.%N%Z"
    # Add application labels
    - record_transformer:
        records:
          app_version: "1.0.0"
          team: "platform"
    # Drop debug logs in production
    - grep:
        exclude:
          - key: level
            pattern: /^DEBUG$/
  # Route critical errors to dedicated output
  match:
    - select:
        labels:
          tier: critical
      outputRefs:
        - critical-logs-output
    - select: {}  # Default: send all other logs to Loki
      outputRefs:
        - loki-output
---
# critical-output.yaml - High-priority log output
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
  name: critical-logs-output
  namespace: production
spec:
  elasticsearch:
    host: elasticsearch.observability.svc.cluster.local
    port: 9200
    index_name: critical-logs
    type_name: log
    buffer:
      type: file
      path: /buffers/critical
      flush_mode: interval
      flush_interval: 5s
```

## Step 4: Deploy Fluent Bit as DaemonSet (Alternative)

```yaml
# fluent-bit-values.yaml - Fluent Bit configuration
config:
  service: |
    [SERVICE]
      Flush         5
      Log_Level     info
      Daemon        off
      HTTP_Server   On
      HTTP_Listen   0.0.0.0
      HTTP_Port     2020

  inputs: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            cri
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

  filters: |
    # Kubernetes metadata enrichment
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    # Parse JSON log messages
    [FILTER]
        Name    parser
        Match   kube.*
        Key_Name log
        Parser  json
        Reserve_Data On

    # Add cluster label
    [FILTER]
        Name    record_modifier
        Match   kube.*
        Record  cluster rancher-production

  outputs: |
    # Send to Loki
    [OUTPUT]
        Name             loki
        Match            *
        Host             loki.observability.svc.cluster.local
        Port             3100
        Labels           job=fluentbit,cluster=rancher-production
        Label_Keys       $kubernetes['namespace_name'],$kubernetes['pod_name'],$kubernetes['container_name'],$level
        Line_Format      json
        Retry_Limit      5

    # Send errors to Elasticsearch
    [OUTPUT]
        Name            es
        Match           *error*
        Host            elasticsearch.observability.svc.cluster.local
        Port            9200
        Index           k8s-errors
        Type            log
        Retry_Limit     False

  customParsers: |
    [PARSER]
        Name        json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%LZ

    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

```bash
# Install Fluent Bit
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit \
  --namespace cattle-logging-system \
  --create-namespace \
  --values fluent-bit-values.yaml \
  --wait
```

## Step 5: Configure Log Rotation and Retention

```yaml
# log-retention-configmap.yaml - Log retention configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-retention-config
  namespace: observability
data:
  # Set per-namespace retention via Loki tenant configuration
  retention: |
    # Production logs: 30 days
    - selector: '{namespace="production"}'
      period: 720h
    # Development logs: 7 days
    - selector: '{namespace="development"}'
      period: 168h
    # Default: 14 days
    - selector: '{}'
      period: 336h
```

## Step 6: Test the Log Pipeline

```bash
# Generate test logs
kubectl run test-logger --image=busybox \
  --restart=Never \
  -n production \
  -- sh -c 'for i in $(seq 1 100); do echo "{\"level\":\"info\",\"msg\":\"test message $i\",\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}"; done'

# Wait for logs to be collected
sleep 30

# Query Loki for the test logs
kubectl port-forward -n observability svc/loki 3100:3100 &
curl 'http://localhost:3100/loki/api/v1/query?query={namespace="production",pod=~"test-logger.*"}&limit=10' | jq '.data.result[0].values'
```

## Conclusion

Log aggregation pipelines in Rancher provide centralized, searchable log storage across all your workloads. The Rancher Logging operator provides the most elegant Kubernetes-native approach with CRD-based pipeline configuration. For high-volume environments, use Fluent Bit (lightweight) for collection and Fluentd for complex routing and transformation. Always configure buffering and retry logic to handle backend outages, and set appropriate retention policies to control storage costs.
