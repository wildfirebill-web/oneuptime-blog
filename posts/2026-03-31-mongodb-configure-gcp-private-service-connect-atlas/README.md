# How to Configure GCP Private Service Connect for MongoDB Atlas

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, GCP, Networking, Security

Description: Configure GCP Private Service Connect for MongoDB Atlas to securely route database traffic through private endpoints in your Google Cloud VPC.

---

## What is GCP Private Service Connect

GCP Private Service Connect (PSC) allows consumers to access managed services through private IP addresses within their own VPC. For MongoDB Atlas, PSC replaces the need for VPC peering or public internet access. Traffic never leaves Google's network, improving security and reducing latency for GCP-based workloads.

## Prerequisites

- Atlas M10 or higher cluster deployed in a GCP region
- GCP project with `compute.admin` and `dns.admin` IAM roles
- Atlas project owner permissions

## Step 1: Enable Private Endpoint in Atlas

In the Atlas UI, go to **Network Access** > **Private Endpoint** > **Add Private Endpoint**. Select **GCP** and your region. Atlas will provide a Service Attachment URI, for example:

```text
projects/mongodb-atlas-prod/regions/us-central1/serviceAttachments/psc-atlas-us-central1
```

## Step 2: Reserve a Static Internal IP

Reserve internal IP addresses in your GCP subnet for the PSC endpoint:

```bash
gcloud compute addresses create atlas-psc-ip \
  --region=us-central1 \
  --subnet=my-subnet \
  --address-type=INTERNAL
```

Get the allocated IP:

```bash
gcloud compute addresses describe atlas-psc-ip \
  --region=us-central1 \
  --format="get(address)"
```

## Step 3: Create a Forwarding Rule (PSC Endpoint)

Create a forwarding rule that connects your reserved IP to the Atlas service attachment:

```bash
gcloud compute forwarding-rules create atlas-psc-endpoint \
  --region=us-central1 \
  --network=my-vpc \
  --address=atlas-psc-ip \
  --target-service-attachment=projects/mongodb-atlas-prod/regions/us-central1/serviceAttachments/psc-atlas-us-central1 \
  --load-balancing-scheme=""
```

## Step 4: Register the Endpoint in Atlas

Submit the forwarding rule details to Atlas via the API:

```bash
curl --user "PUBLIC_KEY:PRIVATE_KEY" --digest \
  --header "Content-Type: application/json" \
  --request POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/privateEndpoint/gcp/endpointService/{endpointServiceId}/endpoint" \
  --data '{
    "endpointGroupName": "atlas-psc-endpoint",
    "gcpProjectId": "my-gcp-project",
    "endpoints": [
      {"ipAddress": "10.128.0.10", "endpointName": "atlas-psc-endpoint"}
    ]
  }'
```

## Step 5: Configure Private DNS

Create a private DNS zone so Atlas hostnames resolve to your PSC endpoint IP:

```bash
gcloud dns managed-zones create atlas-private-zone \
  --dns-name="privatelink.mongodb.net." \
  --description="Atlas Private DNS" \
  --visibility=private \
  --networks=my-vpc

gcloud dns record-sets create cluster0-pl-0.abcde.privatelink.mongodb.net. \
  --zone=atlas-private-zone \
  --type=A \
  --ttl=300 \
  --rrdatas=10.128.0.10
```

## Step 6: Connect from GCP

Use the private endpoint connection string from the Atlas UI:

```python
from pymongo import MongoClient

uri = "mongodb+srv://cluster0-pl-0.abcde.privatelink.mongodb.net/?authSource=admin"
client = MongoClient(uri, username="user", password="pass", tls=True)
print(client.server_info())
```

## Summary

GCP Private Service Connect for MongoDB Atlas enables private, fully internal connectivity from your GCP VPC to Atlas. The process involves reserving a static internal IP, creating a forwarding rule targeting the Atlas service attachment, registering the endpoint in Atlas, configuring a private DNS zone, and using the Atlas-provided private endpoint connection string. This approach eliminates public internet exposure and keeps all traffic on Google's internal network.
