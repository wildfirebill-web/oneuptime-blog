# How to Set Up Rancher HA on K3s - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, k3s, High Availability, Kubernetes, Lightweight, Edge

Description: Deploy Rancher in HA mode on K3s for resource-constrained environments using embedded etcd, minimal node requirements, and a simple load balancer.

## Introduction

K3s is a lightweight Kubernetes distribution ideal for edge, IoT, and resource-constrained environments. Running Rancher on K3s is appropriate when you need HA but want to minimize resource overhead. K3s's embedded etcd mode provides HA without a separate etcd cluster.

## Prerequisites

- 3 Linux nodes with at least 2 CPU / 4GB RAM each
- A load balancer or VIP for the K3s API endpoint
- `helm` and `kubectl`

## Step 1: Install K3s on the First Server

```bash
# On server-1 - Initialize the embedded etcd cluster

curl -sfL https://get.k3s.io | \
  K3S_TOKEN="mysupersecrettoken" \
  sh -s - server \
  --cluster-init \
  --tls-san rancher.example.com \
  --tls-san 10.0.0.10 \
  --disable traefik \    # Disable built-in Traefik (we'll use NGINX ingress)
  --node-taint "CriticalAddonsOnly=true:NoExecute"

# Get the server token
cat /var/lib/rancher/k3s/server/node-token
```

## Step 2: Join Additional Server Nodes

```bash
# On server-2 and server-3
curl -sfL https://get.k3s.io | \
  K3S_TOKEN="mysupersecrettoken" \
  sh -s - server \
  --server https://10.0.0.11:6443 \
  --tls-san rancher.example.com \
  --disable traefik \
  --node-taint "CriticalAddonsOnly=true:NoExecute"
```

## Step 3: Verify Cluster Health

```bash
# Check all K3s nodes
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes

# Verify etcd members
kubectl exec -it -n kube-system etcd-server-1 -- \
  etcdctl member list \
  --endpoints=https://localhost:2379 \
  --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/k3s/server/tls/etcd/client.key
```

## Step 4: Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

## Step 5: Install cert-manager and Rancher

```bash
# cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# Rancher
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.example.com \
  --set replicas=3 \
  --set ingress.ingressClassName=nginx
```

## Step 6: Add Worker Nodes

```bash
# Add worker nodes to the K3s cluster
curl -sfL https://get.k3s.io | \
  K3S_URL="https://rancher.example.com:6443" \
  K3S_TOKEN="mysupersecrettoken" \
  sh -
```

## Conclusion

Rancher HA on K3s provides a lightweight, resource-efficient management platform suitable for edge deployments or environments where full Kubernetes distributions are too heavy. K3s's embedded etcd eliminates the need for a separate etcd cluster while still providing HA with three server nodes.
