# How to Create ConfigMaps via YAML Manifest in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, ConfigMaps, YAML, Configuration Management

Description: Learn how to create Kubernetes ConfigMaps by writing or pasting YAML manifests directly in the Portainer UI.

## When to Use YAML for ConfigMaps

While the form interface is convenient for simple key-value pairs, YAML is better when:

- You need to paste a complete configuration file (multi-line strings).
- You have many keys and want to copy-paste from an existing file.
- You need precise control over the ConfigMap structure.
- You're migrating existing YAML from another cluster.

## Accessing the YAML Editor in Portainer

1. Select your Kubernetes environment.
2. Go to **ConfigMaps & Secrets**.
3. Click **Add ConfigMap**.
4. Toggle from **Form** to **Web editor (YAML)** mode.
5. Paste or type your ConfigMap YAML.
6. Click **Create ConfigMap**.

## ConfigMap YAML Examples

### Simple Key-Value ConfigMap

```yaml
# Simple application configuration

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: my-app
    managed-by: portainer
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DB_HOST: "postgres.production.svc.cluster.local"
  DB_PORT: "5432"
  REDIS_URL: "redis://cache:6379"
  MAX_WORKERS: "4"
```

### ConfigMap with Multi-Line File Content

```yaml
# ConfigMap containing a complete configuration file
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: production
data:
  # The key becomes the filename when mounted as a volume
  nginx.conf: |
    user nginx;
    worker_processes auto;

    events {
        worker_connections 1024;
    }

    http {
        upstream backend {
            server api:8080;
        }

        server {
            listen 80;

            location / {
                proxy_pass http://backend;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
            }
        }
    }
```

### ConfigMap with Binary Data

```yaml
# ConfigMap containing binary data (base64 encoded)
apiVersion: v1
kind: ConfigMap
metadata:
  name: binary-config
  namespace: default
data:
  text-key: "plain text value"
binaryData:
  # Binary values must be base64 encoded
  font.ttf: <base64-encoded-binary-data>
```

## Managing ConfigMaps via CLI

```bash
# Apply a ConfigMap from a YAML file
kubectl apply -f app-config.yaml

# Create a ConfigMap from a local file
kubectl create configmap nginx-config \
  --from-file=nginx.conf=./nginx.conf \
  --namespace=production

# Create from a directory (each file becomes a key)
kubectl create configmap app-configs \
  --from-file=./config-dir/ \
  --namespace=production

# View the ConfigMap
kubectl get configmap app-config -n production -o yaml
```

## Using the kubectl Shell in Portainer

For quick operations, use Portainer's built-in kubectl shell:

```bash
# In the Portainer kubectl shell, apply inline YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: quick-config
  namespace: default
data:
  key1: value1
  key2: value2
EOF
```

## Conclusion

YAML-based ConfigMap creation in Portainer provides full control over the ConfigMap structure, especially for complex configurations with multi-line file content. Use it alongside the form editor depending on the complexity of your configuration needs.
