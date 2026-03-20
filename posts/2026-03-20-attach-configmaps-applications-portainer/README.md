# How to Attach ConfigMaps to Applications in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, ConfigMap, Configuration Management, DevOps

Description: Learn how to create ConfigMaps and attach them to Kubernetes applications as environment variables or mounted files in Portainer.

## What Are ConfigMaps?

ConfigMaps store non-sensitive configuration data as key-value pairs. They decouple configuration from container images, making it easy to change application settings without rebuilding images.

## Creating a ConfigMap in Portainer

1. Select your Kubernetes environment.
2. Go to **ConfigMaps** (or **Configs & Secrets > ConfigMaps**).
3. Click **Add ConfigMap**.
4. Enter a name, namespace, and key-value pairs.
5. Click **Create**.

## Attaching a ConfigMap to an Application

When deploying or editing an application in Portainer:

1. Scroll to the **Configuration** or **Environment variables** section.
2. Click **Add environment variable** or **Add configuration**.
3. Select **ConfigMap** as the source.
4. Choose the ConfigMap and key (or select all keys).

## Usage Pattern 1: Single Key from ConfigMap

```yaml
# Mount a single key from a ConfigMap as an env var
env:
  - name: DATABASE_URL
    valueFrom:
      configMapKeyRef:
        name: app-config        # ConfigMap name
        key: database_url       # Specific key in the ConfigMap
```

## Usage Pattern 2: All Keys from ConfigMap

```yaml
# Load all keys from a ConfigMap as environment variables
envFrom:
  - configMapRef:
      name: app-config          # All keys become env vars
```

## Usage Pattern 3: Mount ConfigMap as a File

ConfigMaps can also be mounted as files in the container filesystem. This is useful for config files like `nginx.conf` or `application.properties`:

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app/config    # Mount path in container
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config-files          # ConfigMap containing file data
```

## Creating a ConfigMap from a File

```bash
# Create a ConfigMap from a file (the filename becomes the key)
kubectl create configmap nginx-config \
  --from-file=nginx.conf=./nginx.conf \
  --namespace=production

# Create from multiple files
kubectl create configmap app-configs \
  --from-file=./config/               # All files in directory become keys
  --namespace=production
```

## Updating a ConfigMap

```bash
# Update a ConfigMap (pods must restart to pick up changes)
kubectl edit configmap app-config --namespace=production

# Or recreate it
kubectl create configmap app-config \
  --from-literal=key=newvalue \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Making Applications Auto-Reload ConfigMap Changes

By default, mounted ConfigMap files update automatically (within ~60 seconds), but environment variable injections require a pod restart:

```bash
# Force a rolling restart to pick up ConfigMap changes
kubectl rollout restart deployment/my-app --namespace=production
```

## Conclusion

ConfigMaps are the Kubernetes-native way to externalize application configuration. Portainer makes it easy to create ConfigMaps and attach them to applications through a UI rather than writing YAML by hand.
