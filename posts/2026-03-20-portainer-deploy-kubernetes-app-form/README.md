# How to Deploy a Kubernetes Application via Form in Portainer - Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Deployment, Application, DevOps

Description: Learn how to deploy Kubernetes applications using Portainer's form-based interface without writing YAML manifests.

## Introduction

Portainer's form-based application deployment makes Kubernetes accessible to teams who are not comfortable writing YAML manifests. The form wizard generates Kubernetes Deployments, Services, ConfigMaps, and other resources based on your inputs. This guide walks through deploying a complete application using the form interface.

## Prerequisites

- Portainer with a Kubernetes environment
- At least one namespace to deploy into
- Image accessible from the cluster (registry credentials configured if private)

## Step 1: Navigate to Application Deployment

1. Select your Kubernetes environment in Portainer
2. Click **Applications** in the sidebar
3. Click **+ Add application**
4. Choose **Form** (not YAML or Helm)

## Step 2: Configure Basic Details

```text
Application name:  my-web-app
Namespace:        production
```

**Naming rules:**
- Lowercase letters, numbers, hyphens only
- Must start with a letter
- Maximum 63 characters

## Step 3: Configure the Image

```bash
Image:           nginx:alpine
Registry:        Docker Hub (or your private registry)
```

For private registries, select the registry from the dropdown - Portainer uses stored credentials automatically.

## Step 4: Configure Replicas and Resources

```text
Replicas:        3

Resource limits:
  Memory limit:   256Mi
  CPU limit:      500m

Resource requests:
  Memory request: 128Mi
  CPU request:    100m
```

## Step 5: Configure Environment Variables

Click **+ Add environment variable** for each variable:

```text
DATABASE_URL:    postgresql://postgres:5432/mydb
NODE_ENV:        production
LOG_LEVEL:       info
```

Or reference from ConfigMap/Secret:

```text
DB_PASSWORD:     valueFrom → Secret: db-secret → key: password
API_KEY:         valueFrom → ConfigMap: app-config → key: api_key
```

## Step 6: Configure Persistent Storage

Under **Volumes**, click **+ Add volume**:

```text
Volume type:       Persistent volume
Mount path:        /data
Persistent volume claim:
  Create new:       [x]
  Name:             my-web-app-data
  Size:             5Gi
  Storage class:    standard
  Access mode:      ReadWriteOnce
```

## Step 7: Configure Published Services

Under **Ports**, click **+ Publish a new port**:

```text
Container port:    80
Service type:      NodePort (or LoadBalancer, ClusterIP)
Published port:    30080      (for NodePort)
```

For internal services (ClusterIP):

```text
Container port:    8080
Service type:      ClusterIP
```

## Step 8: Configure Health Checks (Optional but Recommended)

Under **Health checks**:

### Liveness Probe

```text
Type:         HTTP
Path:         /health
Port:         80
Initial delay: 30s
Period:       10s
Failure threshold: 3
```

### Readiness Probe

```text
Type:         HTTP
Path:         /ready
Port:         80
Initial delay: 10s
Period:       5s
```

## Step 9: Configure Auto-Scaling (Optional)

```text
Auto-scaling:    Enabled
Minimum replicas: 2
Maximum replicas: 10

CPU threshold:   70%     (scale up when avg CPU > 70%)
Memory threshold: 80%   (scale up when avg memory > 80%)
```

This creates a HorizontalPodAutoscaler (HPA) automatically.

## Step 10: Configure Labels and Annotations

```text
Labels:
  app: my-web-app
  environment: production
  version: v2.0

Annotations:
  deployment.kubernetes.io/revision: "1"
  description: "Main web application"
```

## Step 11: Deploy the Application

1. Review all settings
2. Click **Deploy the application**
3. Portainer creates the Kubernetes resources:
   - `Deployment`: manages the pods
   - `Service`: exposes the application
   - `PersistentVolumeClaim`: if storage was configured
   - `HorizontalPodAutoscaler`: if auto-scaling was configured

## Step 12: Verify the Deployment

After deploying:

1. Go to **Applications** to see the application listed
2. Click on it to see pod status and events
3. Check that all replicas are **Running**

```bash
# CLI verification

kubectl get deployment my-web-app -n production
kubectl get pods -n production -l app=my-web-app
kubectl get svc -n production

# Check events
kubectl describe deployment my-web-app -n production
```

## Viewing the Generated YAML

Portainer generates YAML behind the scenes. To see it:

1. Click on the application
2. Click the **YAML** or **Edit** tab
3. View the generated manifest

This is a great way to learn Kubernetes YAML by using the form first.

## Conclusion

Portainer's form-based application deployment makes Kubernetes accessible without requiring YAML knowledge. The form covers all essential configuration including replicas, resources, environment variables, storage, and services. As you become more comfortable with Kubernetes, use the generated YAML as a learning tool and transition to YAML-based deployments for more advanced configurations.
