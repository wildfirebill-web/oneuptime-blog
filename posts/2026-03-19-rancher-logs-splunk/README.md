# How to Send Logs to Splunk from Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Logging, Splunk

Description: Configure Rancher to forward Kubernetes logs to Splunk using the HTTP Event Collector (HEC) integration.

Splunk is a widely used enterprise log management and analytics platform. Rancher can forward Kubernetes cluster logs to Splunk using the HTTP Event Collector (HEC), which provides a reliable and scalable method for log ingestion. This guide covers the complete setup from Splunk HEC configuration to Rancher ClusterOutput creation.

## Prerequisites

- Rancher v2.6 or later with the Logging chart installed.
- A Splunk Enterprise or Splunk Cloud instance.
- Admin access to Splunk for creating HEC tokens.
- Cluster admin permissions in Rancher.

## Step 1: Configure Splunk HTTP Event Collector

1. Log in to your Splunk instance.
2. Go to **Settings > Data Inputs > HTTP Event Collector**.
3. Click **New Token**.
4. Enter a name like "Kubernetes Logs" and click **Next**.
5. Select the target index (e.g., `kubernetes`) and source type (e.g., `_json`).
6. Review and click **Submit**.
7. Copy the generated HEC token.

Ensure the HEC is enabled globally:
1. Go to **Settings > Data Inputs > HTTP Event Collector**.
2. Click **Global Settings**.
3. Set **All Tokens** to **Enabled**.
4. Note the HTTP port number (default is 8088).

## Step 2: Create the HEC Token Secret

```bash
kubectl create secret generic splunk-hec-secret \
  --namespace cattle-logging-system \
  --from-literal=hec-token='your-splunk-hec-token'
```

## Step 3: Create a ClusterOutput for Splunk

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: splunk-output
  namespace: cattle-logging-system
spec:
  splunkHec:
    hec_host: splunk.example.com
    hec_port: 8088
    hec_token:
      valueFrom:
        secretKeyRef:
          name: splunk-hec-secret
          key: hec-token
    protocol: https
    insecure_ssl: false
    index: kubernetes
    source: kubernetes
    sourcetype: _json
    buffer:
      type: file
      path: /buffers/splunk
      chunk_limit_size: 8MB
      total_limit_size: 2GB
      flush_interval: 5s
      flush_thread_count: 2
      retry_max_interval: 60
      retry_forever: true
```

## Step 4: Create a ClusterFlow

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: all-to-splunk
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

    - record_transformer:
        records:
          - cluster_name: "production"
            environment: "prod"

  globalOutputRefs:
    - splunk-output
```

## Step 5: Route Logs to Different Splunk Indexes

Send different log types to different Splunk indexes:

```yaml
# Infrastructure logs

apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: splunk-infra
  namespace: cattle-logging-system
spec:
  splunkHec:
    hec_host: splunk.example.com
    hec_port: 8088
    hec_token:
      valueFrom:
        secretKeyRef:
          name: splunk-hec-secret
          key: hec-token
    protocol: https
    index: kubernetes-infra
    source: kubernetes-infra
    sourcetype: _json
    buffer:
      type: file
      path: /buffers/splunk-infra
      chunk_limit_size: 8MB
      flush_interval: 5s
      retry_forever: true
---
# Application logs
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: splunk-app
  namespace: cattle-logging-system
spec:
  splunkHec:
    hec_host: splunk.example.com
    hec_port: 8088
    hec_token:
      valueFrom:
        secretKeyRef:
          name: splunk-hec-secret
          key: hec-token
    protocol: https
    index: kubernetes-app
    source: kubernetes-app
    sourcetype: _json
    buffer:
      type: file
      path: /buffers/splunk-app
      chunk_limit_size: 8MB
      flush_interval: 5s
      retry_forever: true
---
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: infra-logs
  namespace: cattle-logging-system
spec:
  match:
    - select:
        namespaces:
          - kube-system
          - cattle-system
  globalOutputRefs:
    - splunk-infra
---
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: app-logs
  namespace: cattle-logging-system
spec:
  match:
    - exclude:
        namespaces:
          - kube-system
          - cattle-system
          - cattle-logging-system
          - cattle-monitoring-system
    - select: {}
  globalOutputRefs:
    - splunk-app
```

## Step 6: Configure Splunk Cloud

For Splunk Cloud, use the provided HEC endpoint:

```yaml
spec:
  splunkHec:
    hec_host: http-inputs-your-instance.splunkcloud.com
    hec_port: 443
    protocol: https
    insecure_ssl: false
    hec_token:
      valueFrom:
        secretKeyRef:
          name: splunk-hec-secret
          key: hec-token
```

## Step 7: Add Kubernetes Metadata to Splunk Events

Enrich log events with Kubernetes metadata for better search in Splunk:

```yaml
spec:
  filters:
    - record_transformer:
        enable_ruby: true
        records:
          - host: "${record.dig('kubernetes', 'host')}"
            source: "kubernetes:${record.dig('kubernetes', 'namespace_name')}/${record.dig('kubernetes', 'pod_name')}"
            sourcetype: "${record.dig('kubernetes', 'container_name')}"

    - record_transformer:
        records:
          - cluster: "production-us-east-1"
            environment: "production"
```

## Step 8: Configure Acknowledgment

For guaranteed delivery, enable HEC acknowledgment:

```yaml
spec:
  splunkHec:
    use_ack: true
    channel: "your-channel-id"
```

Note: HEC acknowledgment requires indexer acknowledgment to be enabled in Splunk.

## Step 9: Verify Log Delivery

Check Fluentd logs:

```bash
kubectl logs -n cattle-logging-system -l app.kubernetes.io/name=fluentd -c fluentd | grep -i splunk
```

Search in Splunk:

```plaintext
index=kubernetes source=kubernetes earliest=-15m
```

## Troubleshooting

- **Connection refused**: Verify the HEC host and port. Check firewall rules.
- **403 Forbidden**: The HEC token is invalid or disabled. Check Splunk HEC settings.
- **400 Bad Request**: The event format is invalid. Check the sourcetype configuration.
- **SSL errors**: Set `insecure_ssl: true` for testing, or provide the correct CA certificate.
- **No data in Splunk**: Verify the index exists and the HEC token has permission to write to it.

## Summary

Sending logs to Splunk from Rancher uses the HTTP Event Collector integration through ClusterOutputs. Configure the HEC endpoint, token, and index settings, then create ClusterFlows to route logs. Use different indexes for infrastructure and application logs, and enrich events with Kubernetes metadata for effective searching in Splunk.
