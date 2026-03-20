# How to Set Up Alias IP Ranges for GCP Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Alias IP, IPv4, Kubernetes, Compute Engine, Networking

Description: Configure alias IP ranges on GCP Compute Engine instances to assign multiple IPv4 addresses from a subnet to a single VM or to Kubernetes pods running on that VM.

## Introduction

Alias IP ranges allow a VM's network interface to have additional secondary IPv4 address ranges assigned from the subnet. This is commonly used with Kubernetes Engine (GKE) to assign pod IP addresses from the VPC subnet directly, enabling native VPC routing for pod traffic.

## Adding Alias IP Range at Instance Creation

```bash
PROJECT_ID="my-gcp-project"

# Create a VM with an alias IP range for pods
gcloud compute instances create k8s-node-01 \
  --project=$PROJECT_ID \
  --zone=us-central1-a \
  --machine-type=e2-standard-4 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --subnet=app-subnet \
  --aliases=/24
```

`--aliases=/24` allocates a /24 block from the subnet as an alias range — the VM can use any IP in that block.

## Adding Alias IP Range to an Existing Instance

```bash
# Add a specific alias IP range to a running VM
gcloud compute instances network-interfaces update k8s-node-01 \
  --project=$PROJECT_ID \
  --zone=us-central1-a \
  --aliases=10.1.2.64/26
```

## Configuring Alias Range on the OS

After adding the alias range in GCP, configure it on the Linux OS:

```bash
# Add a secondary IP from the alias range
sudo ip addr add 10.1.2.65/26 dev ens4

# Add multiple IPs (for container/pod use)
for i in $(seq 65 70); do
  sudo ip addr add 10.1.2.$i/32 dev ens4
done

# Verify
ip addr show ens4
```

## Subnet-Level Secondary IP Ranges

To use alias IPs with GKE, create a secondary range on the subnet:

```bash
# Add a secondary range to the subnet for pod IPs
gcloud compute networks subnets update app-subnet \
  --project=$PROJECT_ID \
  --region=us-central1 \
  --add-secondary-ranges=pods=10.4.0.0/14

# Add another for services
gcloud compute networks subnets update app-subnet \
  --project=$PROJECT_ID \
  --region=us-central1 \
  --add-secondary-ranges=services=10.8.0.0/20
```

## Using Alias IPs with GKE (VPC-Native Clusters)

```bash
# Create a VPC-native GKE cluster using secondary ranges
gcloud container clusters create prod-cluster \
  --project=$PROJECT_ID \
  --zone=us-central1-a \
  --network=prod-vpc \
  --subnetwork=app-subnet \
  --cluster-secondary-range-name=pods \
  --services-secondary-range-name=services \
  --enable-ip-alias
```

With this configuration, each pod gets an IP from the `pods` secondary range, making pods directly reachable from other VMs in the VPC without NAT.

## Viewing Alias IP Ranges

```bash
# List alias IP ranges for an instance
gcloud compute instances describe k8s-node-01 \
  --project=$PROJECT_ID \
  --zone=us-central1-a \
  --format="get(networkInterfaces[0].aliasIpRanges)"

# List secondary ranges on a subnet
gcloud compute networks subnets describe app-subnet \
  --project=$PROJECT_ID \
  --region=us-central1 \
  --format="get(secondaryIpRanges)"
```

## Removing an Alias IP Range

```bash
gcloud compute instances network-interfaces update k8s-node-01 \
  --project=$PROJECT_ID \
  --zone=us-central1-a \
  --aliases=""
```

## Conclusion

Alias IP ranges enable secondary IPv4 addresses on GCP VM interfaces. Create subnet-level secondary ranges with `--add-secondary-ranges` and reference them with `--aliases` at instance or cluster level. VPC-native GKE clusters rely on alias IPs for pod networking, enabling direct pod IP routing within the VPC.
