# How to Configure Rancher HA with External Load Balancer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, High Availability, Load Balancer, Networking

Description: Configure an external load balancer for Rancher HA to distribute traffic across multiple Rancher server nodes and provide health check-based failover.

## Introduction

An external load balancer is the entry point for all Rancher traffic in an HA configuration. It distributes incoming requests across healthy Rancher instances and provides automatic failover when nodes fail. This guide covers configuring AWS ALB/NLB, GCP Load Balancer, and software load balancers for Rancher HA.

## Prerequisites

- Running Rancher HA cluster (RKE2 or K3s)
- Multiple Rancher server nodes
- Access to cloud provider or software load balancer
- TLS certificate for Rancher hostname

## Step 1: AWS Application Load Balancer

```bash
# Create target group for Rancher HTTPS

aws elbv2 create-target-group \
  --name rancher-tg \
  --protocol HTTPS \
  --port 443 \
  --vpc-id vpc-xxxxxxxx \
  --target-type instance \
  --health-check-protocol HTTPS \
  --health-check-path /ping \
  --health-check-interval-seconds 10 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200,301,302

# Register targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...target-group/rancher-tg/... \
  --targets \
    Id=i-rke2-server-01,Port=443 \
    Id=i-rke2-server-02,Port=443 \
    Id=i-rke2-server-03,Port=443

# Create ALB
aws elbv2 create-load-balancer \
  --name rancher-alb \
  --type application \
  --scheme internet-facing \
  --subnets subnet-az1 subnet-az2 subnet-az3 \
  --security-groups sg-rancher-lb

# Create HTTPS listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:.../rancher-alb/... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:.../certificate/... \
  --default-actions Type=forward,TargetGroupArn=arn:...rancher-tg/...

# Add HTTP redirect
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:.../rancher-alb/... \
  --protocol HTTP \
  --port 80 \
  --default-actions '[{"Type":"redirect","RedirectConfig":{"Protocol":"HTTPS","StatusCode":"HTTP_301"}}]'
```

## Step 2: AWS Network Load Balancer (for WebSocket Support)

```bash
# NLB is recommended for Rancher due to WebSocket connections (cluster agents)
aws elbv2 create-load-balancer \
  --name rancher-nlb \
  --type network \
  --scheme internet-facing \
  --subnets subnet-az1 subnet-az2 subnet-az3

# Create TCP target group
aws elbv2 create-target-group \
  --name rancher-tcp-tg \
  --protocol TCP \
  --port 443 \
  --vpc-id vpc-xxxxxxxx \
  --target-type instance \
  --health-check-protocol TCP \
  --health-check-port 443 \
  --health-check-interval-seconds 10

# NLB passes through SSL to Rancher nodes (SSL termination at Rancher)
aws elbv2 create-listener \
  --load-balancer-arn arn:.../rancher-nlb/... \
  --protocol TCP \
  --port 443 \
  --default-actions Type=forward,TargetGroupArn=arn:.../rancher-tcp-tg/...

# Also create listener for RKE2 API server (port 6443)
aws elbv2 create-listener \
  --load-balancer-arn arn:.../rancher-nlb/... \
  --protocol TCP \
  --port 6443 \
  --default-actions Type=forward,TargetGroupArn=arn:.../rke2-api-tg/...
```

## Step 3: GCP Load Balancer Configuration

```bash
# Create backend service for Rancher
gcloud compute backend-services create rancher-backend \
  --protocol=HTTPS \
  --port-name=https \
  --health-checks=rancher-health-check \
  --global

# Create health check
gcloud compute health-checks create https rancher-health-check \
  --port=443 \
  --request-path=/ping \
  --check-interval=10 \
  --timeout=5 \
  --healthy-threshold=2 \
  --unhealthy-threshold=3

# Add instance group to backend
gcloud compute backend-services add-backend rancher-backend \
  --instance-group=rancher-server-group \
  --instance-group-zone=us-central1-a \
  --global

# Create URL map and forwarding rule
gcloud compute url-maps create rancher-url-map \
  --default-service=rancher-backend

gcloud compute target-https-proxies create rancher-https-proxy \
  --url-map=rancher-url-map \
  --ssl-certificates=rancher-ssl-cert

gcloud compute forwarding-rules create rancher-forwarding-rule \
  --target-https-proxy=rancher-https-proxy \
  --ports=443 \
  --global
```

## Step 4: Session Persistence Configuration

```bash
# Rancher v2.6+ is stateless and doesn't require sticky sessions
# But for older versions or specific operations, configure session affinity

# AWS ALB - enable session stickiness
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:.../rancher-tg/... \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=86400
```

## Step 5: Health Check Configuration

```bash
# Rancher health endpoint
curl -sk https://rancher-node-01/ping
# Returns: pong

# Configure health check to use /ping
# This is a simple health check that Rancher always returns 200 for

# For deeper health checks, use:
curl -sk https://rancher-node-01/healthz
```

## Step 6: WebSocket Timeout Configuration

```bash
# Rancher cluster agents use long-lived WebSocket connections
# Configure load balancer for long timeouts

# AWS ALB - idle timeout
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:.../rancher-alb/... \
  --attributes Key=idle_timeout.timeout_seconds,Value=3600

# GCP - backend timeout
gcloud compute backend-services update rancher-backend \
  --timeout=3600 \
  --global
```

## Step 7: DNS Configuration

```bash
# Create DNS record pointing to load balancer
# AWS Route53
aws route53 change-resource-record-sets \
  --hosted-zone-id XXXXXXXXXX \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "rancher.example.com",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "rancher-nlb-xxx.elb.amazonaws.com"}]
      }
    }]
  }'
```

## Conclusion

A properly configured load balancer is essential for Rancher HA's reliability and performance. Use NLB over ALB when possible for better WebSocket support-the persistent connections from Rancher cluster agents require long-lived TCP connections that NLB handles more efficiently. Configure generous idle timeouts (3600s+) to prevent premature connection termination, and use the `/ping` endpoint for fast health checks. For production deployments, test failover behavior by temporarily removing nodes from the target group to verify traffic shifts correctly.
