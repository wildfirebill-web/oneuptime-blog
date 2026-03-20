# How to Enable NeuVector Deep Packet Inspection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Deep Packet Inspection, DPI, Network Security, Container Security

Description: Learn how to enable and configure NeuVector's Deep Packet Inspection (DPI) to analyze Layer 7 traffic and detect application-level threats in containerized workloads.

## Introduction

NeuVector's Deep Packet Inspection (DPI) goes beyond port-based network filtering to analyze the actual content of network packets. DPI allows NeuVector to identify and enforce application protocols, detect anomalous patterns, and apply WAF rules at the network level. This guide explains how to enable and configure DPI for maximum security visibility.

## What DPI Provides

With DPI enabled, NeuVector can:

- Identify application protocols regardless of the port used (e.g., HTTP on port 9090)
- Apply protocol-specific rules (HTTP methods, DNS queries, SQL patterns)
- Detect protocol violations and anomalies
- Enable WAF signature matching
- Enable DLP (Data Loss Prevention) inspection
- Generate detailed network activity logs

## Prerequisites

- NeuVector with Enforcer DaemonSet running on all nodes
- Sufficient CPU resources (DPI adds ~10-15% CPU overhead per node)
- NeuVector Manager access

## Step 1: Verify DPI Capability

First, confirm DPI is available in your environment:

```bash
# Check enforcer capabilities
kubectl logs -n neuvector -l app=neuvector-enforcer-pod \
  --tail=20 | grep -i "dpi\|inspection"

# Check controller status
curl -sk \
  "https://neuvector-manager:8443/v1/system/summary" \
  -H "X-Auth-Token: ${TOKEN}" | jq '{
    total_enforcers: .summary.total_enforcers,
    total_connections: .summary.total_connections
  }'
```

## Step 2: Enable DPI at the Group Level

DPI is enabled per group. Enable it for specific workloads:

```bash
# Enable DPI for a group
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/group/nv.webapp.default" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "mode": "Protect",
      "not_scored": false
    }
  }'
```

In the NeuVector UI:
1. Go to **Policy** > **Groups**
2. Select a group
3. Click **Edit**
4. Ensure the group mode is set to **Monitor** or **Protect** (DPI is active in these modes)

## Step 3: Configure Network Rules with Application Layer Protocols

DPI enables protocol-level filtering in network rules:

```bash
# Create a rule that enforces HTTP protocol on port 8080
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/policy/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "insert": {
      "after": 0,
      "rules": [
        {
          "comment": "Allow HTTP between services",
          "from": "nv.frontend.default",
          "to": "nv.api.default",
          "ports": "tcp/8080",
          "applications": ["HTTP"],
          "action": "allow"
        },
        {
          "comment": "Allow gRPC communication",
          "from": "nv.service-a.default",
          "to": "nv.service-b.default",
          "ports": "tcp/9090",
          "applications": ["GRPC"],
          "action": "allow"
        },
        {
          "comment": "Allow MySQL",
          "from": "nv.app.default",
          "to": "nv.mysql.default",
          "ports": "tcp/3306",
          "applications": ["MySQL"],
          "action": "allow"
        }
      ]
    }
  }'
```

## Step 4: Supported Application Protocols

NeuVector DPI supports inspection of these protocols:

```
Layer 7 Protocols:
- HTTP / HTTPS
- gRPC
- DNS
- DHCP
- NTP
- TFTP
- Echo
- MySQL
- PostgreSQL
- Redis
- MongoDB
- Cassandra
- Elasticsearch
- Kafka
- Zookeeper
- SSH
- SSL/TLS (metadata)
```

Configure rules to use these protocol names in the `applications` field.

## Step 5: Monitor DPI-Generated Traffic Events

DPI generates detailed events for protocol violations:

```bash
# View network events with protocol details
curl -sk \
  "https://neuvector-manager:8443/v1/event?type=network&start=0&limit=50" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.events[] | {
    from: .workload_name,
    to: .remote_workload_name,
    protocol: .application,
    port: .port,
    action: .action,
    reason: .reason
  }'
```

## Step 6: Enable DPI for WAF Rules

DPI is required for WAF (Web Application Firewall) rules to work:

```bash
# Create a WAF sensor that uses DPI-inspected HTTP traffic
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/dpi/waf/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "sql-injection-detection",
      "comment": "Detect SQL injection in HTTP requests",
      "patterns": [
        {
          "key": "request",
          "op": "regex",
          "value": "(?i)(select|insert|update|delete|drop|union).*from",
          "context": "uri",
          "name": "sql-inject-pattern"
        }
      ]
    }
  }'
```

## Step 7: Configure DPI for DLP Sensors

DLP requires DPI to inspect packet payloads:

```bash
# Create a DLP rule to detect credit card numbers in traffic
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/dpi/dlp/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "credit-card-detection",
      "comment": "Detect credit card numbers",
      "patterns": [
        {
          "key": "packet",
          "op": "regex",
          "value": "\\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14})\\b",
          "context": "packet",
          "name": "cc-pattern"
        }
      ]
    }
  }'
```

## Step 8: Tune DPI Performance

DPI can impact performance. Tune it for your environment:

```bash
# Monitor enforcer CPU usage during DPI operation
kubectl top pods -n neuvector -l app=neuvector-enforcer-pod

# Scale enforcers if necessary (DaemonSet, so one per node automatically)
# Adjust node resources or limit DPI to critical namespaces

# Disable DPI for low-priority groups
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/group/nv.batch-jobs.default" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "mode": "Discover"
    }
  }'
```

## Conclusion

NeuVector's Deep Packet Inspection transforms network security from port-based filtering to true application-aware enforcement. By enabling DPI, you gain the ability to enforce specific protocols, detect protocol violations, and power advanced features like WAF and DLP. While DPI adds some CPU overhead, the visibility and control it provides are essential for securing microservices in production Kubernetes environments.
