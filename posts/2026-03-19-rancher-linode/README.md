# How to Install Rancher on Linode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Linode, Cloud, Installation

Description: Learn how to set up Rancher on a Linode instance for easy Kubernetes cluster management.

Linode, now part of Akamai, offers straightforward cloud computing with predictable pricing. Running Rancher on a Linode instance provides a simple path to centralized Kubernetes management. This guide covers the complete setup from provisioning a Linode to accessing the Rancher dashboard.

## Prerequisites

- A Linode account
- The Linode CLI installed and configured (`linode-cli`)
- An SSH key pair
- A domain name (optional but recommended)

## Step 1: Create a Linode Instance

Create a Linode with at least 4 GB RAM. The Linode 4GB plan is a good starting point:

```bash
linode-cli linodes create \
  --type g6-standard-2 \
  --region us-east \
  --image linode/ubuntu22.04 \
  --root_pass "YourSecurePassword123!" \
  --authorized_keys "$(cat ~/.ssh/id_rsa.pub)" \
  --label rancher-server \
  --booted true
```

Get the Linode IP address:

```bash
linode-cli linodes list --label rancher-server --format ipv4 --text --no-headers
```

## Step 2: Configure the Firewall

Create a Cloud Firewall for your Linode:

```bash
linode-cli firewalls create \
  --label rancher-fw \
  --rules.inbound_policy DROP \
  --rules.outbound_policy ACCEPT \
  --rules.inbound '[
    {"action":"ACCEPT","protocol":"TCP","ports":"22","addresses":{"ipv4":["0.0.0.0/0"]}},
    {"action":"ACCEPT","protocol":"TCP","ports":"80","addresses":{"ipv4":["0.0.0.0/0"]}},
    {"action":"ACCEPT","protocol":"TCP","ports":"443","addresses":{"ipv4":["0.0.0.0/0"]}},
    {"action":"ACCEPT","protocol":"TCP","ports":"6443","addresses":{"ipv4":["0.0.0.0/0"]}}
  ]'
```

Attach the firewall to your Linode:

```bash
LINODE_ID=$(linode-cli linodes list --label rancher-server --format id --text --no-headers)
FIREWALL_ID=$(linode-cli firewalls list --label rancher-fw --format id --text --no-headers)

linode-cli firewalls device-create $FIREWALL_ID \
  --id $LINODE_ID \
  --type linode
```

## Step 3: SSH into the Linode

```bash
ssh root@<linode-ip>
```

## Step 4: Install K3s

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
```

Verify K3s is running:

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

Confirm cert-manager pods are running:

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

## Step 8: Configure DNS

Create a DNS A record pointing your domain to the Linode IP address. If you use Linode DNS Manager:

```bash
linode-cli domains records-create <domain-id> \
  --type A \
  --name rancher \
  --target <linode-ip> \
  --ttl_sec 300
```

## Step 9: Verify and Access Rancher

```bash
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get pods
```

Navigate to `https://rancher.example.com`, log in with the bootstrap password, and configure your admin credentials.

## Using Linode Node Driver with Rancher

Rancher includes a Linode node driver that lets you provision Kubernetes clusters directly on Linode infrastructure:

1. In the Rancher UI, go to Cluster Management
2. Click Create and select Linode
3. Enter your Linode API token
4. Configure instance type, region, and image
5. Set up your node pools and create the cluster

Rancher will provision Linode instances and configure them as Kubernetes nodes automatically.

## NodeBalancer Integration

For production deployments, consider placing a Linode NodeBalancer in front of your Rancher server:

```bash
linode-cli nodebalancers create \
  --region us-east \
  --label rancher-lb

linode-cli nodebalancers config-create <nodebalancer-id> \
  --port 443 \
  --protocol https \
  --ssl_cert "$(cat /path/to/cert.pem)" \
  --ssl_key "$(cat /path/to/key.pem)"
```

## Cleanup

```bash
linode-cli linodes delete $LINODE_ID
linode-cli firewalls delete $FIREWALL_ID
```

## Summary

You have Rancher running on a Linode instance with K3s as the underlying Kubernetes distribution. Linode provides predictable pricing and solid performance for hosting Rancher. With the built-in Linode node driver, you can easily provision additional clusters and manage all your Kubernetes infrastructure from a single dashboard.
