# How to Configure Alert Notifications via Microsoft Teams in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Alerting, Microsoft Teams, Monitoring

Description: Set up Microsoft Teams notifications for Rancher alerts using Alertmanager webhook integration with Teams connectors.

Microsoft Teams is widely used in enterprise environments for team communication. Integrating Rancher alerts with Teams channels keeps your operations team informed about cluster events without switching contexts. This guide covers setting up Teams webhooks and configuring Alertmanager to send formatted alert notifications.

## Prerequisites

- Rancher v2.6 or later with the Monitoring chart installed.
- Cluster admin permissions.
- A Microsoft Teams workspace with permission to add connectors to channels.

## Step 1: Create a Teams Incoming Webhook

1. Open Microsoft Teams and navigate to the channel where you want to receive alerts.
2. Click the three-dot menu on the channel name and select **Connectors** (or **Manage channel > Connectors**).
3. Find **Incoming Webhook** and click **Configure**.
4. Give the webhook a name like "Rancher Alerts" and optionally upload an icon.
5. Click **Create**.
6. Copy the webhook URL. It will look like: `https://outlook.office.com/webhook/...`.

## Step 2: Create a Secret for the Webhook URL

Store the Teams webhook URL as a Kubernetes secret:

```bash
kubectl create secret generic alertmanager-teams-secret \
  --namespace cattle-monitoring-system \
  --from-literal=webhook-url='https://outlook.office.com/webhook/your-webhook-url'
```

## Step 3: Configure Alertmanager for Microsoft Teams

Alertmanager does not have a native Microsoft Teams integration, so you need to use the generic webhook receiver with Teams-compatible message formatting. However, the simplest approach is to use the `msteams` receiver through the webhook configuration.

Configure the monitoring chart values:

```yaml
alertmanager:
  alertmanagerSpec:
    secrets:
      - alertmanager-teams-secret
  config:
    route:
      receiver: "teams-notifications"
      group_by:
        - alertname
        - namespace
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
    receivers:
      - name: "teams-notifications"
        webhook_configs:
          - url: "http://prometheus-msteams.cattle-monitoring-system.svc:2000/alertmanager"
            send_resolved: true
```

## Step 4: Deploy the Prometheus-MSTeams Adapter

Since Alertmanager lacks native Teams support, deploy the prometheus-msteams adapter that converts Alertmanager webhooks into Teams-compatible messages:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-msteams
  namespace: cattle-monitoring-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-msteams
  template:
    metadata:
      labels:
        app: prometheus-msteams
    spec:
      containers:
        - name: prometheus-msteams
          image: quay.io/prometheusmsteams/prometheus-msteams:v1.5.2
          ports:
            - containerPort: 2000
          env:
            - name: TEAMS_INCOMING_WEBHOOK_URL
              valueFrom:
                secretKeyRef:
                  name: alertmanager-teams-secret
                  key: webhook-url
            - name: TEAMS_REQUEST_URI
              value: "alertmanager"
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-msteams
  namespace: cattle-monitoring-system
spec:
  selector:
    app: prometheus-msteams
  ports:
    - port: 2000
      targetPort: 2000
```

Apply the deployment:

```bash
kubectl apply -f prometheus-msteams.yaml
```

## Step 5: Configure Custom Message Templates

Create a custom message card template for richer Teams notifications:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-msteams-config
  namespace: cattle-monitoring-system
data:
  card.tmpl: |
    {{ define "teams.card" }}
    {
      "@type": "MessageCard",
      "@context": "http://schema.org/extensions",
      "themeColor": "{{- if eq .Status "firing" -}}FF0000{{- else -}}00FF00{{- end -}}",
      "summary": "{{ .CommonLabels.alertname }}",
      "sections": [{
        "activityTitle": "{{ if eq .Status "firing" }}🔴{{ else }}✅{{ end }} {{ .CommonLabels.alertname }}",
        "facts": [
          {{ range $i, $alert := .Alerts }}
          {
            "name": "Severity",
            "value": "{{ $alert.Labels.severity }}"
          },
          {
            "name": "Namespace",
            "value": "{{ $alert.Labels.namespace }}"
          },
          {
            "name": "Summary",
            "value": "{{ $alert.Annotations.summary }}"
          },
          {
            "name": "Description",
            "value": "{{ $alert.Annotations.description }}"
          }
          {{ if lt $i (sub (len $.Alerts) 1) }},{{ end }}
          {{ end }}
        ],
        "markdown": true
      }],
      "potentialAction": [{
        "@type": "OpenUri",
        "name": "View in Rancher",
        "targets": [{
          "os": "default",
          "uri": "https://rancher.example.com"
        }]
      }]
    }
    {{ end }}
```

Mount the template in the prometheus-msteams deployment:

```yaml
volumes:
  - name: config
    configMap:
      name: prometheus-msteams-config
volumeMounts:
  - name: config
    mountPath: /etc/prometheus-msteams/
```

## Step 6: Route Alerts to Different Teams Channels

To send alerts to multiple Teams channels, create separate webhooks and adapter endpoints:

```bash
kubectl create secret generic teams-platform-webhook \
  --namespace cattle-monitoring-system \
  --from-literal=webhook-url='https://outlook.office.com/webhook/platform-channel-url'

kubectl create secret generic teams-devops-webhook \
  --namespace cattle-monitoring-system \
  --from-literal=webhook-url='https://outlook.office.com/webhook/devops-channel-url'
```

Configure separate request URIs in the adapter and route accordingly in Alertmanager:

```yaml
alertmanager:
  config:
    route:
      receiver: "teams-default"
      routes:
        - receiver: "teams-platform"
          matchers:
            - team = platform
        - receiver: "teams-devops"
          matchers:
            - team = devops
    receivers:
      - name: "teams-default"
        webhook_configs:
          - url: "http://prometheus-msteams.cattle-monitoring-system.svc:2000/default"
      - name: "teams-platform"
        webhook_configs:
          - url: "http://prometheus-msteams.cattle-monitoring-system.svc:2000/platform"
      - name: "teams-devops"
        webhook_configs:
          - url: "http://prometheus-msteams.cattle-monitoring-system.svc:2000/devops"
```

## Step 7: Test the Integration

Create a test alert:

```bash
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: test-teams-alert
  namespace: cattle-monitoring-system
  labels:
    app: rancher-monitoring
    release: rancher-monitoring
spec:
  groups:
    - name: test
      rules:
        - alert: TestTeamsNotification
          expr: vector(1)
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "Test Teams notification"
            description: "This is a test alert to verify Microsoft Teams integration."
EOF
```

After verification, clean up:

```bash
kubectl delete prometheusrule test-teams-alert -n cattle-monitoring-system
```

## Troubleshooting

Check the prometheus-msteams adapter logs:

```bash
kubectl logs -n cattle-monitoring-system -l app=prometheus-msteams
```

Common issues:

- **Webhook URL expired**: Microsoft Teams webhook URLs can expire. Create a new connector if needed.
- **Message formatting errors**: Check the adapter logs for template rendering errors.
- **Network connectivity**: Ensure the cluster can reach `outlook.office.com` on port 443.

## Summary

Microsoft Teams integration with Rancher alerts requires deploying a webhook adapter (prometheus-msteams) since Alertmanager does not have native Teams support. Set up Teams incoming webhooks, deploy the adapter, configure Alertmanager to route alerts through the adapter, and use custom message templates for rich notification cards.
