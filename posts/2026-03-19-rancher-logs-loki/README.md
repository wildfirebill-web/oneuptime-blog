# How to Send Logs to Loki from Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Logging, Loki

Description: Configure Rancher to send Kubernetes logs to Grafana Loki for cost-effective log aggregation integrated with Grafana.

Grafana Loki is a horizontally scalable log aggregation system designed to be cost-effective and easy to operate. Unlike Elasticsearch, Loki indexes only metadata (labels) rather than the full text of log lines, making it significantly more resource-efficient. Combined with Grafana, it provides a powerful logging solution. This guide covers sending Kubernetes logs from Rancher to Loki.

## Prerequisites

- Rancher v2.6 or later with the Logging chart installed.
- A Loki instance (self-hosted or Grafana Cloud).
- Cluster admin permissions.

## Step 1: Deploy Loki in the Cluster

If you do not have Loki running, install it via Helm:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki \
  --namespace logging \
  --create-namespace \
  --set loki.auth_enabled=false \
  --set loki.storage.type=filesystem \
  --set singleBinary.replicas=1 \
  --set singleBinary.resources.requests.cpu=200m \
  --set singleBinary.resources.requests.memory=256Mi
```

For production, use the distributed mode with object storage:

```bash
helm install loki grafana/loki \
  --namespace logging \
  --create-namespace \
  --set loki.storage.type=s3 \
  --set loki.storage.s3.endpoint=s3.amazonaws.com \
  --set loki.storage.s3.region=us-east-1 \
  --set loki.storage.s3.bucketnames=my-loki-bucket \
  --set loki.storage.s3.access_key_id=AKIAIOSFODNN7EXAMPLE \
  --set loki.storage.s3.secret_access_key=wJalrXUtnFEMI/K7MDENG
```

## Step 2: Create a ClusterOutput for Loki

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: loki-output
  namespace: cattle-logging-system
spec:
  loki:
    url: http://loki.logging.svc.cluster.local:3100
    labels:
      cluster: production
    extract_kubernetes_labels: true
    remove_keys:
      - logtag
      - kubernetes
    configure_kubernetes_labels: false
    buffer:
      type: file
      path: /buffers/loki
      chunk_limit_size: 8MB
      total_limit_size: 2GB
      flush_interval: 5s
      flush_thread_count: 2
      retry_max_interval: 30
      retry_forever: true
```

## Step 3: Create a ClusterFlow

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: all-to-loki
  namespace: cattle-logging-system
spec:
  filters:
    - parser:
        parse:
          type: json
        key_name: log
        reserve_data: true
        remove_key_name_field: true
        suppress_parse_error_log: true

  globalOutputRefs:
    - loki-output
```

## Step 4: Configure Labels for Loki

Loki uses labels for indexing, so proper label configuration is crucial for query performance:

```yaml
spec:
  loki:
    url: http://loki.logging.svc.cluster.local:3100
    labels:
      cluster: production
      job: kubernetes
    extra_labels:
      environment: production
    extract_kubernetes_labels: true
    drop_single_key: true
```

Important label guidelines:
- Keep the number of unique label combinations (cardinality) low.
- Good labels: namespace, pod name, container name, app label.
- Avoid high-cardinality labels: request IDs, timestamps, user IDs.

## Step 5: Configure for Grafana Cloud Loki

For Grafana Cloud:

```bash
kubectl create secret generic grafana-cloud-secret \
  --namespace cattle-logging-system \
  --from-literal=username='your-grafana-cloud-user-id' \
  --from-literal=password='your-grafana-cloud-api-key'
```

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: grafana-cloud-loki
  namespace: cattle-logging-system
spec:
  loki:
    url: https://logs-prod-us-central1.grafana.net
    username:
      valueFrom:
        secretKeyRef:
          name: grafana-cloud-secret
          key: username
    password:
      valueFrom:
        secretKeyRef:
          name: grafana-cloud-secret
          key: password
    labels:
      cluster: production
    extract_kubernetes_labels: true
    buffer:
      type: file
      path: /buffers/grafana-cloud
      chunk_limit_size: 8MB
      flush_interval: 10s
      retry_forever: true
```

## Step 6: Configure Loki as a Grafana Data Source

Add Loki as a data source in Grafana:

1. Open Grafana from **Monitoring > Grafana**.
2. Go to **Configuration > Data Sources**.
3. Click **Add data source**.
4. Select **Loki**.
5. Set the URL to `http://loki.logging.svc.cluster.local:3100`.
6. Click **Save & Test**.

Or configure it through the monitoring chart:

```yaml
grafana:
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki.logging.svc.cluster.local:3100
      access: proxy
      isDefault: false
```

## Step 7: Query Logs in Grafana

Use LogQL to query logs in Grafana's Explore view:

```logql
# All logs from a namespace

{namespace="production"}

# Filter by pod name
{namespace="production", pod=~"api-server.*"}

# Search for error messages
{namespace="production"} |= "ERROR"

# Parse JSON logs and filter
{namespace="production"} | json | level="error"

# Count errors over time
count_over_time({namespace="production"} |= "ERROR" [5m])

# Top error-producing pods
topk(10, count_over_time({namespace="production"} |= "ERROR" [1h]) by (pod))
```

## Step 8: Set Up Log-Based Alerts in Grafana

Create alerts based on log patterns:

1. In Grafana, go to **Alerting > Alert rules**.
2. Click **New alert rule**.
3. Select Loki as the data source.
4. Enter a LogQL query:

```logql
count_over_time({namespace="production"} |= "ERROR" [5m]) > 50
```

5. Set the evaluation interval and conditions.
6. Configure notification channels.

## Step 9: Configure Loki Retention

Configure retention in Loki's configuration:

```yaml
# Loki config
limits_config:
  retention_period: 720h  # 30 days

compactor:
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150
```

## Step 10: Verify Log Delivery

Check Fluentd logs:

```bash
kubectl logs -n cattle-logging-system -l app.kubernetes.io/name=fluentd -c fluentd | grep -i loki
```

Query Loki directly:

```bash
kubectl port-forward -n logging svc/loki 3100:3100

curl -s "http://localhost:3100/loki/api/v1/query?query={cluster=\"production\"}&limit=5" | jq '.data.result'
```

Check Loki health:

```bash
curl -s http://localhost:3100/ready
curl -s http://localhost:3100/metrics | grep loki_ingester_streams_created_total
```

## Troubleshooting

- **Connection refused**: Verify Loki URL and port. Check if Loki pod is running.
- **Rate limiting**: Loki may reject logs if ingestion rate exceeds limits. Adjust `ingestion_rate_mb` in Loki config.
- **High cardinality errors**: Reduce the number of unique label combinations.
- **Query timeout**: Reduce the query time range or add more specific label filters.
- **No logs appearing**: Check that labels match your query. Verify the ClusterFlow is routing correctly.

## Summary

Sending logs to Loki from Rancher provides a cost-effective logging solution that integrates seamlessly with Grafana. Configure a ClusterOutput with Loki connection details, manage labels carefully to keep cardinality low, and use LogQL in Grafana for powerful log querying. Loki's label-based indexing makes it significantly cheaper to operate than full-text indexing solutions for most Kubernetes logging use cases.
