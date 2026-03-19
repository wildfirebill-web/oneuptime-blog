# How to Configure Rancher Webhooks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Webhooks, Automation

Description: Learn how to set up and configure webhooks in Rancher to automate responses to cluster events, scaling triggers, and policy enforcement.

Rancher webhooks allow you to trigger automated actions based on events in your clusters. You can use them to enforce policies, automate scaling, send notifications, and integrate with external systems. This guide covers setting up and configuring Rancher webhooks for common use cases.

## Understanding Rancher Webhooks

Rancher provides a webhook receiver that listens for specific events and executes actions in response. The webhook system is deployed as part of the `rancher-webhook` component and handles admission control, validation, and mutation of Kubernetes resources.

There are two types of webhook functionality in Rancher:

1. **Admission Webhooks**: Built-in validation and mutation webhooks that enforce Rancher policies
2. **Custom Webhook Receivers**: User-configurable webhooks that trigger actions on external events

## Setting Up the Rancher Webhook Receiver

### Installing via Helm

The Rancher webhook is typically installed automatically with Rancher. Verify it is running:

```bash
kubectl get pods -n cattle-system -l app=rancher-webhook

kubectl get validatingwebhookconfigurations | grep rancher
kubectl get mutatingwebhookconfigurations | grep rancher
```

If you need to reinstall or upgrade:

```bash
helm upgrade --install rancher-webhook rancher-charts/rancher-webhook \
  --namespace cattle-system \
  --set certs.mode=auto
```

## Creating Webhook Receivers for Scaling

Rancher allows you to create webhook receivers that can scale workloads up or down based on external triggers.

### Step 1: Create a Webhook Receiver via the API

```bash
export RANCHER_URL="https://rancher.example.com"
export RANCHER_TOKEN="token-xxxxx:yyyyyyyyyyyyyyyy"
export CLUSTER_ID="c-m-abc12345"
export PROJECT_ID="c-m-abc12345:p-xyz789"

# Create a scale-up webhook
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "scale-up-nginx",
    "driver": "scaleService",
    "scaleServiceConfig": {
      "action": "up",
      "amount": 2,
      "serviceId": "deployment:default:nginx",
      "min": 1,
      "max": 20
    }
  }' \
  "${RANCHER_URL}/v3/projects/${PROJECT_ID}/receivers"
```

The response includes a webhook URL:

```json
{
  "id": "receiver-xxxxx",
  "url": "https://rancher.example.com/hooks/xxxxx?token=yyyyy"
}
```

### Step 2: Create a Scale-Down Webhook

```bash
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "scale-down-nginx",
    "driver": "scaleService",
    "scaleServiceConfig": {
      "action": "down",
      "amount": 1,
      "serviceId": "deployment:default:nginx",
      "min": 1,
      "max": 20
    }
  }' \
  "${RANCHER_URL}/v3/projects/${PROJECT_ID}/receivers"
```

### Step 3: Trigger the Webhook

Call the webhook URL to trigger the scaling action:

```bash
WEBHOOK_URL="https://rancher.example.com/hooks/xxxxx?token=yyyyy"

curl -s -k -X POST "${WEBHOOK_URL}"
```

## Integrating with External Monitoring

### Alertmanager Integration

Configure Alertmanager to call your Rancher webhook when alerts fire:

```yaml
# alertmanager.yml
route:
  receiver: 'rancher-scaler'
  routes:
    - match:
        alertname: HighCPU
      receiver: 'rancher-scale-up'
    - match:
        alertname: LowCPU
      receiver: 'rancher-scale-down'

receivers:
  - name: 'rancher-scale-up'
    webhook_configs:
      - url: 'https://rancher.example.com/hooks/scale-up-xxxxx?token=yyyyy'
        send_resolved: false

  - name: 'rancher-scale-down'
    webhook_configs:
      - url: 'https://rancher.example.com/hooks/scale-down-xxxxx?token=yyyyy'
        send_resolved: false
```

### Prometheus Alert Rules

Create alert rules that trigger your webhooks:

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: scaling-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: scaling
      rules:
        - alert: HighCPU
          expr: avg(rate(container_cpu_usage_seconds_total{namespace="default", pod=~"nginx.*"}[5m])) > 0.8
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "High CPU usage on nginx pods"

        - alert: LowCPU
          expr: avg(rate(container_cpu_usage_seconds_total{namespace="default", pod=~"nginx.*"}[5m])) < 0.2
          for: 10m
          labels:
            severity: info
          annotations:
            summary: "Low CPU usage on nginx pods"
```

## Building a Custom Webhook Endpoint

If you need more complex logic than Rancher's built-in webhook receivers provide, build a custom webhook server:

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/exec"
)

type AlertmanagerPayload struct {
    Alerts []struct {
        Status string            `json:"status"`
        Labels map[string]string `json:"labels"`
    } `json:"alerts"`
}

func webhookHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }

    var payload AlertmanagerPayload
    if err := json.NewDecoder(r.Body).Decode(&payload); err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }

    for _, alert := range payload.Alerts {
        log.Printf("Alert: %s, Status: %s", alert.Labels["alertname"], alert.Status)

        switch alert.Labels["alertname"] {
        case "HighCPU":
            scaleDeployment("default", "nginx", 5)
        case "LowCPU":
            scaleDeployment("default", "nginx", 2)
        }
    }

    w.WriteHeader(http.StatusOK)
    fmt.Fprint(w, "OK")
}

func scaleDeployment(namespace, name string, replicas int) {
    cmd := exec.Command("kubectl", "scale", "deployment",
        name, "-n", namespace,
        fmt.Sprintf("--replicas=%d", replicas))
    output, err := cmd.CombinedOutput()
    if err != nil {
        log.Printf("Error scaling: %v, output: %s", err, output)
        return
    }
    log.Printf("Scaled %s/%s to %d replicas", namespace, name, replicas)
}

func main() {
    http.HandleFunc("/webhook", webhookHandler)
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    log.Printf("Webhook server listening on :%s", port)
    log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

Deploy this as a service in your cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook-handler
  namespace: automation
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webhook-handler
  template:
    metadata:
      labels:
        app: webhook-handler
    spec:
      serviceAccountName: webhook-handler
      containers:
        - name: handler
          image: your-registry/webhook-handler:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: webhook-handler
  namespace: automation
spec:
  selector:
    app: webhook-handler
  ports:
    - port: 80
      targetPort: 8080
```

## Managing Webhook Receivers

### List All Webhook Receivers

```bash
curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/projects/${PROJECT_ID}/receivers" | jq '.data[] | {
    id,
    name,
    driver,
    url: .url
  }'
```

### Delete a Webhook Receiver

```bash
RECEIVER_ID="receiver-xxxxx"

curl -s -k -X DELETE \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/projects/${PROJECT_ID}/receivers/${RECEIVER_ID}"
```

## Configuring Admission Webhook Policies

Rancher's admission webhooks can enforce policies on resource creation and modification.

### Viewing Current Webhook Configurations

```bash
kubectl get validatingwebhookconfigurations rancher.cattle.io -o yaml
```

### Escalation Prevention

The Rancher webhook automatically prevents privilege escalation. If a user tries to create a role with more permissions than they have, the webhook blocks it:

```bash
# This would be rejected if the user lacks cluster-admin privileges
kubectl create clusterrolebinding admin-binding \
  --clusterrole=cluster-admin \
  --user=regular-user
```

## Troubleshooting Webhooks

### Check Webhook Pod Logs

```bash
kubectl logs -n cattle-system -l app=rancher-webhook --tail=100
```

### Verify Webhook Endpoint Connectivity

```bash
# From inside the cluster
kubectl run curl-test --rm -it --image=curlimages/curl -- \
  curl -s -o /dev/null -w "%{http_code}" https://rancher.example.com/hooks/xxxxx
```

### Common Issues

If webhook receivers are not triggering, check:

1. The webhook URL is accessible from the calling system
2. The token in the webhook URL has not expired
3. The target workload exists and the service ID is correct
4. Network policies are not blocking the webhook traffic

## Summary

Rancher webhooks provide a powerful mechanism for automating responses to events in your clusters. Use built-in webhook receivers for simple scaling operations, integrate with Alertmanager for monitoring-driven automation, and build custom webhook servers for complex logic. The built-in admission webhooks handle policy enforcement automatically, preventing privilege escalation and resource validation issues.
