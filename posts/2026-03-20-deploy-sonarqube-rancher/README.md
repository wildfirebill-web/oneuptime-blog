# How to Deploy SonarQube on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, sonarqube, code-quality, kubernetes, helm

Description: Step-by-step guide to deploying SonarQube on Rancher for continuous code quality analysis.

## Introduction

How to Deploy SonarQube on Rancher on Rancher gives your team a production-ready deployment with enterprise-grade cluster management, monitoring, and access control. This guide walks through a complete setup.

## Prerequisites

- Rancher v2.7+ cluster
- Helm 3.x
- Persistent storage (Longhorn)
- Ingress controller (nginx)
- cert-manager

## Step 1: Prepare Namespace

```bash
kubectl create namespace sonarqube

# Configure project in Rancher
kubectl annotate namespace sonarqube   field.cattle.io/projectId=YOUR_PROJECT_ID
```

## Step 2: Install with Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install sonarqube bitnami/sonarqube   --namespace sonarqube   --set persistence.enabled=true   --set persistence.storageClass=longhorn   --set ingress.enabled=true   --set ingress.hostname=sonarqube.example.com   --set ingress.tls=true   --wait
```

## Step 3: Configure Storage

```yaml
# sonarqube-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarqube-data
  namespace: sonarqube
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
# sonarqube-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: sonarqube-tls
  namespace: sonarqube
spec:
  secretName: sonarqube-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - sonarqube.example.com
```

## Step 5: Configure Resource Limits

```yaml
# Apply ResourceQuota to namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: sonarqube-quota
  namespace: sonarqube
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
kubectl exec -n sonarqube   $(kubectl get pods -n sonarqube -o name | head -1)   -- curl -s http://localhost:9090/metrics | head -20

# Create ServiceMonitor
kubectl apply -f - << SMEOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sonarqube-monitor
  namespace: sonarqube
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: sonarqube
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
  name: sonarqube-backup
  namespace: sonarqube
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
              echo "Creating sonarqube backup..."
              kubectl exec -n sonarqube                 $(kubectl get pod -n sonarqube -l app.kubernetes.io/name=sonarqube -o name | head -1)                 -- /opt/bitnami/scripts/sonarqube/entrypoint.sh sonarqube-backup
          restartPolicy: OnFailure
```

## Step 8: Test the Deployment

```bash
# Verify pod status
kubectl get pods -n sonarqube

# Check ingress
kubectl get ingress -n sonarqube

# Test HTTP response
curl -L https://sonarqube.example.com/

# View application logs
kubectl logs -n sonarqube   $(kubectl get pods -n sonarqube -l app.kubernetes.io/name=sonarqube -o name | head -1)   --tail=50
```

## Upgrading

```bash
# Upgrade to latest version
helm repo update
helm upgrade sonarqube bitnami/sonarqube   --namespace sonarqube   --reuse-values

# Check upgrade status
kubectl rollout status deployment/sonarqube -n sonarqube
```

## Conclusion

How to Deploy SonarQube on Rancher on Rancher benefits from centralized management, unified monitoring, and enterprise RBAC. The Helm-based installation makes configuration management straightforward, while Rancher's project system enables multi-team governance of the deployment.
