# How to Deploy a Kubernetes Application via YAML Manifest in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, YAML, Manifest Deployment, DevOps

Description: Learn how to deploy Kubernetes applications by applying YAML manifests directly through the Portainer UI.

## Overview

For users comfortable with Kubernetes YAML, Portainer provides a manifest editor that lets you apply raw YAML files to your cluster. This approach gives you full control over the Kubernetes resource specification.

## Deploying via Manifest in Portainer

1. Select your Kubernetes environment.
2. Go to **Applications** and click **Add application**.
3. Select **Advanced deployment** (or **Manifest** depending on Portainer version).
4. Toggle to **Web editor** mode.
5. Paste your manifest YAML.
6. Click **Deploy**.

Alternatively, use the **Manifests** section in the sidebar if available, or go to **kubectl** shell.

## Example: Full Application Manifest

```yaml
# Complete application manifest with Deployment, Service, and ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    version: "1.5.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: registry.mycompany.com/my-app:1.5.0
          ports:
            - containerPort: 8080
          envFrom:
            # Load all keys from ConfigMap as environment variables
            - configMapRef:
                name: app-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

## Multi-Document Manifests

Portainer accepts multi-document YAML files (separated by `---`). This allows deploying an entire application stack in one operation.

## Applying from a Git Repository

Instead of pasting YAML, you can link to a Git repository in Portainer:

1. Go to **Applications > Add application > Advanced deployment**.
2. Select **Git repository**.
3. Enter the repository URL and file path.
4. Portainer fetches and applies the manifest.

## Using the Built-in kubectl Shell

For power users, Portainer provides a kubectl shell:

```bash
# In Portainer: Go to your environment > kubectl shell

# Apply a manifest from within the shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: default
spec:
  containers:
    - name: test
      image: nginx:alpine
EOF
```

## Conclusion

YAML manifest deployment in Portainer bridges the gap between CLI-based kubectl workflows and visual management. You get the precision of raw YAML with the convenience of a browser-based interface and team-level audit logging.
