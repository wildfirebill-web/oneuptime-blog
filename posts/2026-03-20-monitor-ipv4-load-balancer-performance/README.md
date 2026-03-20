# How to Monitor IPv4 Load Balancer Performance and Health

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Load Balancing, Monitoring, IPv4, HAProxy, Nginx, AWS, Metrics

Description: Monitor IPv4 load balancer performance metrics including connection rates, latency, backend health, and error rates using HAProxy stats, Nginx status, and cloud provider metrics.

## Introduction

Monitoring a load balancer requires tracking both the load balancer itself (connections, errors, latency) and the health of individual backends. Different load balancers expose metrics differently - HAProxy has a stats page and socket, Nginx has a status page, and cloud LBs use CloudWatch/Azure Monitor/Cloud Monitoring.

## HAProxy Monitoring

### Enable the Stats Page

```text
frontend stats
    bind 0.0.0.0:8404
    stats enable
    stats uri /haproxy?stats
    stats realm "HAProxy Stats"
    stats auth admin:StrongPassword
    stats refresh 5s
    stats show-legends
    stats show-node
```

### Monitor via the Admin Socket

```bash
# Show all server states

echo "show stat" | sudo socat stdio /run/haproxy/admin.sock | \
  awk -F',' '{print $1,$2,$18,$19,$47}' | \
  column -t

# Key fields:
# $18 = status (UP/DOWN/DRAIN)
# $19 = act (active sessions)
# $47 = hrsp_5xx (5xx error count)

# Current connection rate per second
echo "show info" | sudo socat stdio /run/haproxy/admin.sock | grep ConnRate
```

### HAProxy Prometheus Exporter

```text
frontend stats
    bind 0.0.0.0:8404
    http-request use-service prometheus-exporter if { path /metrics }
    stats enable
    stats uri /stats
```

```bash
# Scrape HAProxy metrics
curl http://localhost:8404/metrics | grep haproxy_backend
```

## Nginx Monitoring

### Enable stub_status

```nginx
server {
    listen 8080;
    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
curl http://localhost:8080/nginx_status
# Output:
# Active connections: 45
# server accepts handled requests
#  1234 1234 5678
# Reading: 2 Writing: 4 Waiting: 39
```

### Nginx with Prometheus (nginx-prometheus-exporter)

```bash
# Install nginx-prometheus-exporter
docker run -d -p 9113:9113 \
  nginx/nginx-prometheus-exporter:latest \
  -nginx.scrape-uri=http://nginx-host:8080/nginx_status
```

## AWS ALB Metrics (CloudWatch)

```bash
# Get ALB request count over last hour
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=app/my-alb/abc123 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum

# Check unhealthy host count
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name UnHealthyHostCount \
  --dimensions Name=LoadBalancer,Value=app/my-alb/abc123 \
               Name=TargetGroup,Value=targetgroup/web-tg/xyz \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average
```

## Key Metrics to Monitor

| Metric | Threshold Alert | Tool |
|---|---|---|
| Active connections | >80% of max | HAProxy info, Nginx status |
| Backend error rate (5xx) | >1% | HAProxy stats, CloudWatch |
| Backend response time | >500ms P99 | ALB TargetResponseTime |
| Unhealthy backends | >0 | All LBs |
| Connection queue depth | >0 | HAProxy queuecur |
| TLS handshake failures | Any increase | Cert expiry check |

## GCP Load Balancer Monitoring

```bash
# Query LB request count via gcloud
gcloud monitoring read \
  'metric.type="loadbalancing.googleapis.com/https/request_count"' \
  --project=my-project \
  --freshness=1h
```

## Setting Up Alerting

```bash
# AWS CloudWatch alarm for high error rate
aws cloudwatch put-metric-alarm \
  --alarm-name alb-high-5xx \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --statistic Sum \
  --period 60 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=LoadBalancer,Value=app/my-alb/abc123 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:ops-alerts
```

## Conclusion

Monitor HAProxy via its stats page or Prometheus exporter. Check Nginx with `stub_status` or the Prometheus exporter. For cloud LBs, use CloudWatch (AWS), Azure Monitor (Azure), or Cloud Monitoring (GCP). Alert on unhealthy backends, high error rates, and connection queue buildup. Track P95/P99 backend response times to detect slow servers before they impact users.
