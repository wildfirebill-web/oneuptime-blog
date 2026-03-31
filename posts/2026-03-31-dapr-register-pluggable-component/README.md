# How to Register a Pluggable Component with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pluggable Component, Registration, Kubernetes, Component

Description: Learn how to register and configure a custom pluggable component with Dapr in both local development and Kubernetes production environments.

---

## Pluggable Component Registration Overview

Dapr discovers pluggable components through Unix domain socket files. At startup, the Dapr sidecar scans a configured socket directory and connects to each component process via gRPC. The component YAML maps a logical name to the component type (derived from the socket filename), so your application can reference it by name.

## Socket File Convention

The socket filename determines the component type name. A socket at `/tmp/dapr-components-sockets/my-state-store.sock` registers as type `state.my-state-store`.

## Local Development Registration

Set the socket folder when starting Dapr:

```bash
# Start your pluggable component process first
./my-state-store-component &

# Start Dapr with the socket folder configured
dapr run --app-id myapp \
  --app-port 8080 \
  --dapr-http-port 3500 \
  --components-path ./components \
  --config ./config.yaml \
  -- python app.py
```

Component YAML (`./components/my-state-store.yaml`):

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-state-store
  namespace: default
spec:
  type: state.my-state-store
  version: v1
  metadata:
  - name: socketFolder
    value: "/tmp/dapr-components-sockets"
  - name: connectionString
    secretKeyRef:
      name: store-secret
      key: connectionString
```

## Kubernetes Registration

In Kubernetes, the pluggable component runs as a sidecar container alongside the Dapr sidecar. Both containers share a volume for the socket file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      volumes:
      - name: dapr-unix-domain-socket
        emptyDir: {}

      containers:
      - name: myapp
        image: myorg/myapp:latest
        ports:
        - containerPort: 8080

      - name: my-state-store-component
        image: myorg/my-state-store:v1.0.0
        volumeMounts:
        - name: dapr-unix-domain-socket
          mountPath: /tmp/dapr-components-sockets

      initContainers:
      - name: wait-for-socket
        image: busybox
        command: ["sh", "-c", "until [ -S /tmp/dapr-components-sockets/my-state-store.sock ]; do sleep 1; done"]
        volumeMounts:
        - name: dapr-unix-domain-socket
          mountPath: /tmp/dapr-components-sockets
```

Annotate the pod for Dapr sidecar injection with the socket folder:

```yaml
metadata:
  annotations:
    dapr.io/enabled: "true"
    dapr.io/app-id: "myapp"
    dapr.io/app-port: "8080"
    dapr.io/unix-domain-socket-path: "/tmp/dapr-components-sockets"
```

## Verifying Registration

Check that Dapr recognized the component:

```bash
# Local
dapr components --app-id myapp

# Kubernetes
kubectl get components -n default
kubectl describe component my-state-store -n default
```

Expected output:

```bash
NAMESPACE  NAME            TYPE                 VERSION  SCOPES  CREATED
default    my-state-store  state.my-state-store  v1               2026-03-31 08:00:00
```

## Troubleshooting Socket Connection

If the component fails to register:

```bash
# Check component process is running and socket exists
ls -la /tmp/dapr-components-sockets/

# Check Dapr sidecar logs
dapr logs --app-id myapp | grep "pluggable"

# In Kubernetes
kubectl logs deployment/myapp -c daprd | grep "component"
```

## Summary

Registering a Dapr pluggable component requires starting the component process, ensuring its socket file appears in the configured socket folder, and providing a component YAML that maps a logical name to the socket-derived type. In Kubernetes, both the component process and Dapr sidecar share an emptyDir volume for the socket, with an init container to synchronize startup ordering. Verifying the component appears in `dapr components` output confirms successful registration.
