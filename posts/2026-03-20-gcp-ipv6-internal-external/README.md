# Internal vs External IPv6 in Google Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, Internal IPv6, External IPv6, ULA, Google Cloud VPC

Description: Understand the difference between GCP internal IPv6 (ULA) and external IPv6 (globally routable) address types, when to use each, and how they affect connectivity.

## Introduction

Google Cloud offers two types of IPv6 for VPC subnets: external IPv6 (globally routable) and internal IPv6 (ULA - Unique Local Addresses). External IPv6 addresses are accessible from the internet and assigned from Google's public IPv6 ranges. Internal IPv6 uses RFC 4193 ULA addresses (`fd00::/8`) that are only routable within the VPC and connected networks. Choosing the right type depends on your security and connectivity requirements.

## External IPv6 (Globally Routable)

```bash
# Create subnet with external IPv6
gcloud compute networks subnets create subnet-external-ipv6 \
    --network=vpc-main \
    --region=us-east1 \
    --range=10.0.1.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=EXTERNAL \
    --project="$PROJECT"

# External IPv6 properties:
# - Addresses from Google's public IPv6 space (2600:1900::/28 range)
# - Globally routable from the internet
# - Can receive inbound connections (controlled by firewall rules)
# - VMs with external IPv6 can initiate outbound internet connections
# - /96 per VM, subnet gets a /48

# View external IPv6 prefix assigned
gcloud compute networks subnets describe subnet-external-ipv6 \
    --region=us-east1 \
    --format="get(externalIpv6Prefix)"
```

## Internal IPv6 (ULA)

```bash
# Create subnet with internal IPv6
gcloud compute networks subnets create subnet-internal-ipv6 \
    --network=vpc-main \
    --region=us-east1 \
    --range=10.0.2.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=INTERNAL \
    --project="$PROJECT"

# Internal IPv6 properties:
# - Addresses from ULA range (fd::/8)
# - NOT globally routable — only within VPC and connected networks
# - No internet access without Cloud NAT for IPv6
# - More secure for backend services
# - Lower risk of accidental internet exposure

# View internal IPv6 prefix
gcloud compute networks subnets describe subnet-internal-ipv6 \
    --region=us-east1 \
    --format="get(ipv6CidrRange)"
```

## Use Case Comparison

```
External IPv6 Use Cases:
  - Web servers and public-facing APIs
  - CDN origin servers
  - Services that need inbound internet IPv6 connections
  - Load balancer backends that need direct IPv6

Internal IPv6 Use Cases:
  - Databases and internal APIs
  - Service mesh communication within GCP
  - Microservices that don't need internet access
  - Workloads where all internet access goes through Cloud NAT
  - Backend clusters where internet access is optional
```

## Check VM IPv6 Address Type

```bash
# Describe a VM's network interface to see IPv6 address type
gcloud compute instances describe vm-web-01 \
    --zone=us-east1-b \
    --format="json(networkInterfaces[].{ipv6Access:ipv6AccessType, ipv6Addr:ipv6Address, internalIpv6:internalIpv6PrefixLength})"

# External IPv6 shows:
# ipv6AccessType: EXTERNAL
# ipv6Address: 2600:1900:4000:abc1:8000::

# Internal IPv6 shows:
# ipv6AccessType: INTERNAL
# ipv6Address: fd20:0000:0000:0001::
```

## Switching Between Internal and External

```bash
# You CANNOT change ipv6-access-type on an existing subnet
# You must create a new subnet with the desired access type

# Option: Create secondary subnet with different type
gcloud compute networks subnets create subnet-web-external \
    --network=vpc-main \
    --region=us-east1 \
    --range=10.0.10.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=EXTERNAL

# Migrate VMs to new subnet or recreate them
```

## Internet Connectivity for Internal IPv6

```bash
# Internal IPv6 VMs need Cloud NAT for internet access
gcloud compute routers create router-nat \
    --network=vpc-main \
    --region=us-east1

gcloud compute routers nats create nat-ipv6 \
    --router=router-nat \
    --region=us-east1 \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips \
    --enable-endpoint-independent-mapping

# Now internal IPv6 VMs can initiate outbound internet connections
# Inbound connections are still blocked
```

## Firewall Rules for External vs Internal

```bash
# External IPv6: need explicit rules for internet inbound
gcloud compute firewall-rules create allow-http-ipv6 \
    --network=vpc-main \
    --direction=INGRESS \
    --priority=1000 \
    --source-ranges="::/0" \
    --rules=tcp:80,tcp:443 \
    --target-tags=web-server

# Internal IPv6: no internet inbound possible
# Firewall rules only needed for inter-VPC or inter-service traffic
gcloud compute firewall-rules create allow-internal-ipv6 \
    --network=vpc-main \
    --direction=INGRESS \
    --priority=1000 \
    --source-ranges="fd00::/8" \  # ULA range
    --rules=all \
    --target-tags=internal-service
```

## Conclusion

GCP external IPv6 provides globally routable addresses for internet-facing services, while internal IPv6 (ULA) provides secure inter-service communication without internet exposure. External subnets suit public-facing workloads; internal subnets suit backend services. The `ipv6-access-type` cannot be changed after subnet creation, so plan carefully. Internal IPv6 VMs can access the internet via Cloud NAT for IPv6, maintaining outbound connectivity while blocking inbound connections from the internet.
