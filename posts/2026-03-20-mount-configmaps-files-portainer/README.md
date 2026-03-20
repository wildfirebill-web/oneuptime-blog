# How to Mount ConfigMaps as Files in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, ConfigMap, Volume Mounts, Configuration

Description: Learn how to mount Kubernetes ConfigMaps as files inside container pods using Portainer.

## When to Mount ConfigMaps as Files

Some applications expect configuration as files rather than environment variables:

- Web servers (Nginx, Apache) — need `nginx.conf`, `.htaccess`.
- Application frameworks — need `application.properties`, `config.yaml`.
- Scripts and tools — need config files in specific filesystem paths.

## How File Mounting Works

Each key in the ConfigMap becomes a filename, and its value becomes the file content.

## Example ConfigMap with File Content

```yaml
# ConfigMap containing an Nginx configuration file
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: production
data:
  # Key = filename, value = file content
  nginx.conf: |
    server {
        listen 80;
        server_name _;

        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location /health {
            return 200 'ok';
            add_header Content-Type text/plain;
        }
    }
  # Another file in the same ConfigMap
  mime.types: |
    text/html                             html htm;
    text/css                              css;
    application/javascript                js;
```

## Mounting the ConfigMap in a Pod

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: nginx-config-volume
          # Directory where files will be created
          mountPath: /etc/nginx/conf.d/
          readOnly: true
  volumes:
    - name: nginx-config-volume
      configMap:
        name: nginx-config         # The ConfigMap to mount
        # Optional: specify which keys to expose as files
        items:
          - key: nginx.conf        # ConfigMap key
            path: default.conf     # Filename inside the container
```

## Configuring in Portainer

When deploying an application in Portainer:

1. Scroll to **Volumes** or **Persistent storage**.
2. Click **Add volume** or **Add config mount**.
3. Select **ConfigMap** as the volume type.
4. Choose the ConfigMap.
5. Enter the **mount path** in the container.
6. Optionally specify which keys to include.

## Mounting a Single File (Specific Key)

To mount only one file from a ConfigMap:

```yaml
volumes:
  - name: app-config
    configMap:
      name: app-settings
      items:
        - key: application.yaml    # Only this key
          path: application.yaml   # Mount as this filename
```

## Verifying the Mount

```bash
# Check that the file exists inside the container
kubectl exec -it <pod-name> --namespace=production \
  -- cat /etc/nginx/conf.d/default.conf

# List all mounted files
kubectl exec -it <pod-name> --namespace=production \
  -- ls -la /etc/nginx/conf.d/
```

## Automatic File Updates

Unlike environment variables, mounted ConfigMap files update automatically when the ConfigMap is changed (within ~60 seconds kubelet sync period). This enables runtime configuration updates without pod restarts for applications that watch their config files.

```bash
# Update the ConfigMap
kubectl edit configmap nginx-config --namespace=production

# Watch for the file to update in the pod (no restart needed)
kubectl exec -it <pod-name> -- watch cat /etc/nginx/conf.d/default.conf
```

## Conclusion

Mounting ConfigMaps as files is the right approach for applications that expect filesystem-based configuration. The automatic file update feature makes it powerful for configuration changes that don't require process restarts.
