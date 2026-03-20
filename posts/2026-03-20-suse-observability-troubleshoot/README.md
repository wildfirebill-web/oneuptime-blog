# How to Troubleshoot SUSE Observability

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SUSE Observability, Troubleshooting, Kubernetes, Monitoring, Debugging, SUSE Rancher

Description: Learn how to diagnose and fix common SUSE Observability issues including agent connectivity problems, missing topology data, health monitor failures, and ingestion pipeline bottlenecks.

---

SUSE Observability issues typically fall into three categories: agent connectivity problems, missing or stale topology data, and server-side ingestion failures. This guide covers systematic diagnosis for each.

---

## Common Issues

| Symptom | Likely Cause |
|---|---|
| Cluster not appearing in UI | Agent cannot reach the server |
| Topology data is stale | Agent stopped sending data |
| Components missing from topology | Collector not configured for that resource type |
| Health monitors not triggering | Monitor configuration or metric name issue |
| High memory on server | Elasticsearch or Kafka backpressure |

---

## Step 1: Check Agent Connectivity

```bash
# Check if all agent pods are running

kubectl get pods -n suse-observability \
  -l app.kubernetes.io/name=suse-observability-agent

# Check node agent logs for connection errors
kubectl logs -n suse-observability \
  daemonset/suse-observability-agent-node-agent \
  | grep -i "error\|refused\|timeout\|failed"

# Check cluster agent logs
kubectl logs -n suse-observability \
  deployment/suse-observability-agent-cluster-agent \
  | grep -i "error\|refused\|timeout"
```

---

## Step 2: Verify the API Key and Server URL

```bash
# Check the API key stored in the secret
kubectl get secret suse-observability-agent \
  -n suse-observability \
  -o jsonpath='{.data.stackstate-api-key}' | base64 -d

# Verify the server URL from the agent config
kubectl get configmap suse-observability-agent \
  -n suse-observability -o yaml | grep url

# Test connectivity from the agent pod to the server
kubectl exec -n suse-observability \
  $(kubectl get pod -n suse-observability \
    -l app.kubernetes.io/component=cluster-agent -o name | head -1) \
  -- curl -v https://observability.example.com/receiver/solarwinds/health
```

---

## Step 3: Check the Ingestion Pipeline on the Server

```bash
# Check server-side pod health
kubectl get pods -n suse-observability

# Look at the receiver service logs
kubectl logs -n suse-observability \
  deployment/suse-observability-receiver \
  | tail -100

# Check Kafka consumer lag (high lag = ingestion bottleneck)
kubectl exec -n suse-observability \
  $(kubectl get pod -n suse-observability -l app=kafka -o name | head -1) \
  -- kafka-consumer-groups.sh \
     --bootstrap-server localhost:9092 \
     --describe --all-groups
```

---

## Step 4: Diagnose Missing Topology Components

If specific resource types are missing from the topology:

```bash
# Check which collectors are enabled
kubectl get configmap suse-observability-agent \
  -n suse-observability -o yaml | grep -A 5 collection

# Verify the cluster agent has RBAC permissions to read the missing resources
kubectl auth can-i list deployments \
  --as=system:serviceaccount:suse-observability:suse-observability-agent-cluster-agent

# Force a topology sync
kubectl rollout restart \
  deployment/suse-observability-agent-cluster-agent \
  -n suse-observability
```

---

## Step 5: Check Elasticsearch Health

SUSE Observability stores topology and metric data in Elasticsearch:

```bash
# Check Elasticsearch cluster health
kubectl exec -n suse-observability \
  $(kubectl get pod -n suse-observability -l app=elasticsearch,role=master -o name | head -1) \
  -- curl -s localhost:9200/_cluster/health | jq .

# Check disk usage
kubectl exec -n suse-observability \
  $(kubectl get pod -n suse-observability -l app=elasticsearch,role=master -o name | head -1) \
  -- curl -s localhost:9200/_cat/allocation?v

# If Elasticsearch is RED, check for unassigned shards
kubectl exec -n suse-observability \
  $(kubectl get pod -n suse-observability -l app=elasticsearch,role=master -o name | head -1) \
  -- curl -s localhost:9200/_cat/shards?h=index,shard,prirep,state,docs \
     | grep UNASSIGNED
```

---

## Step 6: Restart Components in the Correct Order

If the server has issues, restart components in this order to avoid data loss:

```bash
# 1. Restart Zookeeper first
kubectl rollout restart statefulset/suse-observability-zookeeper -n suse-observability

# 2. Wait for Zookeeper to be ready, then restart Kafka
kubectl rollout status statefulset/suse-observability-zookeeper -n suse-observability
kubectl rollout restart statefulset/suse-observability-kafka -n suse-observability

# 3. Then restart Elasticsearch
kubectl rollout status statefulset/suse-observability-kafka -n suse-observability
kubectl rollout restart statefulset/suse-observability-elasticsearch -n suse-observability

# 4. Finally restart the main server
kubectl rollout restart deployment/suse-observability-server -n suse-observability
```

---

## Step 7: Enable Debug Logging on the Agent

```bash
# Set debug log level via Helm upgrade
helm upgrade suse-observability-agent \
  suse-observability-agent/suse-observability-agent \
  --namespace suse-observability \
  --reuse-values \
  --set nodeAgent.logLevel=debug \
  --set clusterAgent.logLevel=debug

# Tail logs with debug output
kubectl logs -n suse-observability \
  daemonset/suse-observability-agent-node-agent -f \
  | grep -v "DEBUG" | head -200
```

---

## Troubleshooting Checklist

```bash
# 1. Are all agent pods running?
kubectl get pods -n suse-observability

# 2. Can the agent reach the server?
# Check: curl from agent pod to server URL

# 3. Is the API key correct?
# Check: kubectl get secret and compare with server config

# 4. Is Elasticsearch healthy?
# Check: curl localhost:9200/_cluster/health

# 5. Is Kafka processing messages?
# Check: kafka-consumer-groups.sh consumer lag

# 6. Are RBAC permissions correct for the cluster agent?
# Check: kubectl auth can-i list <resource> --as=<serviceaccount>
```

---

## Best Practices

- Monitor the SUSE Observability server itself with Prometheus and alert on Elasticsearch disk usage and Kafka consumer lag.
- Keep the agent and server versions in sync - version mismatches can cause silent data loss.
- Use `helm upgrade --reuse-values` when changing a single setting to avoid accidentally resetting other configuration.
