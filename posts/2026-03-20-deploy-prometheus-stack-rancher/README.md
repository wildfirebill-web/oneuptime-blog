# How to Deploy Prometheus Stack on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Prometheus, Monitoring, Alertmanager, Kubernetes

Description: Guide to deploying the full Prometheus monitoring stack on Rancher for comprehensive cluster observability.

## Introduction

How to Deploy Prometheus Stack on Rancher on Rancher gives your team a production-ready deployment with enterprise-grade cluster management, monitoring, and access control. This guide walks through a complete setup.

## Prerequisites

- Rancher v2.7+ cluster
- Helm 3.x
- Persistent storage (Longhorn)
- Ingress controller (nginx)
- cert-manager

## Step 1: Prepare Namespace

```bash
kubectl create namespace prometheus-stack

# Configure project in Rancher

kubectl annotate namespace prometheus-stack   field.cattle.io/projectId=YOUR_PROJECT_ID
```

## Step 2: Install with Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install prometheus-stack bitnami/prometheus-stack   --namespace prometheus-stack   --set persistence.enabled=true   --set persistence.storageClass=longhorn   --set ingress.enabled=true   --set ingress.hostname=prometheus-stack.example.com   --set ingress.tls=true   --wait
```

## Step 3: Configure Storage

```yaml
# prometheus-stack-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-stack-data
  namespace: prometheus-stack
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: longhorn
```

## Step 4: Configure TLS Certificate

```yaml
# prometheus-stack-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: prometheus-stack-tls
  namespace: prometheus-stack
spec:
  secretName: prometheus-stack-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - prometheus-stack.example.com
```

## Step 5: Configure Resource Limits

```yaml
# Apply ResourceQuota to namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prometheus-stack-quota
  namespace: prometheus-stack
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    persistentvolumeclaims: "5"
```

## Step 6: Set Up Monitoring

```bash
# Check if metrics endpoint is available
kubectl exec -n prometheus-stack   $(kubectl get pods -n prometheus-stack -o name | head -1)   -- curl -s http://localhost:9090/metrics | head -20

# Create ServiceMonitor
kubectl apply -f - << SMEOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus-stack-monitor
  namespace: prometheus-stack
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: prometheus-stack
  endpoints:
  - port: http
    path: /metrics
    interval: 60s
SMEOF
```

## Step 7: Configure Backup Policy

```yaml
# Backup using Velero or custom CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: prometheus-stack-backup
  namespace: prometheus-stack
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-sa
          containers:
          - name: backup
            image: bitnami/kubectl:latest
            command:
            - sh
            - -c
            - |
              echo "Creating prometheus-stack backup..."
              kubectl exec -n prometheus-stack                 $(kubectl get pod -n prometheus-stack -l app.kubernetes.io/name=prometheus-stack -o name | head -1)                 -- /opt/bitnami/scripts/prometheus-stack/entrypoint.sh prometheus-stack-backup
          restartPolicy: OnFailure
```

## Step 8: Test the Deployment

```bash
# Verify pod status
kubectl get pods -n prometheus-stack

# Check ingress
kubectl get ingress -n prometheus-stack

# Test HTTP response
curl -L https://prometheus-stack.example.com/

# View application logs
kubectl logs -n prometheus-stack   $(kubectl get pods -n prometheus-stack -l app.kubernetes.io/name=prometheus-stack -o name | head -1)   --tail=50
```

## Upgrading

```bash
# Upgrade to latest version
helm repo update
helm upgrade prometheus-stack bitnami/prometheus-stack   --namespace prometheus-stack   --reuse-values

# Check upgrade status
kubectl rollout status deployment/prometheus-stack -n prometheus-stack
```

## Conclusion

How to Deploy Prometheus Stack on Rancher on Rancher benefits from centralized management, unified monitoring, and enterprise RBAC. The Helm-based installation makes configuration management straightforward, while Rancher's project system enables multi-team governance of the deployment.
