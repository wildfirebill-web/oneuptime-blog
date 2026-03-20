# How to Configure NeuVector Webhook Notifications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Webhooks, Notifications, Slack, PagerDuty, Kubernetes

Description: Set up NeuVector webhook notifications to send real-time security alerts to Slack, PagerDuty, Microsoft Teams, and custom HTTP endpoints.

## Introduction

NeuVector can send security event notifications to external systems via webhooks. This enables real-time alerting in your communication and incident management tools, ensuring your security team is immediately aware of threats, policy violations, and compliance issues.

## Prerequisites

- NeuVector installed and running
- Webhook endpoints configured in your notification tools
- NeuVector Manager access

## Step 1: Configure a Slack Webhook

### Create a Slack Webhook URL

1. Go to https://api.slack.com/apps
2. Click **Create New App** > **From Scratch**
3. Name the app "NeuVector Security"
4. Select your workspace
5. Go to **Incoming Webhooks**
6. Enable incoming webhooks
7. Click **Add New Webhook to Workspace**
8. Select the channel (e.g., `#security-alerts`)
9. Copy the webhook URL

### Add Slack Webhook to NeuVector

```bash
# Configure Slack webhook

curl -sk -X POST \
  "https://neuvector-manager:8443/v1/system/webhook" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "slack-security",
      "url": "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXX",
      "type": "Slack",
      "enable": true,
      "cfg_type": "user"
    }
  }'
```

## Step 2: Configure a Microsoft Teams Webhook

### Create a Teams Connector

1. In Microsoft Teams, open a channel
2. Click **...** > **Connectors**
3. Search for and add **Incoming Webhook**
4. Configure the webhook name and icon
5. Copy the webhook URL

### Add Teams Webhook to NeuVector

```bash
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/system/webhook" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "teams-security",
      "url": "https://company.webhook.office.com/webhookb2/xxxx/IncomingWebhook/yyyy",
      "type": "Teams",
      "enable": true,
      "cfg_type": "user"
    }
  }'
```

## Step 3: Configure a PagerDuty Integration

### Get PagerDuty Integration Key

1. In PagerDuty, go to **Services** > **Service Directory**
2. Select or create a service
3. Go to **Integrations** > **Add an Integration**
4. Select **Events API v2**
5. Copy the **Integration Key**

### Add PagerDuty to NeuVector

```bash
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/system/webhook" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "pagerduty-critical",
      "url": "https://events.pagerduty.com/v2/enqueue",
      "type": "PagerDuty",
      "enable": true,
      "cfg_type": "user"
    }
  }'
```

Note: NeuVector sends PagerDuty-compatible payloads when the type is set to `PagerDuty`.

## Step 4: Configure a Custom HTTP Webhook

For any custom endpoint or webhook aggregator:

```bash
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/system/webhook" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "custom-siem",
      "url": "https://siem.company.com/api/events/ingest",
      "type": "JSON",
      "enable": true,
      "cfg_type": "user"
    }
  }'
```

## Step 5: Build a Custom Webhook Receiver

For advanced processing, create your own webhook receiver:

```python
# webhook-receiver.py
from flask import Flask, request, jsonify
import json
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

@app.route('/neuvector/events', methods=['POST'])
def receive_event():
    """Receive and process NeuVector security events"""
    event = request.get_json()

    if not event:
        return jsonify({'error': 'No JSON payload'}), 400

    event_level = event.get('level', 'unknown')
    event_name = event.get('name', 'unknown')
    workload = event.get('workload_name', 'unknown')
    namespace = event.get('namespace', 'unknown')

    logging.info(f"Security Event: [{event_level}] {event_name} in {namespace}/{workload}")

    # Route critical events to PagerDuty
    if event_level == 'Critical':
        trigger_pagerduty(event)

    # Route all events to Elasticsearch
    index_to_elasticsearch(event)

    return jsonify({'status': 'received'}), 200


def trigger_pagerduty(event):
    """Forward critical events to PagerDuty"""
    import requests
    payload = {
        "routing_key": "YOUR_INTEGRATION_KEY",
        "event_action": "trigger",
        "dedup_key": f"neuvector-{event.get('id', '')}",
        "payload": {
            "summary": f"NeuVector: {event.get('name')} in {event.get('namespace')}/{event.get('workload_name')}",
            "severity": "critical",
            "source": "NeuVector",
            "custom_details": event
        }
    }
    requests.post("https://events.pagerduty.com/v2/enqueue", json=payload)


def index_to_elasticsearch(event):
    """Index events to Elasticsearch"""
    import requests
    requests.post(
        "http://elasticsearch:9200/neuvector-events/_doc",
        json=event,
        headers={'Content-Type': 'application/json'}
    )


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

Deploy the receiver:

```yaml
# webhook-receiver-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nv-webhook-receiver
  namespace: security-tools
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nv-webhook-receiver
  template:
    metadata:
      labels:
        app: nv-webhook-receiver
    spec:
      containers:
        - name: receiver
          image: company/nv-webhook-receiver:latest
          ports:
            - containerPort: 8080
```

## Step 6: Test Webhook Delivery

```bash
# List configured webhooks
curl -sk \
  "https://neuvector-manager:8443/v1/system/webhook" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.webhooks[] | {
    name: .config.name,
    url: .config.url,
    type: .config.type,
    enabled: .config.enable
  }'

# Test webhook (in UI: Settings > Webhooks > Test)
# Or trigger a test security event by scanning an image
```

## Step 7: Create Response Rules That Use Webhooks

Link webhooks to specific security events via response rules:

```bash
# Alert on critical process violations via Slack
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "event": "security-event",
      "comment": "Alert on critical events",
      "conditions": [{"type": "level", "value": "critical"}],
      "actions": ["webhook"],
      "webhooks": ["slack-security", "pagerduty-critical"],
      "disable": false
    }
  }'
```

## Conclusion

NeuVector webhook notifications bridge the gap between container security events and your team's existing communication and incident management workflows. By routing different severity levels to appropriate channels (Slack for informational, PagerDuty for critical), you ensure the right people are notified with the right urgency. Custom webhook receivers give you full control over event processing, enrichment, and routing to any downstream system.
