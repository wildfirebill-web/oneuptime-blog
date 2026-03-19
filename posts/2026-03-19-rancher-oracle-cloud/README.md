# How to Install Rancher on Oracle Cloud Infrastructure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Oracle Cloud, Cloud, Installation

Description: A practical guide to deploying Rancher on an Oracle Cloud Infrastructure compute instance.

Oracle Cloud Infrastructure (OCI) offers competitive pricing and a generous free tier that includes compute instances suitable for running Rancher. This guide covers the complete process of setting up Rancher on an OCI compute instance, from creating the infrastructure to accessing the Rancher dashboard.

## Prerequisites

- An Oracle Cloud account
- OCI CLI installed and configured
- An SSH key pair
- A domain name (optional but recommended)
- A compartment ID for resource organization

## Step 1: Set Up Environment Variables

Set commonly used values as environment variables:

```bash
export COMPARTMENT_ID="ocid1.compartment.oc1..your-compartment-id"
export AVAILABILITY_DOMAIN="Uocm:US-ASHBURN-AD-1"
```

## Step 2: Create a Virtual Cloud Network

Create a VCN with the necessary networking components:

```bash
VCN_ID=$(oci network vcn create \
  --compartment-id $COMPARTMENT_ID \
  --display-name rancher-vcn \
  --cidr-blocks '["10.0.0.0/16"]' \
  --query 'data.id' --raw-output)

SUBNET_ID=$(oci network subnet create \
  --compartment-id $COMPARTMENT_ID \
  --vcn-id $VCN_ID \
  --display-name rancher-subnet \
  --cidr-block 10.0.1.0/24 \
  --query 'data.id' --raw-output)

IGW_ID=$(oci network internet-gateway create \
  --compartment-id $COMPARTMENT_ID \
  --vcn-id $VCN_ID \
  --display-name rancher-igw \
  --is-enabled true \
  --query 'data.id' --raw-output)
```

Create a route table with a default route to the internet gateway:

```bash
RT_ID=$(oci network route-table create \
  --compartment-id $COMPARTMENT_ID \
  --vcn-id $VCN_ID \
  --display-name rancher-rt \
  --route-rules "[{\"destination\":\"0.0.0.0/0\",\"destinationType\":\"CIDR_BLOCK\",\"networkEntityId\":\"$IGW_ID\"}]" \
  --query 'data.id' --raw-output)
```

## Step 3: Create a Security List

Create a security list that allows SSH, HTTP, HTTPS, and Kubernetes API traffic:

```bash
SL_ID=$(oci network security-list create \
  --compartment-id $COMPARTMENT_ID \
  --vcn-id $VCN_ID \
  --display-name rancher-sl \
  --ingress-security-rules '[
    {"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"min":22,"max":22}}},
    {"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"min":80,"max":80}}},
    {"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"min":443,"max":443}}},
    {"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"min":6443,"max":6443}}}
  ]' \
  --egress-security-rules '[{"protocol":"all","destination":"0.0.0.0/0"}]' \
  --query 'data.id' --raw-output)
```

## Step 4: Launch the Compute Instance

Find the Ubuntu 22.04 image for your region:

```bash
IMAGE_ID=$(oci compute image list \
  --compartment-id $COMPARTMENT_ID \
  --operating-system "Canonical Ubuntu" \
  --operating-system-version "22.04" \
  --shape "VM.Standard.E4.Flex" \
  --sort-by TIMECREATED \
  --sort-order DESC \
  --limit 1 \
  --query 'data[0].id' --raw-output)
```

Create the instance:

```bash
oci compute instance launch \
  --compartment-id $COMPARTMENT_ID \
  --availability-domain $AVAILABILITY_DOMAIN \
  --display-name rancher-server \
  --image-id $IMAGE_ID \
  --shape VM.Standard.E4.Flex \
  --shape-config '{"ocpus":2,"memoryInGBs":16}' \
  --subnet-id $SUBNET_ID \
  --assign-public-ip true \
  --ssh-authorized-keys-file ~/.ssh/id_rsa.pub \
  --boot-volume-size-in-gbs 50
```

Retrieve the public IP address:

```bash
oci compute instance list-vnics \
  --compartment-id $COMPARTMENT_ID \
  --instance-id <instance-id> \
  --query 'data[0]."public-ip"' --raw-output
```

## Step 5: SSH into the Instance

```bash
ssh ubuntu@<public-ip>
```

## Step 6: Install K3s, Helm, and Rancher

Install K3s:

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

Install Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Install cert-manager:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Install Rancher:

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

## Step 7: Verify and Access

```bash
kubectl -n cattle-system rollout status deploy/rancher
```

Configure your DNS to point to the instance public IP and access `https://rancher.example.com` in your browser.

## Using OCI Free Tier

OCI offers Always Free compute instances that can run Rancher for testing. The ARM-based Ampere A1 instances provide up to 4 OCPUs and 24 GB of RAM in the free tier, which is more than sufficient for a Rancher server.

## Summary

Rancher is now running on Oracle Cloud Infrastructure. OCI provides competitive pricing and a generous free tier, making it an excellent platform for hosting your Rancher management server. You can import existing clusters or create new ones from the Rancher dashboard.
