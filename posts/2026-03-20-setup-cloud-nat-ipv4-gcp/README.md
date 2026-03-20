# How to Set Up Cloud NAT for IPv4 Outbound Access in GCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Cloud NAT, IPv4, Outbound Access, Networking, NAT Gateway

Description: Configure GCP Cloud NAT with a Cloud Router to provide outbound IPv4 internet access for VM instances without external IP addresses in a private subnet.

## Introduction

Cloud NAT is a managed NAT gateway in GCP that allows VM instances without external IPv4 addresses to access the internet. Cloud NAT is regional and works with a Cloud Router for control-plane operations. It scales automatically and does not require a NAT VM.

## Step 1: Create a Cloud Router

Cloud NAT requires a Cloud Router in the same region:

```bash
PROJECT_ID="my-gcp-project"
REGION="us-central1"
NETWORK="prod-vpc"

gcloud compute routers create prod-router \
  --project=$PROJECT_ID \
  --network=$NETWORK \
  --region=$REGION
```

## Step 2: Create Cloud NAT

```bash
# Create NAT that covers all subnets in the region automatically

gcloud compute routers nats create prod-nat \
  --project=$PROJECT_ID \
  --router=prod-router \
  --region=$REGION \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

## Step 3: Verify VMs Can Reach the Internet

```bash
# VMs without external IPs should now reach internet
gcloud compute ssh private-vm-01 \
  --project=$PROJECT_ID \
  --zone=us-central1-a \
  --command="curl -s https://icanhazip.com"
```

The returned IP is the Cloud NAT's automatically allocated external IP.

## Using Static External IPs for NAT

For allowlisting outbound IPs:

```bash
# Reserve a static external IP
gcloud compute addresses create nat-ip-1 \
  --project=$PROJECT_ID \
  --region=$REGION

# Create NAT with the static IP
gcloud compute routers nats create prod-nat \
  --project=$PROJECT_ID \
  --router=prod-router \
  --region=$REGION \
  --nat-external-ip-pool=nat-ip-1 \
  --nat-all-subnet-ip-ranges
```

## Limiting NAT to Specific Subnets

```bash
# Only NAT traffic from specific subnets
gcloud compute routers nats create prod-nat \
  --project=$PROJECT_ID \
  --router=prod-router \
  --region=$REGION \
  --auto-allocate-nat-external-ips \
  --nat-custom-subnet-ip-ranges=app-subnet,db-subnet
```

## Configuring NAT Port Allocation

```bash
# Increase minimum ports per VM for high-concurrency apps
gcloud compute routers nats update prod-nat \
  --project=$PROJECT_ID \
  --router=prod-router \
  --region=$REGION \
  --min-ports-per-vm=128
```

Default is 64 ports per VM. High-concurrency apps may need 128–1024.

## Enabling Cloud NAT Logging

```bash
# Enable logging for all NAT translations
gcloud compute routers nats update prod-nat \
  --project=$PROJECT_ID \
  --router=prod-router \
  --region=$REGION \
  --enable-logging \
  --log-filter=ALL
```

Logs appear in Cloud Logging and include source IP, destination, NAT IP, and port.

## Checking NAT Configuration

```bash
# View NAT gateway details
gcloud compute routers nats describe prod-nat \
  --project=$PROJECT_ID \
  --router=prod-router \
  --region=$REGION

# View current external IPs used
gcloud compute routers get-nat-ip-info prod-router \
  --project=$PROJECT_ID \
  --region=$REGION
```

## Cloud NAT vs NAT Instance

| Feature | Cloud NAT | NAT Instance |
|---|---|---|
| Managed | Yes | No |
| Scaling | Automatic | Manual |
| HA | Built-in | Requires setup |
| Ports per VM | Configurable | Limited by VM size |
| Cost | Per hour + per GB | VM cost |

## Conclusion

Cloud NAT requires a Cloud Router in the same region. Create the NAT gateway with `--auto-allocate-nat-external-ips` for simplicity, or specify static IPs with `--nat-external-ip-pool` for predictable outbound addresses. Monitor port exhaustion with Cloud Logging and increase `--min-ports-per-vm` for high-concurrency workloads.
