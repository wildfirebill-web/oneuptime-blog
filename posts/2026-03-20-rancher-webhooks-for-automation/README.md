# How to Use Rancher Webhooks for Automation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Webhooks, Automation, Integration, Kubernetes, Events

Description: Use Rancher webhooks and Kubernetes admission webhooks to automate responses to cluster events, enforce policies, trigger external systems, and build event-driven automation workflows.

## Introduction

Webhooks in the Rancher ecosystem serve two distinct purposes: Kubernetes admission webhooks (intercept API requests for validation/mutation) and event-driven webhooks that trigger external systems when cluster events occur. Both patterns enable automation that responds to cluster state changes without polling.

## Part 1: Kubernetes Admission Webhooks

### Validating Webhook: Enforce Naming Conventions

```yaml
# Deployment to validate pod names follow naming conventions
apiVersion: apps/v1
kind: Deployment
metadata:
  name: naming-validator
  namespace: webhook-system
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: validator
          image: myregistry/webhook-validator:1.0.0
          ports:
            - containerPort: 8443
          volumeMounts:
            - name: tls-certs
              mountPath: /tls
      volumes:
        - name: tls-certs
          secret:
            secretName: webhook-tls
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: naming-convention-validator
  annotations:
    cert-manager.io/inject-ca-from: webhook-system/webhook-cert
webhooks:
  - name: validate.naming.company.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    rules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        resources: ["deployments"]
        operations: ["CREATE", "UPDATE"]
    clientConfig:
      service:
        name: naming-validator
        namespace: webhook-system
        path: /validate
    failurePolicy: Fail
    namespaceSelector:
      matchLabels:
        enforce-naming: "true"
```

Webhook handler (Python):
```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/validate', methods=['POST'])
def validate():
    review = request.json
    deployment = review['request']['object']
    name = deployment['metadata']['name']

    # Enforce pattern: {team}-{app}-{env}
    import re
    if not re.match(r'^[a-z]+-[a-z]+-(?:prod|staging|dev)$', name):
        return jsonify({
            'apiVersion': 'admission.k8s.io/v1',
            'kind': 'AdmissionReview',
            'response': {
                'uid': review['request']['uid'],
                'allowed': False,
                'status': {
                    'message': f'Deployment name "{name}" must match pattern: team-app-env'
                }
            }
        })

    return jsonify({'response': {'uid': review['request']['uid'], 'allowed': True}})
```

### Mutating Webhook: Auto-Inject Labels

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: label-injector
webhooks:
  - name: inject.labels.company.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    rules:
      - apiGroups: [""]
        resources: ["pods"]
        operations: ["CREATE"]
    clientConfig:
      service:
        name: label-injector
        namespace: webhook-system
        path: /mutate
    failurePolicy: Ignore    # Don't block pods if webhook fails
```

## Part 2: Event-Driven External Webhooks

### Step 2: Alertmanager Webhooks to Slack/PagerDuty

```yaml
# alertmanager.yaml - Send alerts to custom webhook
receivers:
  - name: rancher-ops-webhook
    webhook_configs:
      - url: "https://ops-automation.company.com/rancher-alert"
        send_resolved: true
        http_config:
          authorization:
            type: Bearer
            credentials_file: /etc/alertmanager/webhook-token
```

### Step 3: Build an Event Router

```yaml
# Deploy event-router to forward K8s events to external systems
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-router
  namespace: kube-system
spec:
  template:
    spec:
      serviceAccountName: event-router-sa
      containers:
        - name: event-router
          image: giantswarm/eventrouter:latest
          env:
            - name: SINK
              value: "webhook"
            - name: WEBHOOK_URL
              value: "https://ops-automation.company.com/k8s-events"
            - name: WEBHOOK_HEADERS
              value: "Authorization: Bearer ${WEBHOOK_TOKEN}"
```

### Step 4: Fleet GitRepo Webhook Trigger

```bash
# Trigger Fleet sync when GitHub push happens
# Configure GitHub webhook to call Rancher Fleet

# In GitHub repository settings:
# Payload URL: https://rancher.company.com/v1/fleet/namespaces/fleet-default/gitrepos/myapp?action=forceUpdate
# Content type: application/json
# Secret: <webhook-secret>
# Events: Push

# Verify webhook delivery
curl -s -H "Authorization: Bearer $RANCHER_TOKEN" \
  -X POST \
  "https://rancher.company.com/v1/fleet/namespaces/fleet-default/gitrepos/myapp?action=forceUpdate"
```

### Step 5: Custom Event Handler Service

```python
# webhook_handler.py - Process Rancher/K8s events for automation

from flask import Flask, request, jsonify
import subprocess
import json

app = Flask(__name__)

@app.route('/k8s-events', methods=['POST'])
def handle_event():
    event = request.json
    event_type = event.get('type')
    reason = event.get('reason')
    namespace = event.get('involvedObject', {}).get('namespace')

    # Auto-scale response to resource pressure
    if reason == 'FailedScheduling' and 'Insufficient' in event.get('message', ''):
        notify_ops_channel(
            f":warning: Pod scheduling failed in {namespace}: {event['message']}\n"
            f"Consider scaling cluster or reducing resource requests."
        )

    # Alert on security events
    if reason in ['Failed', 'BackOff'] and event_type == 'Warning':
        if 'ImagePullBackOff' in event.get('message', ''):
            notify_ops_channel(
                f":x: Image pull failure in {namespace}: {event['message']}"
            )

    return jsonify({'status': 'processed'})

def notify_ops_channel(message: str):
    import requests
    requests.post(SLACK_WEBHOOK_URL, json={'text': message})
```

## Conclusion

Rancher webhooks enable both policy enforcement and event-driven automation. Validating admission webhooks block non-compliant deployments at the API level, while mutating webhooks auto-inject labels, sidecars, and annotations. For external integration, route Kubernetes events through an event-router to trigger Slack notifications, PagerDuty incidents, or custom automation scripts. Combine with Alertmanager webhooks for complete observability-driven automation.
