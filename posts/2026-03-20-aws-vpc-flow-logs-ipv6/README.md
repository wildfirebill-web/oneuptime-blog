# How to Analyze IPv6 Traffic in AWS VPC Flow Logs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, IPv6, VPC Flow Logs, Network Monitoring, CloudWatch, Athena

Description: Enable and analyze VPC Flow Logs for IPv6 traffic, filter IPv6 flows in CloudWatch Insights and Athena, and monitor IPv6 security events.

## Introduction

AWS VPC Flow Logs capture information about IP traffic flowing through network interfaces, including IPv6 flows. Flow logs record source/destination IPv6 addresses, ports, protocols, and accept/reject decisions. Analyzing IPv6 flow logs helps identify security threats, verify routing, and understand IPv6 traffic patterns in your VPC.

## Enable VPC Flow Logs

```bash
VPC_ID="vpc-12345678"
LOG_GROUP="/aws/vpc/flowlogs"

# Create CloudWatch log group
aws logs create-log-group --log-group-name "$LOG_GROUP"

# Create IAM role for flow logs
ROLE_ARN=$(aws iam create-role \
    --role-name VPCFlowLogsRole \
    --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"Service": "vpc-flow-logs.amazonaws.com"},
            "Action": "sts:AssumeRole"
        }]
    }' \
    --query "Role.Arn" \
    --output text)

# Enable flow logs with IPv6 fields
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids "$VPC_ID" \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name "$LOG_GROUP" \
    --deliver-logs-permission-arn "$ROLE_ARN" \
    --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${windowstart} ${windowend} ${action} ${flowlogstatus} ${pkt-srcaddr} ${pkt-dstaddr} ${tcp-flags}'
```

## Terraform Flow Logs with IPv6 Fields

```hcl
# flow_logs.tf

resource "aws_flow_log" "vpc" {
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id

  # Custom format including IPv6-relevant fields
  log_format = "$${version} $${account-id} $${interface-id} $${srcaddr} $${dstaddr} $${srcport} $${dstport} $${protocol} $${packets} $${bytes} $${action} $${flow-direction} $${pkt-src-aws-service} $${pkt-dst-aws-service}"

  tags = { Name = "vpc-flow-logs" }
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/flowlogs"
  retention_in_days = 30
}

resource "aws_iam_role" "flow_logs" {
  name = "vpc-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "vpc-flow-logs.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "flow_logs" {
  name   = "vpc-flow-logs-policy"
  role   = aws_iam_role.flow_logs.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"]
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}
```

## Query IPv6 Flow Logs in CloudWatch Insights

```
# CloudWatch Logs Insights queries for IPv6 traffic

# All IPv6 traffic
fields srcaddr, dstaddr, srcport, dstport, protocol, bytes, action
| filter srcaddr like /:/
| sort bytes desc
| limit 100

# IPv6 traffic rejected by security groups
fields srcaddr, dstaddr, dstport, bytes
| filter srcaddr like /:/ and action = "REJECT"
| stats sum(bytes) as total_bytes by srcaddr
| sort total_bytes desc
| limit 20

# Top IPv6 destinations
fields dstaddr, bytes
| filter dstaddr like /:/
| stats sum(bytes) as total_bytes by dstaddr
| sort total_bytes desc
| limit 10

# IPv6 traffic to specific port (e.g., 443)
fields srcaddr, dstaddr, bytes
| filter srcaddr like /:/ and dstport = 443
| stats sum(bytes) as total by srcaddr
| sort total desc

# Security: Detect IPv6 port scanning (many ports from one source)
fields srcaddr, dstport
| filter srcaddr like /:/
| stats count_distinct(dstport) as ports_scanned by srcaddr
| filter ports_scanned > 10
| sort ports_scanned desc
```

## Query IPv6 Flows in Athena (S3 Flow Logs)

```sql
-- Create Athena table for flow logs
CREATE EXTERNAL TABLE vpc_flow_logs (
    version INT,
    account STRING,
    interfaceid STRING,
    sourceaddress STRING,
    destinationaddress STRING,
    sourceport INT,
    destinationport INT,
    protocol INT,
    numpackets INT,
    numbytes BIGINT,
    starttime INT,
    endtime INT,
    action STRING,
    logstatus STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ' '
LOCATION 's3://my-flow-logs-bucket/AWSLogs/123456789/vpcflowlogs/us-east-1/';

-- Query all IPv6 traffic
SELECT sourceaddress, destinationaddress, destinationport,
       SUM(numbytes) as total_bytes
FROM vpc_flow_logs
WHERE sourceaddress LIKE '%:%'  -- IPv6 addresses contain ':'
GROUP BY sourceaddress, destinationaddress, destinationport
ORDER BY total_bytes DESC
LIMIT 20;

-- Find rejected IPv6 connections
SELECT sourceaddress, destinationaddress, destinationport,
       COUNT(*) as attempts
FROM vpc_flow_logs
WHERE sourceaddress LIKE '%:%'
  AND action = 'REJECT'
GROUP BY sourceaddress, destinationaddress, destinationport
ORDER BY attempts DESC;
```

## Conclusion

VPC Flow Logs capture IPv6 traffic using the same format as IPv4, with IPv6 addresses recorded in the `srcaddr` and `dstaddr` fields. Filter IPv6 flows in CloudWatch Insights using `filter srcaddr like /:/` since IPv6 addresses always contain colons. In Athena, use `WHERE sourceaddress LIKE '%:%'`. Custom log formats can include additional fields like `flow-direction` and `pkt-src-aws-service` for richer analysis. Monitor IPv6 REJECT actions to identify misconfigured security groups or NACLs blocking legitimate IPv6 traffic.
