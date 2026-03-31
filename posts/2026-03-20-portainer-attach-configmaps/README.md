# How to Attach ConfigMaps to Applications in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, ConfigMap, Configuration, DevOps

Description: Learn how to create ConfigMaps and attach them to Kubernetes applications as environment variables or file volumes in Portainer.

## Introduction

ConfigMaps allow you to decouple configuration from container images, making applications portable across environments. Portainer provides UI tools for creating ConfigMaps and attaching them to applications as environment variables or mounted files. This guide covers the complete workflow.

## Prerequisites

- Portainer with Kubernetes environment
- Target namespace for the ConfigMap

## Step 1: Create a ConfigMap in Portainer

### Via Form

1. Select your Kubernetes environment
2. Navigate to **ConfigMaps** in the sidebar
3. Click **+ Add ConfigMap**
4. Fill in:

```text
Name:        app-config
Namespace:   production
```

5. Add key-value pairs:

```text
Key: NODE_ENV          Value: production
Key: LOG_LEVEL         Value: info
Key: DB_HOST           Value: postgres.production.svc.cluster.local
Key: MAX_POOL_SIZE     Value: 20
Key: CACHE_ENABLED     Value: "true"
```

6. Click **Create ConfigMap**

### Via YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: production
data:
  # Simple key-value pairs
  upstream_host: "backend.production.svc.cluster.local"
  upstream_port: "8080"

  # Multi-line values (useful for config files)
  nginx.conf: |
    upstream backend {
        server backend.production.svc.cluster.local:8080;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
        }
    }

  # JSON configuration
  app-settings.json: |
    {
      "featureFlags": {
        "newUI": true,
        "darkMode": false
      },
      "pagination": {
        "defaultPageSize": 20,
        "maxPageSize": 100
      }
    }
```

## Step 2: Attach ConfigMap as Environment Variables

### Via Portainer Form

When creating/editing an application:

1. Go to the **Environment** section
2. Click **+ From ConfigMap**
3. Select:

```sql
ConfigMap: app-config
  [x] Import all keys as environment variables

  Or select individual keys:
  Key: NODE_ENV    → Environment variable name: NODE_ENV
  Key: LOG_LEVEL   → Environment variable name: LOG_LEVEL
```

### Via YAML

```yaml
spec:
  containers:
    - name: app
      image: myapp:latest
      # Load all keys from ConfigMap as env vars
      envFrom:
        - configMapRef:
            name: app-config

      # Or load specific keys
      env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: NODE_ENV
```

## Step 3: Attach ConfigMap as a Volume (File Mount)

This is ideal for configuration files that need to be present on the filesystem.

### Via Portainer Form

In the application editor under **Volumes**:

```text
Volume type:    ConfigMap
ConfigMap:      nginx-config
Mount path:     /etc/nginx/conf.d
```

### Via YAML

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d    # Mount all keys as files
          readOnly: true
        - name: app-settings
          mountPath: /app/config/settings.json  # Mount specific key as file
          subPath: app-settings.json           # Which key to mount
          readOnly: true

  volumes:
    - name: nginx-config
      configMap:
        name: nginx-config   # All keys become files in the directory
        defaultMode: 0444    # Read-only for all users

    - name: app-settings
      configMap:
        name: nginx-config
        items:
          - key: app-settings.json    # Mount only this key
            path: settings.json       # As this filename
```

## Step 4: Verify ConfigMap Attachment

```bash
# Check if ConfigMap exists

kubectl get configmap app-config -n production

# View ConfigMap contents
kubectl describe configmap app-config -n production

# Verify inside a running pod
kubectl exec -it <pod-name> -n production -- env | grep NODE_ENV
kubectl exec -it <pod-name> -n production -- cat /etc/nginx/conf.d/nginx.conf
```

## Step 5: Update a ConfigMap

```bash
# Edit directly
kubectl edit configmap app-config -n production

# Apply updated YAML
kubectl apply -f app-config.yaml

# Or in Portainer: navigate to ConfigMaps, click Edit, save
```

**Important:** Updating a ConfigMap does NOT automatically restart pods. Pods see the changes in mounted volumes after a short delay (~1-2 minutes), but environment variables from ConfigMaps require a pod restart.

```bash
# Restart pods to pick up env var changes
kubectl rollout restart deployment myapp -n production
```

## Step 6: ConfigMap Best Practices

```yaml
# Good: Separate ConfigMaps by environment
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-production
data:
  NODE_ENV: production
  LOG_LEVEL: error
  DB_HOST: postgres-prod.svc.cluster.local

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-staging
data:
  NODE_ENV: staging
  LOG_LEVEL: debug
  DB_HOST: postgres-staging.svc.cluster.local
```

## Conclusion

ConfigMaps in Portainer provide a clean, Kubernetes-native way to manage application configuration. Use environment variable references for simple key-value configuration and volume mounts for configuration files. Remember that ConfigMap changes are not automatically picked up by running pods for environment variables - plan for pod restarts when updating critical configuration.
