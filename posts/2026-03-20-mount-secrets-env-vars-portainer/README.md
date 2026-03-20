# How to Mount Secrets as Environment Variables in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Secrets, Environment Variables, Security

Description: Learn how to inject Kubernetes Secrets into container pods as environment variables using Portainer.

## Why Use Secrets for Environment Variables?

While you could put sensitive values directly into a pod spec or ConfigMap, Kubernetes Secrets provide:

- Separation of sensitive data from application configuration.
- RBAC-based access control (only authorized subjects can read Secrets).
- Optional encryption at rest when etcd encryption is enabled.
- Audit logging for Secret access.

## Method 1: Inject All Secret Keys as Environment Variables

```yaml
spec:
  containers:
    - name: app
      image: my-app:latest
      envFrom:
        # Inject ALL keys from the secret as environment variables
        - secretRef:
            name: app-secrets
```

All keys in `app-secrets` (e.g., `DB_PASSWORD`, `API_KEY`) become environment variables in the container.

## Method 2: Inject Specific Keys from a Secret

```yaml
spec:
  containers:
    - name: app
      image: my-app:latest
      env:
        # Inject DB_PASSWORD from the db-credentials secret
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials    # Secret name
              key: password           # Key within the secret
              optional: false         # Pod fails to start if key is missing

        # Inject API key from a different secret
        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: payment-secrets
              key: stripe_api_key
```

## Configuring in Portainer

When creating or editing an application:

1. Scroll to **Environment variables**.
2. Click **Add environment variable**.
3. Choose **From Secret** as the type.
4. Select the Secret from the dropdown.
5. Choose the key you want to inject (or all keys).
6. Set the environment variable name.

## Combining Secrets and ConfigMaps

A typical application uses both:

```yaml
spec:
  containers:
    - name: app
      image: my-app:latest
      # Load all non-sensitive config
      envFrom:
        - configMapRef:
            name: app-config
      # Load all sensitive credentials
        - secretRef:
            name: app-secrets
      # Plus some specific overrides
      env:
        - name: ADMIN_TOKEN
          valueFrom:
            secretKeyRef:
              name: admin-secrets
              key: token
```

## Verifying Secret Injection

```bash
# Confirm environment variables are set (never log actual values in production)
kubectl exec -it <pod-name> --namespace=production \
  -- env | grep -E "PASSWORD|API_KEY|TOKEN" | cut -d= -f1
# Output: only the variable names, not values

# Check pod events if secret injection fails
kubectl describe pod <pod-name> --namespace=production | grep -A5 Events
```

## Security Best Practices

- Avoid printing secret environment variable values in logs.
- Set `optional: false` on required secrets so the pod fails fast rather than running misconfigured.
- Use namespaced Secrets and restrict access with RBAC.

```bash
# Restrict who can read secrets in a namespace
kubectl create role secret-reader \
  --verb=get,list --resource=secrets \
  --namespace=production
```

## Conclusion

Injecting Secrets as environment variables is a clean pattern for supplying sensitive configuration to containers. Portainer's form interface makes it easy without needing to write YAML, and the automatic encoding ensures values are stored correctly.
