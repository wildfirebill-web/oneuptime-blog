# How to Configure DNS-Based IPv4 Load Balancing with Route 53

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Route 53, Load Balancing, IPv4, AWS, Weighted Routing

Description: Configure AWS Route 53 for DNS-based IPv4 load balancing using weighted, latency-based, and failover routing policies to distribute traffic across multiple endpoints.

## Introduction

Route 53 DNS-based load balancing distributes traffic by returning different IPv4 addresses in DNS responses. It operates at the DNS level - no single load balancer proxy handles all traffic. It supports weighted, latency-based, geolocation, failover, and multivalue routing policies.

## Weighted Routing (Round Robin with Control)

Distribute traffic proportionally based on weights:

```bash
HOSTED_ZONE_ID="Z1234567890ABC"

# Create server 1 record with weight 70 (70% of traffic)

aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "server1",
        "Weight": 70,
        "TTL": 60,
        "ResourceRecords": [{"Value": "203.0.113.10"}]
      }
    }]
  }'

# Create server 2 record with weight 30 (30% of traffic)
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "server2",
        "Weight": 30,
        "TTL": 60,
        "ResourceRecords": [{"Value": "203.0.113.11"}]
      }
    }]
  }'
```

## Latency-Based Routing

Route clients to the region with lowest latency:

```bash
# US East endpoint
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "us-east-1",
        "Region": "us-east-1",
        "TTL": 60,
        "ResourceRecords": [{"Value": "203.0.113.10"}]
      }
    }]
  }'

# EU West endpoint
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "eu-west-1",
        "Region": "eu-west-1",
        "TTL": 60,
        "ResourceRecords": [{"Value": "203.0.113.20"}]
      }
    }]
  }'
```

## Failover Routing

Active-passive failover:

```bash
# Create health check for primary
HC_ID=$(aws route53 create-health-check \
  --caller-reference "$(date +%s)" \
  --health-check-config '{
    "IPAddress": "203.0.113.10",
    "Port": 80,
    "Type": "HTTP",
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }' \
  --query 'HealthCheck.Id' --output text)

# Primary record with health check
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"app.example.com\",
        \"Type\": \"A\",
        \"SetIdentifier\": \"primary\",
        \"Failover\": \"PRIMARY\",
        \"HealthCheckId\": \"$HC_ID\",
        \"TTL\": 60,
        \"ResourceRecords\": [{\"Value\": \"203.0.113.10\"}]
      }
    }]
  }"

# Secondary record (no health check needed)
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "secondary",
        "Failover": "SECONDARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "203.0.113.11"}]
      }
    }]
  }'
```

## Multivalue Answer Routing

Return up to 8 healthy IPs per query:

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"app.example.com\",
        \"Type\": \"A\",
        \"SetIdentifier\": \"server3\",
        \"MultiValueAnswer\": true,
        \"HealthCheckId\": \"$HC_ID\",
        \"TTL\": 60,
        \"ResourceRecords\": [{\"Value\": \"203.0.113.12\"}]
      }
    }]
  }"
```

## DNS Load Balancing Limitations

| Limitation | Impact |
|---|---|
| Client DNS caching | Slow failover (depends on TTL) |
| Non-health-aware clients | May cache down server IPs |
| No connection draining | Abrupt cutover |
| Uneven distribution | Client caching skews distribution |

Use low TTL (60s) for faster failover. For precise LB, combine Route 53 with an ALB.

## Checking Current DNS Records

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --query 'ResourceRecordSets[?Name==`app.example.com.`]'

# Test DNS resolution
dig app.example.com @8.8.8.8 +short
```

## Conclusion

Route 53 weighted routing distributes traffic proportionally (use weights 0–255 or any integer). Latency-based routing sends clients to the lowest-latency region. Failover routing requires health checks on the primary record. Use multivalue answer routing as a basic round-robin across up to 8 IPs with health checking. Keep TTLs at 60s or lower for faster failover responsiveness.
