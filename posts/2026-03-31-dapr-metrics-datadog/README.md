# How to Send Dapr Metrics to Datadog

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Datadog, Metrics, Monitoring, Kubernetes

Description: Learn how to collect and forward Dapr Prometheus metrics to Datadog using the Datadog Agent with OpenMetrics integration for unified observability.

---

## Overview

Sending Dapr metrics to Datadog unifies your microservice metrics with infrastructure and APM data in a single platform. The Datadog Agent can scrape Dapr's Prometheus `/metrics` endpoint and forward the data as Datadog metrics, enabling dashboards, monitors, and SLO tracking.

## Enabling Dapr Metrics Endpoint

Configure the Dapr sidecar to expose metrics:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
spec:
  metric:
    enabled: true
    port: 9090
```

Apply via pod annotation:

```yaml
annotations:
  dapr.io/config: "dapr-config"
  dapr.io/enable-metrics: "true"
  dapr.io/metrics-port: "9090"
```

## Installing the Datadog Agent

Install the Datadog Agent with Prometheus collection enabled:

```bash
helm install datadog-agent datadog/datadog \
  --namespace datadog \
  --create-namespace \
  --set datadog.apiKey="${DD_API_KEY}" \
  --set datadog.prometheusScrape.enabled=true \
  --set datadog.prometheusScrape.serviceEndpoints=true \
  --set agents.image.tag=7
```

## Configuring OpenMetrics Check via Pod Annotations

Add Datadog auto-discovery annotations to your Dapr-enabled pods to enable Prometheus scraping:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "fulfillment-service"
  dapr.io/app-port: "8080"
  dapr.io/enable-metrics: "true"
  dapr.io/metrics-port: "9090"
  ad.datadoghq.com/daprd.checks: |
    {
      "openmetrics": {
        "instances": [
          {
            "openmetrics_endpoint": "http://%%host%%:9090/metrics",
            "namespace": "dapr",
            "metrics": [
              "dapr_http_server_request_count",
              "dapr_http_server_latency",
              "dapr_resiliency_activations_total",
              "dapr_component_pubsub_count",
              "dapr_actor_active_actors"
            ]
          }
        ]
      }
    }
```

## Viewing Dapr Metrics in Datadog

Dapr metrics appear in Datadog with the namespace prefix `dapr.*`. Use the Metrics Explorer:

```
# Request rate per service
dapr.http.server.request.count{*} by {app_id}.as_rate()

# P99 latency
p99:dapr.http.server.latency{*} by {app_id}

# Resiliency activation rate
dapr.resiliency.activations.total{*}.as_rate() by {app_id}
```

## Creating a Datadog Monitor for Dapr

Create a monitor that alerts on high error rates:

```bash
curl -X POST "https://api.datadoghq.com/api/v1/monitor" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Dapr High Error Rate",
    "type": "metric alert",
    "query": "sum(last_5m):sum:dapr.http.server.request.count{status_code:5xx} by {app_id}.as_rate() > 0.05",
    "message": "Dapr service {{app_id.name}} has a high error rate. @pagerduty",
    "options": {
      "thresholds": {
        "critical": 0.05,
        "warning": 0.02
      },
      "notify_no_data": true,
      "no_data_timeframe": 10
    }
  }'
```

## Building a Dapr Dashboard in Datadog

Create a dashboard with key Dapr metrics using the Datadog API:

```bash
curl -X POST "https://api.datadoghq.com/api/v1/dashboard" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Dapr Microservices Overview",
    "layout_type": "ordered",
    "widgets": [
      {
        "definition": {
          "type": "timeseries",
          "title": "Request Rate by Service",
          "requests": [{
            "q": "sum:dapr.http.server.request.count{*} by {app_id}.as_rate()",
            "display_type": "line"
          }]
        }
      }
    ]
  }'
```

## Summary

Sending Dapr metrics to Datadog uses the Datadog Agent's OpenMetrics check, configured through pod annotations for automatic discovery. Dapr metrics appear under the `dapr.*` namespace and can be used in Datadog dashboards, monitors, and SLOs. This integration unifies Dapr service health with infrastructure metrics and APM traces in a single Datadog workspace.
