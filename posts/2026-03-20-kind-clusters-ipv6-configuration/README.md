# How to Configure kind Clusters for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kind, Kubernetes, IPv6, Dual-Stack, Local Development, Testing

Description: A guide to creating kind (Kubernetes in Docker) clusters with IPv6 or dual-stack networking for local development and testing.

kind (Kubernetes in Docker) is the standard tool for running local Kubernetes clusters in Docker containers. It supports both IPv6-only and dual-stack cluster configurations through a simple YAML configuration file.

## Prerequisites

- Docker installed and running
- `kind` CLI installed (`go install sigs.k8s.io/kind@latest`)
- `kubectl` installed
- Docker daemon configured to support IPv6

## Step 1: Enable IPv6 in Docker Daemon

Docker must have IPv6 enabled before kind can use it.

```json
// /etc/docker/daemon.json - enable IPv6 in Docker
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/64",
  "experimental": true,
  "ip6tables": true
}
```

Restart Docker after editing:

```bash
sudo systemctl restart docker
```

## Step 2: Create a Dual-Stack kind Cluster Config

```yaml
# kind-dual-stack.yaml - kind cluster configuration for dual-stack

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  # Enable dual-stack
  ipFamily: dual
  # Define the pod and service CIDRs for both address families
  podSubnet: "10.244.0.0/16,fd00:10:244::/56"
  serviceSubnet: "10.96.0.0/16,fd00:10:96::/112"
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

## Step 3: Create an IPv6-Only kind Cluster Config

```yaml
# kind-ipv6-only.yaml - kind cluster configuration for IPv6-only
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  # IPv6-only cluster
  ipFamily: ipv6
  podSubnet: "fd00:10:244::/56"
  serviceSubnet: "fd00:10:96::/112"
nodes:
  - role: control-plane
  - role: worker
```

## Step 4: Create the Cluster

```bash
# Create the dual-stack cluster
kind create cluster --config kind-dual-stack.yaml --name ipv6-test

# Or create the IPv6-only cluster
kind create cluster --config kind-ipv6-only.yaml --name ipv6-only

# Set kubectl context
kubectl cluster-info --context kind-ipv6-test
```

## Step 5: Verify Dual-Stack Configuration

```bash
# Check node addresses - should include both IPv4 and IPv6
kubectl get nodes -o wide

# Check the kube-dns service for dual ClusterIPs
kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIPs}'

# Deploy a test pod and confirm dual IP assignment
kubectl run test --image=busybox:1.36 -- sleep 3600
kubectl get pod test -o jsonpath='{.status.podIPs}'
```

## Step 6: Test IPv6 Pod Connectivity

```bash
# Get the IPv6 address of the test pod
POD_IPV6=$(kubectl get pod test -o jsonpath='{.status.podIPs[1].ip}')

# From another pod, ping the IPv6 address
kubectl run pinger --image=busybox:1.36 --restart=Never -- ping6 -c 3 $POD_IPV6

# Check the result
kubectl logs pinger
```

## Step 7: Test IPv6 Service

```bash
# Deploy nginx with a dual-stack service
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80

# Check the service for IPv6 ClusterIP
kubectl get svc nginx -o jsonpath='{.spec.clusterIPs}'

# Curl the IPv6 ClusterIP from a pod
IPV6_SVC=$(kubectl get svc nginx -o jsonpath='{.spec.clusterIPs[1]}')
kubectl exec -it test -- wget -O- "http://[$IPV6_SVC]/"
```

## Cleanup

```bash
# Delete the kind cluster when done
kind delete cluster --name ipv6-test
```

kind's native dual-stack support makes it an excellent tool for testing IPv6 application behavior before deploying to production Kubernetes clusters.
