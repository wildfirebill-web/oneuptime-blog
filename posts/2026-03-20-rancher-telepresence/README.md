# How to Use Telepresence with Rancher Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Telepresence, Development, Inner Loop

Description: Use Telepresence with Rancher to run microservices locally while connecting them to your remote Kubernetes cluster for realistic development and debugging.

## Introduction

Telepresence allows you to run a single service locally while connecting it to a remote Rancher Kubernetes cluster. Your local service can make calls to and receive calls from other services in the cluster as if it were running there. This provides a realistic development environment without needing to build and push container images for every code change.

## Prerequisites

- Telepresence CLI installed
- kubectl configured for your Rancher cluster
- Cluster admin access (for initial setup)
- A local development environment

## Step 1: Install Telepresence

```bash
# macOS
brew install datawire/blackbird/telepresence

# Linux
sudo curl -fL https://app.getambassador.io/download/tel2/linux/amd64/latest/telepresence \
  -o /usr/local/bin/telepresence
sudo chmod +x /usr/local/bin/telepresence

# Verify
telepresence version
```

## Step 2: Connect to Your Rancher Cluster

```bash
# Connect Telepresence to your cluster
# This installs the Traffic Manager in the cluster
telepresence connect

# Verify connection
telepresence status

# List intercept-able services
telepresence list --namespace production
```

## Step 3: Intercept a Service

```bash
# Full intercept: all traffic to the service goes to your local process
telepresence intercept my-service \
  --namespace production \
  --port 3000:3000

# Now run your service locally
python src/main.py
# or
node server.js
# or
go run main.go

# Your local service now receives all traffic destined for my-service in the cluster
```

## Step 4: Personal Intercepts (Multi-Developer)

Personal intercepts only route traffic with specific headers to your local service:

```bash
# Intercept only requests with a specific header
telepresence intercept my-service \
  --namespace production \
  --port 3000:3000 \
  --http-header x-developer=alice

# Now, requests with the header 'x-developer: alice' go to your local service
# All other requests continue to the in-cluster service

# Test your intercept
curl -H "x-developer: alice" http://my-service.production.svc.cluster.local:3000/api/orders
```

## Step 5: Access Cluster Services from Local Machine

Once connected, you can access cluster services directly:

```bash
# Access any service in the cluster from your local machine
curl http://postgresql.databases.svc.cluster.local:5432
curl http://redis.databases.svc.cluster.local:6379
curl http://kafka.kafka.svc.cluster.local:9092

# Use cluster DNS
nslookup my-service.production.svc.cluster.local
```

## Step 6: Configure Environment Variables for Local Service

```bash
# Get environment variables from a running pod
telepresence intercept my-service \
  --namespace production \
  --port 3000 \
  --env-file /tmp/my-service.env

# Load the environment and start your service
source /tmp/my-service.env
node server.js

# Or use docker with the env file
docker run --rm \
  --env-file /tmp/my-service.env \
  -p 3000:3000 \
  my-app:dev
```

## Step 7: Mount Cluster Volumes Locally

```bash
# Mount config volumes from the pod to your local filesystem
telepresence intercept my-service \
  --namespace production \
  --port 3000 \
  --mount /tmp/cluster-volumes

# Access cluster ConfigMaps and secrets locally
ls /tmp/cluster-volumes/run/secrets/
cat /tmp/cluster-volumes/etc/config/app-config.yaml
```

## Step 8: Debug with a Remote Shell

```bash
# Get a shell in the cluster with Telepresence networking
telepresence connect
telepresence helm install --namespace production

# Or use a debug pod
kubectl run debug-pod --rm -it \
  --namespace production \
  --image=python:3.11 \
  -- bash

# From inside the debug pod, you can reach all cluster services
curl http://my-service:3000/api/health
curl http://postgresql.databases:5432
```

## Step 9: Disconnect and Cleanup

```bash
# Leave the intercept (restore original service)
telepresence leave my-service-production

# Disconnect from the cluster
telepresence disconnect

# Uninstall Traffic Manager from cluster (optional)
telepresence helm uninstall
```

## Step 10: Configure Telepresence for Team Use

```yaml
# telepresence.yaml - Team configuration
cloud:
  skipLogin: true

intercept:
  defaultPort: 8080

# Exclude high-churn namespaces from watching
# (improves performance in large clusters)
excludeNamespaces:
  - kube-system
  - cattle-system

# Configure grpc limits
grpc:
  maxReceiveSize: 256Mi
```

## Troubleshooting

```bash
# Check Telepresence Traffic Manager status
kubectl get pods -n ambassador \
  -l app=traffic-manager

# Check Telepresence logs
telepresence loglevel debug
kubectl logs -n ambassador deployment/traffic-manager

# Reset stuck connection
telepresence quit -s  # Quit daemon
telepresence connect   # Reconnect

# Check intercept status
telepresence list -n production
```

## Conclusion

Telepresence revolutionizes the Kubernetes development workflow by eliminating the build-push-deploy cycle for iterative development. The ability to run a service locally while fully integrated with the remote Rancher cluster means you get instant code reloading while having realistic access to all cluster services, databases, and configuration. Personal intercepts are particularly valuable for teams, allowing multiple developers to test their changes simultaneously without interfering with each other.
