# How to Configure IP Access Lists for MongoDB Atlas

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Security, Networking, Access Control

Description: Learn how to configure IP access lists in MongoDB Atlas to control which IP addresses and CIDR blocks can connect to your clusters.

---

## What Are IP Access Lists in Atlas

MongoDB Atlas uses IP access lists (formerly IP whitelists) as the first line of defense for network access. By default, no IP address can connect to an Atlas cluster until explicitly allowed. Access entries can be individual IP addresses, CIDR blocks, or AWS security groups (for peered connections).

## Adding an Entry via the Atlas UI

Navigate to **Network Access** > **IP Access List** > **Add IP Address**. You can:
- Enter a specific IP address: `203.0.113.42`
- Enter a CIDR block: `10.0.0.0/16`
- Click **Add Current IP Address** to add your local machine's IP

Provide a comment to describe the purpose of each entry for audit purposes.

## Managing IP Access List via the API

### List Current Entries

```bash
curl --user "PUBLIC_KEY:PRIVATE_KEY" --digest \
  --request GET \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/accessList"
```

### Add Multiple Entries

```bash
curl --user "PUBLIC_KEY:PRIVATE_KEY" --digest \
  --header "Content-Type: application/json" \
  --request POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/accessList" \
  --data '[
    {"ipAddress": "203.0.113.42", "comment": "CI/CD server"},
    {"cidrBlock": "10.0.0.0/16", "comment": "App VPC"},
    {"cidrBlock": "192.168.1.0/24", "comment": "Office network"}
  ]'
```

### Delete an Entry

```bash
curl --user "PUBLIC_KEY:PRIVATE_KEY" --digest \
  --request DELETE \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/accessList/203.0.113.42"
```

## Using Temporary Access

Atlas supports temporary access entries that expire after a set time, useful for debugging or one-off operations:

```bash
curl --user "PUBLIC_KEY:PRIVATE_KEY" --digest \
  --header "Content-Type: application/json" \
  --request POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/accessList" \
  --data '[{
    "ipAddress": "203.0.113.99",
    "comment": "Temp debug access",
    "deleteAfterDate": "2026-04-01T00:00:00Z"
  }]'
```

## Managing Access List with Terraform

For infrastructure-as-code workflows, manage access list entries with the MongoDB Atlas Terraform provider:

```text
resource "mongodbatlas_project_ip_access_list" "app_vpc" {
  project_id = var.atlas_project_id
  cidr_block = "10.0.0.0/16"
  comment    = "Application VPC CIDR"
}

resource "mongodbatlas_project_ip_access_list" "ci_server" {
  project_id = var.atlas_project_id
  ip_address = "203.0.113.42"
  comment    = "CI/CD server"
}
```

## Allow Access from Anywhere (Not Recommended)

Adding `0.0.0.0/0` allows connections from any IP. This is only appropriate for development clusters and should never be used for production:

```bash
curl --user "PUBLIC_KEY:PRIVATE_KEY" --digest \
  --header "Content-Type: application/json" \
  --request POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/accessList" \
  --data '[{"cidrBlock": "0.0.0.0/0", "comment": "Allow all - DEV ONLY"}]'
```

## Summary

Atlas IP access lists provide network-level access control for your clusters by allowlisting specific IP addresses or CIDR ranges. Entries can be managed through the Atlas UI, REST API, or Terraform provider. For production, use the most restrictive CIDR possible - prefer VPC CIDRs over individual IPs to avoid constant updates, and use temporary entries for short-lived access needs.
