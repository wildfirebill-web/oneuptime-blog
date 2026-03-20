# How to Configure Init Containers in Kubernetes via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Init Containers, Docker, Pods, Infrastructure

Description: Configure Kubernetes init containers through Portainer to perform setup tasks like database migrations, configuration downloads, or dependency checks before main application containers start.

---

Kubernetes init containers run and complete before the main application containers in a Pod start. They are ideal for one-time setup tasks: running database migrations, fetching configuration from a remote source, waiting for dependent services, or setting file permissions. Portainer's manifest editor makes it easy to define init containers alongside your application pods.

## Common Init Container Patterns

| Pattern | Use Case |
|---|---|
| Database migration | Run `alembic upgrade head` before the API starts |
| Dependency wait | Poll until a database or service is ready |
| Config fetch | Download configuration from Vault or S3 |
| Permission setup | `chown` shared volumes before main container runs |
| Clone repository | Pull latest configuration into a shared volume |

## Step 1: Wait for Database Readiness

Deploy a pod with an init container that waits for PostgreSQL before starting the API:

```yaml
# api-with-init.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      initContainers:
        - name: wait-for-postgres
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # Poll until PostgreSQL responds on port 5432
              until nc -z postgres-service 5432; do
                echo "Waiting for PostgreSQL..."
                sleep 2
              done
              echo "PostgreSQL is ready."
      containers:
        - name: api
          image: my-registry/api-server:1.5.0
          ports:
            - containerPort: 8080
```

## Step 2: Database Migration Init Container

Run migrations before the application starts:

```yaml
initContainers:
  - name: run-migrations
    image: my-registry/api-server:1.5.0
    command: ["python", "manage.py", "migrate", "--noinput"]
    env:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: db-credentials
            key: url
```

The main container starts only after migrations complete successfully.

## Step 3: Fetch Configuration from Vault

```yaml
initContainers:
  - name: vault-config-fetch
    image: vault:1.15
    command:
      - sh
      - -c
      - |
        vault login -method=kubernetes role=api-role
        vault kv get -field=config secret/api-config > /config/app.json
    volumeMounts:
      - name: config-volume
        mountPath: /config
    env:
      - name: VAULT_ADDR
        value: "https://vault.internal:8200"

containers:
  - name: api
    volumeMounts:
      - name: config-volume
        mountPath: /app/config

volumes:
  - name: config-volume
    emptyDir: {}
```

The init container writes to a shared `emptyDir` volume that the main container reads at startup.

## Step 4: Deploy via Portainer

Paste the complete YAML into Portainer's **Kubernetes > Advanced Deployment** interface and click **Deploy**. Portainer shows the pod's initialization phase — init containers appear in the pod detail with their own status and logs.

## Step 5: View Init Container Logs

In Portainer, navigate to the pod and use the **Logs** view with the container selector to switch between init containers and the main container:

```bash
# Via terminal — view init container logs
kubectl logs <pod-name> -c wait-for-postgres -n production
kubectl logs <pod-name> -c run-migrations -n production
```

## Summary

Init containers in Kubernetes provide a structured way to handle application startup dependencies and one-time setup tasks. Portainer's manifest interface makes it easy to add init containers to existing deployment definitions, and the pod detail view lets you monitor and debug each init container independently.
