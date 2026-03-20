# How to Create ConfigMaps via Form in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, ConfigMap, Configuration Management, DevOps

Description: Learn how to create Kubernetes ConfigMaps using Portainer's form interface without writing YAML.

## What Is a ConfigMap?

A ConfigMap stores non-sensitive configuration data as key-value pairs or file content. Pods can consume ConfigMaps as environment variables or mounted files. Portainer's form interface makes creating ConfigMaps quick and intuitive.

## Creating a ConfigMap via Form in Portainer

1. Select your Kubernetes environment.
2. In the sidebar, go to **ConfigMaps & Secrets** or **Configurations**.
3. Click **Add ConfigMap**.
4. Fill in:
   - **Name**: A descriptive name (e.g., `app-config`, `nginx-config`).
   - **Namespace**: Select the target namespace.
5. Add entries:
   - For key-value pairs, enter the key and value in the form fields.
   - For file content, enter the filename as the key and the file contents as the value.
6. Click **Create ConfigMap**.

## Form Entry Types

### Simple Key-Value Pairs

| Key | Value |
|-----|-------|
| `APP_ENV` | `production` |
| `LOG_LEVEL` | `info` |
| `MAX_CONNECTIONS` | `100` |
| `FEATURE_FLAG_NEW_UI` | `true` |

### File Content as ConfigMap Entry

When your application expects a configuration file:

| Key | Value |
|-----|-------|
| `nginx.conf` | *(paste entire nginx.conf content)* |
| `app.properties` | *(paste properties file content)* |

## What Portainer Creates

The form generates the following Kubernetes resource:

```yaml
# ConfigMap created by Portainer form
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }
```

## After Creating the ConfigMap

Reference it in your application deployments by going to **Applications > Edit** and selecting the ConfigMap in the environment variable or volume section.

## Verifying the ConfigMap via CLI

```bash
# List ConfigMaps in a namespace
kubectl get configmaps --namespace=production

# View the ConfigMap's content
kubectl describe configmap app-config --namespace=production

# Get the raw YAML
kubectl get configmap app-config --namespace=production -o yaml
```

## Editing an Existing ConfigMap

1. In Portainer, go to **ConfigMaps** and click on the ConfigMap.
2. Edit the key-value pairs.
3. Click **Update ConfigMap**.

```bash
# Edit via CLI
kubectl edit configmap app-config --namespace=production
```

Note: Pods that consume ConfigMaps as environment variables need a restart to pick up changes. Mounted ConfigMap files update automatically within ~60 seconds.

## Conclusion

Portainer's ConfigMap form editor eliminates the need to write YAML for common configuration data. It's particularly useful for teams that need to update application configuration frequently without deploying new container images.
