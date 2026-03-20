# How to Configure NeuVector Response Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Response Rules, Automated Response, Container Security, Kubernetes

Description: Configure NeuVector response rules to automatically take action on security events, including quarantining containers, sending webhooks, and suppressing false positives.

## Introduction

NeuVector Response Rules let you automate responses to security events. Instead of manually reviewing every alert, you can configure rules to automatically quarantine compromised containers, send webhook notifications, or suppress known false positives. This enables a fast, consistent security response without human intervention.

## Response Rule Types

NeuVector supports these automated responses:

| Response Type | Action |
|---|---|
| Quarantine | Isolate container from all network traffic |
| Webhook | Send HTTP notification to external systems |
| Suppress | Ignore known false positives |

## Prerequisites

- NeuVector installed and operational
- Security groups defined and in Monitor/Protect mode
- Webhook endpoint (optional, for notifications)
- NeuVector Manager access

## Step 1: Create a Quarantine Response Rule

Automatically isolate containers that exhibit malicious behavior:

```bash
# Create a response rule to quarantine on reverse shell detection

curl -sk -X POST \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "event": "security-event",
      "comment": "Quarantine on reverse shell detection",
      "group": "nv.webapp.production",
      "conditions": [
        {
          "type": "name",
          "value": "reverse-shell"
        }
      ],
      "actions": ["quarantine"],
      "disable": false,
      "cfg_type": "user"
    }
  }'
```

In the UI:
1. Go to **Policy** > **Response Rules**
2. Click **+ Add Rule**
3. Configure:
   - **Event**: Security Event
   - **Group**: Select your workload group
   - **Conditions**: Add conditions for when to trigger
   - **Actions**: Select **Quarantine**

## Step 2: Create a Webhook Notification Rule

Send notifications to Slack, PagerDuty, or any HTTP endpoint:

```bash
# First, configure the webhook endpoint
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/system/webhook" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "slack-security-alerts",
      "url": "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXX",
      "type": "Slack",
      "enable": true,
      "cfg_type": "user"
    }
  }'

# Create a response rule to notify on critical events
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "event": "security-event",
      "comment": "Alert on process violations",
      "group": "nv.webapp.production",
      "conditions": [
        {
          "type": "level",
          "value": "critical"
        }
      ],
      "actions": ["webhook"],
      "webhooks": ["slack-security-alerts"],
      "disable": false,
      "cfg_type": "user"
    }
  }'
```

## Step 3: Configure Event-Type-Specific Rules

Create targeted rules for different security event types:

```bash
# Rule: Quarantine on privilege escalation
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "event": "security-event",
      "comment": "Quarantine on privilege escalation attempt",
      "conditions": [
        {"type": "name", "value": "privilege-escalation"}
      ],
      "actions": ["quarantine", "webhook"],
      "webhooks": ["pagerduty-critical"],
      "disable": false
    }
  }'

# Rule: Notify on network policy violation
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "event": "security-event",
      "comment": "Notify on network policy violations",
      "conditions": [
        {"type": "level", "value": "high"}
      ],
      "actions": ["webhook"],
      "webhooks": ["slack-security-alerts"],
      "disable": false
    }
  }'
```

## Step 4: Create Suppression Rules (False Positive Management)

Suppress known benign events that generate noise:

```bash
# Suppress known benign process alerts
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "event": "security-event",
      "comment": "Suppress known health check process alerts",
      "group": "nv.webapp.default",
      "conditions": [
        {"type": "name", "value": "healthcheck-sh"}
      ],
      "actions": ["suppress"],
      "disable": false
    }
  }'
```

## Step 5: Set Up Rules for Vulnerability Events

Respond to vulnerability scan results:

```bash
# Notify when critical CVEs are found
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "event": "cve-report",
      "comment": "Alert on critical CVE discovery",
      "conditions": [
        {"type": "level", "value": "critical"}
      ],
      "actions": ["webhook"],
      "webhooks": ["slack-security-alerts"],
      "disable": false
    }
  }'
```

## Step 6: Test Quarantine Behavior

Verify quarantine works as expected:

```bash
# Check if a container is quarantined
WORKLOAD_ID="abc123"
curl -sk \
  "https://neuvector-manager:8443/v1/workload/${WORKLOAD_ID}" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.workload.quarantine'

# Manually quarantine a container for testing
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/workload/${WORKLOAD_ID}" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{"config": {"quarantine": true}}'

# Release quarantine
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/workload/${WORKLOAD_ID}" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{"config": {"quarantine": false}}'
```

## Step 7: Manage Existing Response Rules

```bash
# List all response rules
curl -sk \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.rules[] | {
    id: .id,
    event: .event,
    comment: .comment,
    actions: .actions,
    disabled: .disable
  }'

# Disable a response rule
RULE_ID=5
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/response/rule/${RULE_ID}" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{"config": {"disable": true}}'
```

## Conclusion

NeuVector Response Rules automate your incident response workflow, reducing the time between threat detection and containment. By combining quarantine actions with webhook notifications, you can automatically isolate compromised containers and alert your security team in real time. Start with notification-only rules to understand your event volume, then progressively enable automated quarantine for the highest-confidence threat detections.
