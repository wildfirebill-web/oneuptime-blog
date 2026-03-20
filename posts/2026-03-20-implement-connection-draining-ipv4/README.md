# How to Implement Connection Draining for IPv4 Load Balancers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Load Balancing, Connection Draining, IPv4, HAProxy, Nginx, AWS, Zero Downtime

Description: Configure connection draining on HAProxy, Nginx, and AWS load balancers to gracefully remove IPv4 backend servers from rotation without interrupting active connections.

## Introduction

Connection draining (also called deregistration delay or graceful removal) ensures that when a backend server is removed from a load balancer, existing active connections are allowed to complete while no new connections are sent to it. This is essential for zero-downtime deployments and maintenance windows.

## HAProxy Connection Draining

### Method 1: Admin Socket (Immediate)

```bash
# Put server into DRAIN mode — stops new connections, lets existing finish
echo "set server web-backends/web2 state drain" | sudo socat stdio /run/haproxy/admin.sock

# Wait for connections to drain
echo "show servers state web-backends" | sudo socat stdio /run/haproxy/admin.sock | grep web2

# After connections reach 0, take the server fully offline
echo "set server web-backends/web2 state maint" | sudo socat stdio /run/haproxy/admin.sock
```

### Method 2: Gradual Weight Reduction

```bash
# Reduce weight gradually over time
for weight in 50 25 10 0; do
  echo "set server web-backends/web2 weight $weight" | sudo socat stdio /run/haproxy/admin.sock
  sleep 30
done
echo "set server web-backends/web2 state maint" | sudo socat stdio /run/haproxy/admin.sock
```

### HAProxy timeout Tuning for Draining

```
defaults
    timeout client    30s
    timeout server    30s
    timeout connect   5s
    # Allows in-flight requests to complete within these timeouts
```

## Nginx Upstream Draining

Nginx OSS doesn't have native draining but you can implement it:

```bash
# Remove server from upstream (edit config + reload)
# nginx reload (nginx -s reload) is zero-downtime — existing connections complete
sudo nginx -s reload
```

For Nginx Plus, use the upstream API:

```bash
# Set server to drain state (Nginx Plus API)
curl -X PATCH http://localhost:8080/api/6/http/upstreams/app-servers/servers/1 \
  -d '{"drain": true}'
```

## AWS ALB Connection Draining (Deregistration Delay)

```bash
TG_ARN="arn:aws:elasticloadbalancing:us-east-1:123456789:targetgroup/web-tg/abc123"

# Set deregistration delay to 60 seconds (default is 300s)
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes Key=deregistration_delay.timeout_seconds,Value=60

# Deregister a target — ALB keeps sending existing connections for 60s
aws elbv2 deregister-targets \
  --target-group-arn $TG_ARN \
  --targets Id=i-0abc123def456
```

Monitor draining:

```bash
# Watch target health state (will show "draining")
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --targets Id=i-0abc123def456
```

## AWS NLB Connection Draining

```bash
# NLB uses the same attribute
aws elbv2 modify-target-group-attributes \
  --target-group-arn $NLB_TG_ARN \
  --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

NLB waits up to 30 seconds for TCP connections to close naturally.

## GCP Backend Service Connection Draining

```bash
# Set connection draining timeout (in seconds, 0=disabled, max=3600)
gcloud compute backend-services update web-backend \
  --project=my-project \
  --global \
  --connection-draining-timeout=60
```

When an instance group is updated or a VM is removed, GCP waits up to 60 seconds for connections to close.

## Application-Level Draining (Kubernetes/Containers)

For containerized apps, hook into the pre-stop lifecycle:

```yaml
# Kubernetes deployment with preStop hook
containers:
  - name: web-app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 30"]
    # Kubernetes sends SIGTERM after preStop, then waits terminationGracePeriodSeconds
```

```yaml
spec:
  terminationGracePeriodSeconds: 60
```

## Draining Checklist

```
1. Mark server as draining/disabled in LB → no new connections
2. Wait for active connection count to reach 0
3. Stop the application gracefully (SIGTERM → graceful shutdown)
4. Remove from service discovery / DNS
5. Proceed with update or decommission
```

## Conclusion

Connection draining prevents request failures during deployments. In HAProxy, use `set server state drain` via the socket API. In AWS, set `deregistration_delay.timeout_seconds` and call `deregister-targets`. In GCP, set `--connection-draining-timeout` on the backend service. Choose a drain timeout longer than your longest request processing time.
