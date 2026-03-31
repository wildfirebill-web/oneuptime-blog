# How to Set Up Dapr Alerting for Production

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Alerting, Prometheus, PagerDuty, Production, Observability

Description: Configure production-ready alerting for Dapr using Prometheus AlertManager rules covering control plane health, error rates, and pub/sub lag.

---

Effective alerting for Dapr production deployments catches failures before users notice. This guide covers writing Prometheus alerting rules for the most critical Dapr signals and routing them through AlertManager.

## Core Dapr Alerting Rules

Create a PrometheusRule resource with essential Dapr alerts:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dapr-alerts
  namespace: monitoring
spec:
  groups:
  - name: dapr.control-plane
    rules:
    - alert: DaprControlPlaneDown
      expr: |
        kube_deployment_status_replicas_available{
          namespace="dapr-system"
        } < kube_deployment_spec_replicas{
          namespace="dapr-system"
        }
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Dapr control plane replicas unavailable"
        description: "{{ $labels.deployment }} has fewer replicas than desired."

  - name: dapr.service-invocation
    rules:
    - alert: DaprHighErrorRate
      expr: |
        rate(dapr_service_invocation_req_sent_total{status_code!~"2.."}[5m]) /
        rate(dapr_service_invocation_req_sent_total[5m]) > 0.05
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Dapr service invocation error rate above 5%"
        description: "App {{ $labels.app_id }} error rate is {{ $value | humanizePercentage }}."
```

## Pub/Sub Lag Alerting

Alert when messages are not being consumed fast enough:

```yaml
  - name: dapr.pubsub
    rules:
    - alert: DaprPubSubDeadLetter
      expr: |
        increase(dapr_pubsub_dead_letter_total[10m]) > 0
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Dapr dead-letter messages detected"
        description: "Topic {{ $labels.topic }} has dead-letter messages on {{ $labels.app_id }}."

    - alert: DaprPubSubDropped
      expr: |
        rate(dapr_pubsub_incoming_messages_total{process_status="drop"}[5m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Dapr is dropping pub/sub messages"
```

## AlertManager Routing Configuration

Route Dapr critical alerts to PagerDuty and warnings to Slack:

```yaml
route:
  group_by: ['alertname', 'app_id']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
  - match:
      severity: critical
    receiver: pagerduty-dapr
  - match:
      severity: warning
    receiver: slack-dapr

receivers:
- name: pagerduty-dapr
  pagerduty_configs:
  - service_key: "<your-pagerduty-key>"
    description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'

- name: slack-dapr
  slack_configs:
  - api_url: "https://hooks.slack.com/services/xxx"
    channel: '#dapr-alerts'
    text: '{{ .Annotations.description }}'
```

## Latency Alerting

High latency in service invocation often indicates resource pressure or network issues:

```bash
# Alert when p99 latency exceeds 500ms
- alert: DaprHighLatency
  expr: |
    histogram_quantile(0.99,
      rate(dapr_grpc_io_server_server_latency_bucket[5m])
    ) > 500
  for: 10m
  labels:
    severity: warning
```

## Testing Alerts

Use `amtool` to validate your AlertManager config and test alert routing:

```bash
# Validate config
amtool check-config /etc/alertmanager/alertmanager.yaml

# Test a route
amtool config routes test \
  --config.file=/etc/alertmanager/alertmanager.yaml \
  alertname=DaprControlPlaneDown severity=critical
```

## Summary

Production Dapr alerting should cover control plane availability, service invocation error rates, pub/sub dead-letter accumulation, and latency percentiles. Routing critical alerts to an on-call system and warnings to chat ensures the right team is notified at the right urgency level.
