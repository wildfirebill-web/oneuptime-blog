# How to Install Dapr on Minikube

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Minikube, Kubernetes, Installation, Local Development

Description: Step-by-step guide to installing Dapr on a local Minikube cluster for development and testing of distributed microservice applications.

---

## Prerequisites

Before installing Dapr on Minikube, ensure you have the following tools installed:

```bash
# Check minikube version (1.28+ recommended)
minikube version

# Check kubectl
kubectl version --client

# Install Dapr CLI
wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash
dapr --version
```

## Start Minikube

Start Minikube with sufficient resources for Dapr control plane components:

```bash
minikube start \
  --cpus=4 \
  --memory=8192 \
  --kubernetes-version=v1.29.0 \
  --driver=docker

# Verify cluster is running
kubectl cluster-info
kubectl get nodes
```

## Install Dapr on Minikube

Use the Dapr CLI to initialize Dapr on the cluster:

```bash
dapr init --kubernetes --wait
```

The `--wait` flag blocks until all control plane pods are ready. To install a specific version:

```bash
dapr init --kubernetes --runtime-version 1.13.0 --wait
```

## Verify the Installation

```bash
# Check Dapr control plane status
dapr status -k

# Verify pods in dapr-system namespace
kubectl get pods -n dapr-system

# Expected output:
# NAME                                     READY   STATUS    RESTARTS   AGE
# dapr-dashboard-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
# dapr-operator-xxxxxxxxxx-xxxxx           1/1     Running   0          2m
# dapr-placement-server-0                  1/1     Running   0          2m
# dapr-sentry-xxxxxxxxxx-xxxxx             1/1     Running   0          2m
# dapr-sidecar-injector-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

## Deploy a Sample Application

Test your Dapr installation with the official hello-world sample:

```bash
git clone https://github.com/dapr/quickstarts.git
cd quickstarts/tutorials/hello-kubernetes

kubectl apply -f deploy/redis.yaml
kubectl apply -f deploy/node.yaml
kubectl apply -f deploy/python.yaml

# Watch deployments come up
kubectl rollout status deployment/nodeapp
kubectl rollout status deployment/pythonapp
```

## Access the Dapr Dashboard

```bash
dapr dashboard -k -p 9999
# Open http://localhost:9999 in your browser
```

## Cleanup

```bash
dapr uninstall --kubernetes
minikube stop
```

## Summary

Installing Dapr on Minikube takes a few minutes with the Dapr CLI. Starting Minikube with adequate CPU and memory ensures the control plane components run without resource pressure. Use `dapr status -k` to verify all components are healthy before deploying your applications.
