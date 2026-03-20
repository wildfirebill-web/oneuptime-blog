# How to Deploy Grafana on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Grafana, Monitoring, Dashboards, Kubernetes, Helm

Description: Deploy Grafana on Rancher as a standalone visualization platform with persistent dashboard storage, data source configuration, and LDAP authentication.

## Introduction

While Rancher includes monitoring via kube-prometheus-stack (which includes Grafana), sometimes you need a standalone Grafana deployment with specific configuration, additional plugins, or custom branding. This guide deploys Grafana as a standalone service.

## Step 1: Deploy Grafana with Helm

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

```yaml
# grafana-values.yaml

adminUser: admin
adminPassword: "securepassword"

persistence:
  enabled: true
  storageClass: longhorn
  size: 10Gi    # Dashboard definitions

ingress:
  enabled: true
  ingressClassName: nginx
  hosts:
    - grafana.example.com
  tls:
    - secretName: grafana-tls
      hosts:
        - grafana.example.com
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod

# Pre-configure data sources
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
        isDefault: true
      - name: Loki
        type: loki
        url: http://loki.observability.svc.cluster.local:3100
      - name: Tempo
        type: tempo
        url: http://tempo.observability.svc.cluster.local:3100

# Pre-configure dashboards via ConfigMaps
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: default
        type: file
        options:
          path: /var/lib/grafana/dashboards/default

dashboards:
  default:
    kubernetes-cluster:
      gnetId: 7249      # Kubernetes Cluster Overview dashboard
      revision: 1
      datasource: Prometheus
    node-exporter:
      gnetId: 1860      # Node Exporter Full dashboard
      revision: 37
      datasource: Prometheus

# Install plugins
plugins:
  - grafana-piechart-panel
  - grafana-worldmap-panel
  - grafana-github-datasource

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "1Gi"
    cpu: "500m"
```

```bash
kubectl create namespace monitoring
helm install grafana grafana/grafana \
  --namespace monitoring \
  --values grafana-values.yaml
```

## Step 2: Configure LDAP Authentication

```ini
# grafana.ini
[auth.ldap]
enabled = true
config_file = /etc/grafana/ldap.toml

# ldap.toml
[[servers]]
host = "ldap.example.com"
port = 389
use_ssl = false
bind_dn = "cn=grafana,ou=services,dc=example,dc=com"
bind_password = "ldappassword"
search_filter = "(cn=%s)"
search_base_dns = ["ou=users,dc=example,dc=com"]

[[servers.group_mappings]]
group_dn = "cn=grafana-admins,ou=groups,dc=example,dc=com"
org_role = "Admin"
```

## Step 3: Set Up Alerts

```yaml
# Alert via Slack
notifiers:
  - name: slack-alerts
    type: slack
    settings:
      url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
      channel: "#ops-alerts"
      icon_emoji: ":grafana:"
```

## Conclusion

A standalone Grafana deployment on Rancher provides maximum flexibility for visualization. Pre-provisioning data sources and dashboards via Helm values enables GitOps management of your entire monitoring configuration-dashboards are code, not manual UI configurations.
