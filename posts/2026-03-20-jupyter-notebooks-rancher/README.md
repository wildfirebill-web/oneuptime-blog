# How to Set Up Jupyter Notebooks on Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Jupyter, Notebook, Data-science, Kubernetes

Description: Guide to deploying JupyterHub on Rancher for collaborative data science notebook environments.

## Introduction

JupyterHub provides multi-user Jupyter notebook environments on Kubernetes (using Zero to JupyterHub). This guide covers deploying JupyterHub on Rancher for data science teams.

## Step 1: Install JupyterHub with Helm

```bash
# Add JupyterHub Helm repository

helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm repo update
```

## Step 2: Configure JupyterHub Values

```yaml
# jupyterhub-values.yaml
hub:
  config:
    JupyterHub:
      admin_access: true
    Authenticator:
      admin_users:
        - admin@example.com
    OAuthenticator:
      client_id: "your-oauth-client-id"
      client_secret: "your-oauth-client-secret"
      oauth_callback_url: "https://jupyter.example.com/hub/oauth_callback"
  
  db:
    type: postgres
    url: postgresql://jupyterhub:password@postgres:5432/jupyterhub

proxy:
  service:
    type: ClusterIP
  https:
    enabled: true
    type: letsencrypt
    letsencrypt:
      contactEmail: admin@example.com

singleuser:
  image:
    name: jupyter/datascience-notebook
    tag: python-3.11
  profileList:
  - display_name: "Small (2 CPU, 4GB RAM)"
    description: "For light data exploration"
    kubespawner_override:
      cpu_limit: 2
      cpu_guarantee: 0.5
      mem_limit: "4G"
      mem_guarantee: "1G"
  - display_name: "Large (8 CPU, 32GB RAM)"
    description: "For model training"
    kubespawner_override:
      cpu_limit: 8
      cpu_guarantee: 4
      mem_limit: "32G"
      mem_guarantee: "16G"
  - display_name: "GPU (1 GPU, 32GB RAM)"
    description: "For GPU-accelerated training"
    kubespawner_override:
      extra_resource_limits:
        nvidia.com/gpu: "1"
      cpu_limit: 4
      mem_limit: "32G"
      node_selector:
        nvidia.com/gpu.present: "true"
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule

  storage:
    capacity: 20Gi
    dynamic:
      storageClass: longhorn

  defaultUrl: "/lab"
  extraEnv:
    MLFLOW_TRACKING_URI: "http://mlflow-server.mlops.svc.cluster.local:5000"

scheduling:
  userScheduler:
    enabled: true
  podPriority:
    enabled: true
  userPlaceholder:
    enabled: true
    replicas: 2               # Pre-warm 2 placeholder pods

cull:
  enabled: true
  timeout: 3600               # Cull after 1 hour of inactivity
  every: 300                  # Check every 5 minutes
```

```bash
# Install JupyterHub
helm upgrade --install jupyterhub jupyterhub/jupyterhub \
  --namespace jupyterhub \
  --create-namespace \
  --version 3.2.1 \
  --values jupyterhub-values.yaml

kubectl wait pods --all \
  --for=condition=Ready \
  --namespace jupyterhub \
  --timeout=300s
```

## Step 3: Configure Ingress

```yaml
# jupyterhub-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jupyterhub
  namespace: jupyterhub
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  rules:
  - host: jupyter.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: proxy-public
            port:
              number: 80
  tls:
  - hosts:
    - jupyter.example.com
    secretName: jupyter-tls
```

## Step 4: Configure Shared Storage

```yaml
# shared-datasets-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-datasets
  namespace: jupyterhub
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: nfs-storage
```

```yaml
# Mount shared datasets in notebook config
singleuser:
  storage:
    extraVolumes:
    - name: shared-data
      persistentVolumeClaim:
        claimName: shared-datasets
    extraVolumeMounts:
    - name: shared-data
      mountPath: /shared
      readOnly: true             # Read-only shared data
```

## Step 5: Configure Resource Quotas per Team

```yaml
# team-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: data-science-team-quota
  namespace: jupyterhub
spec:
  hard:
    requests.cpu: "40"
    requests.memory: "200Gi"
    limits.cpu: "80"
    limits.memory: "400Gi"
    requests.nvidia.com/gpu: "4"
    limits.nvidia.com/gpu: "4"
    pods: "20"
```

## Monitoring JupyterHub

```bash
# Check active users
kubectl exec -n jupyterhub \
  $(kubectl get pods -n jupyterhub -l component=hub -o name) \
  -- jupyterhub --debug list-users

# View hub logs
kubectl logs -n jupyterhub \
  -l component=hub \
  --follow

# Check resource usage
kubectl top pods -n jupyterhub
```

## Conclusion

JupyterHub on Rancher provides a scalable, multi-user notebook environment for data science teams. With GPU support, shared storage, and resource quotas, teams can collaborate effectively while operators maintain control over resource consumption. The culling feature ensures idle notebooks don't waste cluster resources.
