# How to Install Rancher on Google Cloud Compute Engine

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, GCP, Cloud, Installation

Description: Step-by-step instructions for deploying Rancher on a Google Cloud Compute Engine instance with K3s.

Google Cloud Platform offers reliable and performant virtual machines through Compute Engine. Running Rancher on a GCE instance lets you manage Kubernetes clusters across multiple environments from a single dashboard. This guide takes you through provisioning a GCE instance, installing Rancher, and accessing the management UI.

## Prerequisites

- A Google Cloud account with a project and billing enabled
- Google Cloud SDK (gcloud CLI) installed and configured
- A domain name (optional but recommended)
- Sufficient quota for creating VM instances in your chosen region

## Step 1: Set Your Project and Region

```bash
gcloud config set project your-project-id
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
```

## Step 2: Create a Firewall Rule

Create a firewall rule that allows HTTP, HTTPS, and Kubernetes API traffic:

```bash
gcloud compute firewall-rules create rancher-allow-traffic \
  --allow tcp:22,tcp:80,tcp:443,tcp:6443 \
  --source-ranges 0.0.0.0/0 \
  --target-tags rancher-server \
  --description "Allow traffic for Rancher server"
```

## Step 3: Create the Compute Engine Instance

Launch an instance with Ubuntu 22.04 and adequate resources:

```bash
gcloud compute instances create rancher-server \
  --machine-type e2-standard-2 \
  --image-family ubuntu-2204-lts \
  --image-project ubuntu-os-cloud \
  --boot-disk-size 50GB \
  --boot-disk-type pd-ssd \
  --tags rancher-server \
  --metadata startup-script='#!/bin/bash
apt-get update && apt-get upgrade -y'
```

## Step 4: Reserve a Static External IP

Reserve and assign a static IP so your Rancher server address does not change:

```bash
gcloud compute addresses create rancher-ip \
  --region us-central1

gcloud compute instances delete-access-config rancher-server \
  --access-config-name "external-nat"

gcloud compute instances add-access-config rancher-server \
  --access-config-name "external-nat" \
  --address $(gcloud compute addresses describe rancher-ip --region us-central1 --format='get(address)')
```

## Step 5: SSH into the Instance

```bash
gcloud compute ssh rancher-server
```

## Step 6: Install K3s

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
```

Verify the node is ready:

```bash
sudo k3s kubectl get nodes
```

Export the kubeconfig:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

## Step 7: Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Step 8: Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Wait for the pods to be ready:

```bash
kubectl get pods -n cert-manager
```

## Step 9: Install Rancher

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

kubectl create namespace cattle-system

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set bootstrapPassword=admin \
  --set replicas=1
```

## Step 10: Configure DNS

Set up a DNS A record pointing to your static IP. If you use Google Cloud DNS:

```bash
gcloud dns record-sets create rancher.example.com \
  --zone=your-dns-zone \
  --type=A \
  --ttl=300 \
  --rrdatas=$(gcloud compute addresses describe rancher-ip --region us-central1 --format='get(address)')
```

## Step 11: Verify the Installation

```bash
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get pods
```

Navigate to `https://rancher.example.com` in your browser. Log in with the bootstrap password and set your new admin password.

## Using the Google Cloud Node Driver

Rancher includes a built-in Google Compute Engine node driver. After logging in, go to Cluster Management, click Create, and select Google GCE. Provide your service account credentials as a JSON key file. Rancher will provision GCE instances as Kubernetes nodes for new clusters.

## Performance Considerations

For a single-node Rancher installation managing a small number of clusters, an e2-standard-2 (2 vCPU, 8 GB RAM) is sufficient. For larger deployments managing many clusters, consider upgrading to e2-standard-4 or higher. Use SSD persistent disks for better I/O performance.

## Cleanup

To remove all resources:

```bash
gcloud compute instances delete rancher-server --quiet
gcloud compute addresses delete rancher-ip --region us-central1 --quiet
gcloud compute firewall-rules delete rancher-allow-traffic --quiet
```

## Summary

You now have Rancher running on Google Cloud Compute Engine. This deployment provides a central dashboard for managing Kubernetes clusters across GCP and other cloud providers. You can provision new clusters directly from Rancher using the GCE node driver or import existing clusters for unified management.
