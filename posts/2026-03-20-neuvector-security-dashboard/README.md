# How to Monitor NeuVector Security Dashboard

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Dashboards, Security Monitoring, Kubernetes, Container Security

Description: Use the NeuVector security dashboard to monitor your cluster's security posture, track vulnerabilities, and visualize container activity in real time.

## Introduction

The NeuVector Manager dashboard provides a comprehensive view of your Kubernetes cluster's security posture. It displays real-time security events, vulnerability statistics, compliance scores, and network activity - all in one place. This guide explains how to effectively use the dashboard for security monitoring and incident triage.

## Dashboard Sections Overview

The NeuVector dashboard is organized into several key sections:

- **Summary**: High-level security status and statistics
- **Security Events**: Recent violations and threats
- **Vulnerabilities**: CVE distribution and trends
- **Compliance**: Benchmark scores and findings
- **Network Activity**: Traffic visualization and anomalies
- **Risk Reports**: Aggregate risk scoring

## Prerequisites

- NeuVector installed and running
- Workloads monitored for at least 24 hours
- NeuVector Manager access

## Step 1: Access the Dashboard

```bash
# Get the NeuVector Manager URL

kubectl get svc neuvector-service-webui -n neuvector

# Access via browser at:
# https://<manager-ip>:<nodeport>
# Default credentials: admin/admin (change immediately)
```

## Step 2: Understand the Summary Panel

The main dashboard shows:

```text
Security Risk Score: A number from 0-100 (lower is better)
├── Critical CVEs: Count of critical vulnerabilities in running containers
├── High CVEs: Count of high severity vulnerabilities
├── Security Events: Count of recent violations
├── Compliance Issues: Failed compliance checks
└── Groups in Protect Mode: Percentage of protected workloads
```

Key metrics to monitor:

```bash
# Get summary metrics via API
curl -sk \
  "https://neuvector-manager:8443/v1/system/summary" \
  -H "X-Auth-Token: ${TOKEN}" | jq '{
    total_pods: .summary.total_workloads,
    critical_cve: .summary.crit_security_event,
    protect_mode_groups: .summary.groups,
    score: .summary.policy_status
  }'
```

## Step 3: Monitor Security Events in Real Time

Use the Security Events panel to track violations:

1. Go to **Notifications** > **Security Events**
2. Use filters to focus on:
   - **Level**: Critical, High, Warning
   - **Type**: Process, Network, File, Package
   - **Namespace**: Filter by application namespace
   - **Time Range**: Last hour, 24 hours, 7 days

```bash
# Get recent security events via API
curl -sk \
  "https://neuvector-manager:8443/v1/event?type=security&start=0&limit=50" \
  -H "X-Auth-Token: ${TOKEN}" | jq '
  .events |
  group_by(.level) |
  map({level: .[0].level, count: length})'
```

## Step 4: Monitor Vulnerability Trends

Track CVE counts over time:

```bash
# Get vulnerability statistics
curl -sk \
  "https://neuvector-manager:8443/v1/scan/workload?start=0&limit=1000" \
  -H "X-Auth-Token: ${TOKEN}" | jq '{
    total_scanned: .total,
    critical_total: [.workloads[].critical] | add,
    high_total: [.workloads[].high] | add,
    medium_total: [.workloads[].medium] | add,
    low_total: [.workloads[].low] | add
  }'

# Find most vulnerable containers
curl -sk \
  "https://neuvector-manager:8443/v1/scan/workload?start=0&limit=1000" \
  -H "X-Auth-Token: ${TOKEN}" | jq '[
    .workloads[] |
    select(.critical > 0 or .high > 5) |
    {name: .name, namespace: .namespace, critical: .critical, high: .high}
  ] | sort_by(.critical) | reverse | .[0:10]'
```

## Step 5: Monitor Network Activity

The Network Activity view shows real-time container communications:

1. Navigate to **Network Activity**
2. Select a namespace or group to visualize
3. Look for:
   - Unexpected connections to external IPs
   - Unusual port usage
   - New connection patterns not seen before
4. Click on a connection line to see details:
   - Protocol and port
   - Bytes transferred
   - Policy action (allowed/blocked/alerted)

```bash
# Get network connection statistics
curl -sk \
  "https://neuvector-manager:8443/v1/network/statistics" \
  -H "X-Auth-Token: ${TOKEN}" | jq '{
    total_connections: .total,
    internal: .ingress,
    external: .egress
  }'
```

## Step 6: Use the Risk Reports View

NeuVector generates risk reports that aggregate security findings:

1. Go to **Security Risks** > **Vulnerability View**
2. Filter by:
   - CVE name (search for specific CVEs)
   - Severity
   - Namespace
   - Package name
3. Click any CVE to see all affected containers

```bash
# Get risk report summary
curl -sk \
  "https://neuvector-manager:8443/v1/security/risk" \
  -H "X-Auth-Token: ${TOKEN}" | jq '{
    high_risk_workloads: .workloads.high_risk,
    compliance_score: .compliance.score,
    vulnerability_score: .vulnerability.score
  }'
```

## Step 7: Set Up Dashboard Alerts

Configure the dashboard to highlight specific conditions:

```bash
# Create a response rule to alert on critical events
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "event": "security-event",
      "comment": "Alert on critical container security events",
      "conditions": [{"type": "level", "value": "critical"}],
      "actions": ["webhook"],
      "webhooks": ["slack-security-channel"],
      "disable": false
    }
  }'
```

## Step 8: Export Dashboard Data for Reporting

```bash
#!/bin/bash
# generate-security-report.sh

DATE=$(date +%Y-%m-%d)

echo "# NeuVector Security Report - ${DATE}" > report.md
echo "" >> report.md

# Vulnerability summary
echo "## Vulnerability Summary" >> report.md
curl -sk \
  "https://neuvector-manager:8443/v1/scan/workload?start=0&limit=1000" \
  -H "X-Auth-Token: ${TOKEN}" | jq -r '"
Total Containers Scanned: \(.total)
Critical CVEs: \([.workloads[].critical] | add)
High CVEs: \([.workloads[].high] | add)
Medium CVEs: \([.workloads[].medium] | add)
"' >> report.md

# Security events count
echo "" >> report.md
echo "## Security Events (Last 24 Hours)" >> report.md
curl -sk \
  "https://neuvector-manager:8443/v1/event?type=security&start=0&limit=1000" \
  -H "X-Auth-Token: ${TOKEN}" | jq -r '
  "Total Events: \(.events | length)
Critical: \([.events[] | select(.level == "Critical")] | length)
High: \([.events[] | select(.level == "High")] | length)"' >> report.md

echo "Report generated: report.md"
```

## Conclusion

The NeuVector security dashboard provides the visibility needed to maintain a strong security posture across your Kubernetes clusters. By regularly monitoring the summary metrics, investigating security events promptly, and tracking vulnerability trends over time, you can identify security improvements and demonstrate compliance to stakeholders. Set up automated exports for weekly reports to track your security posture improvement over time.
