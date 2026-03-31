# How to Set Up ClickHouse with Cloud-Native Load Balancers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Load Balancer, Kubernetes, Cloud, High Availability

Description: Configure AWS ALB, GCP Load Balancer, or Kubernetes ingress in front of a ClickHouse cluster to distribute queries and enable zero-downtime rolling upgrades.

---

ClickHouse exposes both an HTTP interface (port 8123) and a native TCP interface (port 9000). Cloud-native load balancers work well for the HTTP interface; native TCP requires a TCP/L4 load balancer.

## Kubernetes Service (L4 TCP)

For the native ClickHouse client protocol, use a `LoadBalancer` service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clickhouse-lb
spec:
  type: LoadBalancer
  selector:
    app: clickhouse
  ports:
    - name: native
      port: 9000
      targetPort: 9000
    - name: http
      port: 8123
      targetPort: 8123
```

## AWS ALB for HTTP Interface

Use an ALB target group with health checks on the ClickHouse HTTP ping endpoint:

```bash
aws elbv2 create-target-group \
  --name ch-http \
  --protocol HTTP \
  --port 8123 \
  --health-check-path /ping \
  --health-check-interval-seconds 10 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --target-type instance
```

The `/ping` endpoint returns `Ok.` when the node is healthy and ready to serve queries.

## Session Stickiness for Long-Running Queries

If your application uses ClickHouse sessions or temporary tables, enable sticky sessions:

```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --attributes Key=stickiness.enabled,Value=true \
               Key=stickiness.type,Value=lb_cookie \
               Key=stickiness.lb_cookie.duration_seconds,Value=300
```

## NGINX as a Simple HTTP Proxy

For on-premise or lightweight setups:

```text
upstream clickhouse_nodes {
    least_conn;
    server ch-node-01:8123;
    server ch-node-02:8123;
    server ch-node-03:8123;
}

server {
    listen 8123;
    location / {
        proxy_pass http://clickhouse_nodes;
        proxy_read_timeout 300s;
    }
}
```

## Rolling Upgrade with Load Balancer

1. Drain traffic from node 1: remove it from the target group.
2. Upgrade ClickHouse on node 1.
3. Re-add node 1 and verify health checks pass.
4. Repeat for remaining nodes.

## Monitoring

Track load balancer target health and 5xx error rates in [OneUptime](https://oneuptime.com). Alert when any target enters an unhealthy state or when HTTP error rate exceeds 1%.

## Summary

Use an L4 load balancer for the native ClickHouse protocol and an ALB or NGINX for the HTTP interface. Health checks on `/ping` ensure traffic only routes to healthy nodes, enabling zero-downtime upgrades.
