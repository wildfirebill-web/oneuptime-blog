# How to Configure Alerting via Webhooks for Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Alert, Webhook, Monitoring

Description: Configure Alertmanager webhook receivers to send Rook-Ceph Prometheus alerts to any HTTP endpoint, Slack, or custom automation systems.

---

## Overview

The Alertmanager webhook receiver is the most flexible notification method, capable of delivering alerts to any system that accepts HTTP POST requests. This covers Slack, Microsoft Teams, custom automation scripts, incident management platforms, and observability tools like OneUptime.

## Webhook Receiver Basics

Alertmanager sends a POST request with a JSON payload to the configured URL. The payload includes all firing alert labels, annotations, and status information.

## Configure a Slack Webhook

Create a Slack incoming webhook URL from your Slack workspace settings, then configure Alertmanager:

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: rook-ceph-slack
  namespace: rook-ceph
spec:
  route:
    groupBy: ["alertname"]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 6h
    matchers:
    - name: namespace
      value: rook-ceph
    receiver: slack-webhook
  receivers:
  - name: slack-webhook
    slackConfigs:
    - apiURL:
        name: slack-webhook-secret
        key: url
      channel: "#storage-alerts"
      title: "Rook-Ceph Alert: {{ .GroupLabels.alertname }}"
      text: |
        *Severity:* {{ .CommonLabels.severity }}
        *Summary:* {{ .CommonAnnotations.summary }}
        *Description:* {{ .CommonAnnotations.description }}
      sendResolved: true
```

Create the Slack URL secret:

```bash
kubectl create secret generic slack-webhook-secret \
  --from-literal=url="https://hooks.slack.com/services/T00000/B00000/XXXXXXXX" \
  -n rook-ceph
```

## Configure a Generic Webhook Receiver

For custom endpoints or services like OneUptime:

```yaml
receivers:
- name: generic-webhook
  webhookConfigs:
  - url: "https://oneuptime.example.com/incoming-webhook/rook-alerts"
    sendResolved: true
    httpConfig:
      bearerTokenSecret:
        name: webhook-auth-secret
        key: token
```

## Webhook Payload Format

Alertmanager POSTs JSON in this format:

```json
{
  "version": "4",
  "groupKey": "{}:{alertname=\"CephHealthError\"}",
  "status": "firing",
  "receiver": "generic-webhook",
  "groupLabels": {"alertname": "CephHealthError"},
  "commonLabels": {
    "alertname": "CephHealthError",
    "namespace": "rook-ceph",
    "severity": "critical"
  },
  "commonAnnotations": {
    "summary": "Ceph cluster is in error state",
    "description": "Ceph health is HEALTH_ERR"
  },
  "alerts": [
    {
      "status": "firing",
      "labels": {"alertname": "CephHealthError"},
      "annotations": {"summary": "Ceph cluster is in error state"},
      "startsAt": "2026-03-31T10:00:00Z"
    }
  ]
}
```

## Test the Webhook

Use a test HTTP endpoint like requestbin or a local netcat listener:

```bash
# Start a local listener
nc -l -p 8080 &

# Port-forward Alertmanager
kubectl port-forward -n monitoring svc/alertmanager-operated 9093:9093 &

# Send a test alert
curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"CephHealthError","namespace":"rook-ceph","severity":"critical"},"annotations":{"summary":"Test alert"}}]'
```

## Summary

The Alertmanager webhook receiver provides maximum flexibility for routing Rook-Ceph alerts to any destination. Slack and Microsoft Teams have native Alertmanager integrations, while custom HTTP endpoints can receive the standardized Alertmanager JSON payload and trigger any automation - from creating incidents in OneUptime to running remediation scripts.
