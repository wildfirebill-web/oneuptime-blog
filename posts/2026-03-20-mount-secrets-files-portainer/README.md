# How to Mount Secrets as Files in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Secrets, Volume Mounts, Security

Description: Learn how to mount Kubernetes Secrets as files inside container pods through Portainer for certificate and key-based configurations.

## When to Mount Secrets as Files

Some applications require secrets as filesystem files rather than environment variables:

- TLS certificates and private keys (web servers, gRPC services).
- SSH private keys (Git deployment keys, remote access).
- Service account JSON keys (Google Cloud, AWS credentials).
- Configuration files containing embedded secrets.

## How Secret File Mounting Works

Each key in the Secret becomes a file. The Secret's base64-encoded values are decoded and written as plain file content.

## Example: Mounting TLS Certificates

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: tls-certs
          mountPath: /etc/ssl/app   # Directory in the container
          readOnly: true            # Always read-only for security
  volumes:
    - name: tls-certs
      secret:
        secretName: my-tls-secret  # Secret containing tls.crt and tls.key
```

After mounting, the container will have:
- `/etc/ssl/app/tls.crt`
- `/etc/ssl/app/tls.key`

## Example: Mounting a Specific Secret Key as a File

```yaml
volumes:
  - name: gcp-credentials
    secret:
      secretName: gcp-service-account
      items:
        - key: service-account.json     # Key in the Secret
          path: application_credentials.json  # Filename in the container
          mode: 0400                   # File permissions (owner read-only)
```

## Creating a TLS Secret for Mounting

```bash
# Create TLS secret from certificate files

kubectl create secret tls my-tls-secret \
  --cert=fullchain.pem \
  --key=privkey.pem \
  --namespace=production

# Create a generic secret from files
kubectl create secret generic ssh-key \
  --from-file=id_rsa=/home/user/.ssh/id_rsa \
  --namespace=production
```

## Configuring in Portainer

When deploying or editing an application:

1. Scroll to **Volumes** or **Persistent storage**.
2. Click **Add volume** and select **Secret** as the type.
3. Choose the Secret.
4. Enter the mount path in the container.
5. Optionally specify individual keys to expose as specific filenames.

## Setting File Permissions

```yaml
volumes:
  - name: ssh-keys
    secret:
      secretName: ssh-credentials
      defaultMode: 0400    # All files default to owner-readable only
      items:
        - key: private_key
          path: id_rsa
          mode: 0400       # Override mode for specific files
```

## Verifying the Mount

```bash
# Check files are present and have correct permissions
kubectl exec -it <pod-name> --namespace=production \
  -- ls -la /etc/ssl/app/

# Verify the certificate is valid (without printing it)
kubectl exec -it <pod-name> --namespace=production \
  -- openssl x509 -in /etc/ssl/app/tls.crt -noout -subject -dates
```

## Automatic Updates

Like ConfigMaps, mounted Secret files update automatically when the Secret is changed (within the kubelet sync period). This enables certificate rotation without pod restarts.

## Conclusion

Mounting Secrets as files is essential for TLS certificates, SSH keys, and credential files. Portainer simplifies the volume configuration so you don't need to write the volume and volumeMount YAML manually.
