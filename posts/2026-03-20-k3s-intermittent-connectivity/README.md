# How to Configure K3s for Intermittent Connectivity

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Edge Computing, Offline, Resilience, DevOps

Description: Learn how to configure K3s to operate reliably in environments with intermittent internet connectivity, including offline-first workload design and sync strategies.

## Introduction

Many edge deployments operate in environments with unreliable connectivity — remote sites connected via cellular links, satellite internet, or low-bandwidth connections that frequently go offline. K3s is excellent in these scenarios because it continues running autonomously when disconnected, but workloads and configuration need to be designed with intermittent connectivity in mind. This guide covers strategies for resilient K3s deployments in connectivity-challenged environments.

## Key Challenges with Intermittent Connectivity

1. **Image pulls fail** when the registry is unreachable
2. **Helm chart updates** can't be applied without connectivity
3. **Certificate renewal** requires connectivity to some services
4. **Central monitoring** loses visibility during outages
5. **Configuration synchronization** must handle offline periods gracefully

## Step 1: Pre-Pull Container Images

Ensure all required images are cached locally before deployment:

```bash
#!/bin/bash
# pre-pull-images.sh
# Run this script during initial setup or maintenance windows

IMAGES=(
  "nginx:1.25-alpine"
  "redis:7-alpine"
  "influxdb:2.7-alpine"
  "eclipse-mosquitto:2.0"
  "nodered/node-red:3.1"
  "myregistry.local/my-app:v2.1"
)

for IMAGE in "${IMAGES[@]}"; do
  echo "Pulling: $IMAGE"
  k3s crictl pull "$IMAGE"
done

echo "All images cached locally"

# Verify cached images
k3s crictl images
```

Set `imagePullPolicy: IfNotPresent` in all deployments:

```yaml
spec:
  containers:
    - name: my-app
      image: nginx:1.25-alpine
      # Never try to pull - use cached image
      imagePullPolicy: IfNotPresent
```

## Step 2: Deploy a Local Container Registry

Host a local OCI registry at the edge site to serve images offline:

```yaml
# local-registry.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: local-registry
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: local-registry
  template:
    metadata:
      labels:
        app: local-registry
    spec:
      containers:
        - name: registry
          image: registry:2
          ports:
            - containerPort: 5000
          volumeMounts:
            - name: registry-data
              mountPath: /var/lib/registry
          resources:
            limits:
              memory: 256Mi
      volumes:
        - name: registry-data
          hostPath:
            path: /data/registry
            type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  name: local-registry-svc
  namespace: kube-system
spec:
  selector:
    app: local-registry
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30500
  type: NodePort
```

Configure K3s to use the local registry:

```yaml
# /etc/rancher/k3s/registries.yaml
mirrors:
  # Mirror Docker Hub through local registry
  "docker.io":
    endpoint:
      - "http://localhost:30500"
  # Mirror your private registry
  "myregistry.example.com":
    endpoint:
      - "http://localhost:30500"
```

## Step 3: Implement Offline-First Workload Design

Design applications to function without external connectivity:

```yaml
# offline-first-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: offline-first-app
spec:
  template:
    spec:
      containers:
        - name: app
          image: localhost:30500/my-app:v2.1
          imagePullPolicy: IfNotPresent
          env:
            # Use local services when available, fallback to cache
            - name: DATABASE_HOST
              value: "local-postgres.default.svc.cluster.local"
            - name: CACHE_HOST
              value: "local-redis.default.svc.cluster.local"
            # Feature flag for offline mode behavior
            - name: OFFLINE_CAPABLE
              value: "true"
            - name: SYNC_QUEUE_ENABLED
              value: "true"
            # Queue outbound requests when offline
            - name: MESSAGE_QUEUE_URL
              value: "amqp://local-rabbitmq.default.svc.cluster.local"
          livenessProbe:
            httpGet:
              path: /health/local  # Local health check, no network dependency
              port: 8080
            periodSeconds: 30
            failureThreshold: 5
```

## Step 4: Use a Local Message Queue for Reliable Delivery

Buffer outbound data when connectivity is lost:

```yaml
# local-rabbitmq.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: local-rabbitmq
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: rabbitmq
          image: rabbitmq:3.12-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5672
          env:
            - name: RABBITMQ_DEFAULT_USER
              value: edge-user
            - name: RABBITMQ_DEFAULT_PASS
              value: edgepassword
          resources:
            limits:
              memory: 256Mi
          volumeMounts:
            - name: rabbitmq-data
              mountPath: /var/lib/rabbitmq
      volumes:
        - name: rabbitmq-data
          hostPath:
            path: /data/rabbitmq
```

## Step 5: Configure Sync Agent for Intermittent Connectivity

Deploy a sync agent that buffers and replays data:

```yaml
# sync-agent.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: connectivity-sync-agent
spec:
  selector:
    matchLabels:
      app: sync-agent
  template:
    metadata:
      labels:
        app: sync-agent
    spec:
      containers:
        - name: sync-agent
          image: localhost:30500/sync-agent:v1
          imagePullPolicy: IfNotPresent
          env:
            # Central HQ endpoint
            - name: HQ_ENDPOINT
              value: "https://hq.example.com/api/edge-sync"
            # Retry configuration for intermittent connectivity
            - name: RETRY_INTERVAL_SECONDS
              value: "30"
            - name: MAX_RETRY_COUNT
              value: "10"
            - name: OFFLINE_BUFFER_SIZE_MB
              value: "100"
            # Local queue to buffer data while offline
            - name: LOCAL_QUEUE_URL
              value: "amqp://local-rabbitmq:5672"
          resources:
            limits:
              memory: 128Mi
```

## Step 6: Configure K3s Timeouts for Slow Networks

```yaml
# /etc/rancher/k3s/config.yaml
# Increase timeouts for slow/intermittent connections

kubelet-arg:
  # Increase node lease duration (how long before node is considered lost)
  - "node-lease-duration-seconds=60"
  # Wait longer before evicting pods on unreachable nodes
  - "node-monitor-grace-period=60s"

kube-controller-manager-arg:
  # Longer pod eviction grace period for intermittent connections
  - "node-monitor-grace-period=60s"
  - "pod-eviction-timeout=300s"
```

## Step 7: Local DNS with Offline Fallback

```yaml
# /var/lib/rancher/k3s/server/manifests/coredns-offline.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        # Cache DNS longer for offline resilience
        cache 3600
        # Fallback to local hosts file if upstream is unreachable
        hosts {
            # Local service overrides for offline operation
            192.168.1.10 hq.example.com
            fallthrough
        }
        forward . /etc/resolv.conf {
            max_concurrent 1000
            # Use local resolver with short timeout
            expire 10s
            health_check 5s
        }
        loop
        reload
        loadbalance
    }
```

## Step 8: Detect and Handle Connectivity State

```bash
#!/bin/bash
# /usr/local/bin/connectivity-check.sh
# Run via cron to trigger offline/online mode transitions

HQ_ENDPOINT="https://hq.example.com/healthz"
CONNECTIVITY_STATE_FILE="/tmp/k3s-connectivity-state"

# Check connectivity
if curl -s --connect-timeout 10 "$HQ_ENDPOINT" > /dev/null 2>&1; then
  CURRENT_STATE="online"
else
  CURRENT_STATE="offline"
fi

# Read previous state
PREVIOUS_STATE=$(cat "$CONNECTIVITY_STATE_FILE" 2>/dev/null || echo "unknown")

# Only act on state changes
if [ "$CURRENT_STATE" != "$PREVIOUS_STATE" ]; then
  echo "$CURRENT_STATE" > "$CONNECTIVITY_STATE_FILE"
  logger "Connectivity state changed: $PREVIOUS_STATE -> $CURRENT_STATE"

  if [ "$CURRENT_STATE" = "online" ]; then
    # Trigger sync when connectivity is restored
    kubectl annotate pods -n iot -l sync-enabled=true \
      connectivity-restored="$(date -Iseconds)" --overwrite
    logger "Triggered sync for IoT pods"
  fi
fi
```

## Conclusion

K3s deployments in intermittent connectivity environments require a combination of local image caching, offline-first application design, local message queuing for data buffering, and appropriate Kubernetes timeout tuning. The core principle is that all critical workloads should continue functioning locally without any external connectivity, with data being queued for synchronization when connectivity is restored. This design ensures the edge site remains operational regardless of the quality of its uplink.
