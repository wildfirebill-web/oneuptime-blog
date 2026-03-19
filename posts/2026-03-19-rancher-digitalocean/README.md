# How to Install Rancher on DigitalOcean Droplets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, DigitalOcean, Cloud, Installation

Description: A complete guide to deploying Rancher on a DigitalOcean Droplet for managing Kubernetes clusters.

DigitalOcean is known for its simplicity and developer-friendly approach to cloud infrastructure. Running Rancher on a DigitalOcean Droplet provides a straightforward way to set up Kubernetes cluster management without the complexity of larger cloud platforms. This guide covers the entire installation process.

## Prerequisites

- A DigitalOcean account with API access
- The `doctl` CLI tool installed and authenticated
- An SSH key added to your DigitalOcean account
- A domain name (optional but recommended)

## Step 1: Create a Droplet

Create a Droplet with sufficient resources for running Rancher. A 4 GB RAM / 2 vCPU Droplet is the minimum recommended size:

```bash
doctl compute droplet create rancher-server \
  --region nyc3 \
  --size s-2vcpu-4gb \
  --image ubuntu-22-04-x64 \
  --ssh-keys $(doctl compute ssh-key list --format ID --no-header | head -1) \
  --tag-name rancher \
  --wait
```

Retrieve the Droplet IP address:

```bash
doctl compute droplet get rancher-server --format PublicIPv4 --no-header
```

## Step 2: Configure the Firewall

Create a firewall to control traffic to your Droplet:

```bash
DROPLET_ID=$(doctl compute droplet get rancher-server --format ID --no-header)

doctl compute firewall create \
  --name rancher-fw \
  --droplet-ids $DROPLET_ID \
  --inbound-rules "protocol:tcp,ports:22,address:0.0.0.0/0 protocol:tcp,ports:80,address:0.0.0.0/0 protocol:tcp,ports:443,address:0.0.0.0/0 protocol:tcp,ports:6443,address:0.0.0.0/0" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0 protocol:udp,ports:all,address:0.0.0.0/0 protocol:icmp,address:0.0.0.0/0"
```

## Step 3: SSH into the Droplet

```bash
ssh root@<droplet-ip>
```

## Step 4: Install K3s

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
```

Confirm K3s is running:

```bash
k3s kubectl get nodes
```

Set up the kubeconfig:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

## Step 5: Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Step 6: Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Verify the installation:

```bash
kubectl get pods -n cert-manager
```

## Step 7: Install Rancher

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

## Step 8: Set Up DNS

Add a DNS A record for your domain pointing to the Droplet IP. If you manage DNS through DigitalOcean:

```bash
doctl compute domain records create example.com \
  --record-type A \
  --record-name rancher \
  --record-data <droplet-ip> \
  --record-ttl 300
```

## Step 9: Verify and Access Rancher

Wait for the deployment to finish:

```bash
kubectl -n cattle-system rollout status deploy/rancher
```

Open `https://rancher.example.com` in your browser. Accept the certificate warning if using self-signed certificates, log in with the bootstrap password, and set your admin password.

## Step 10: Enable the DigitalOcean Node Driver

Rancher has a built-in DigitalOcean node driver. To use it for provisioning clusters:

1. Navigate to Cluster Management in the Rancher UI
2. Click Create and select DigitalOcean
3. Enter your DigitalOcean API token
4. Configure the Droplet size, region, and OS image for worker nodes
5. Define your node pools and create the cluster

Rancher will automatically provision Droplets and configure them as Kubernetes nodes.

## Using a Reserved IP

For production use, assign a Reserved IP to ensure your Rancher server address stays the same even if you recreate the Droplet:

```bash
doctl compute reserved-ip create --region nyc3

doctl compute reserved-ip-action assign <reserved-ip> $DROPLET_ID
```

Update your DNS record to point to the Reserved IP instead of the Droplet IP.

## Cleanup

To remove the Rancher Droplet and associated resources:

```bash
doctl compute droplet delete rancher-server --force
doctl compute firewall delete <firewall-id> --force
```

## Summary

Rancher is now running on your DigitalOcean Droplet. You have a fully functional Kubernetes management platform that can provision and manage clusters across DigitalOcean and other providers. The DigitalOcean node driver makes it particularly easy to create new clusters with Droplets as worker nodes directly from the Rancher interface.
