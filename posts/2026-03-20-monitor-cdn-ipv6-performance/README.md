# How to Monitor CDN IPv6 Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, CDN, Performance, Monitoring, Prometheus, Grafana

Description: A guide to monitoring CDN performance for IPv6 clients, including latency measurements, cache hit rates, error rates, and comparison with IPv4 performance.

## Key IPv6 CDN Performance Metrics

| Metric | Description | Target |
|---|---|---|
| TTFB (IPv6) | Time to first byte over IPv6 | < 200ms |
| Connection time (IPv6) | TCP handshake duration | < 100ms |
| DNS resolution (AAAA) | Time to resolve AAAA record | < 50ms |
| Cache hit rate (IPv6) | % of IPv6 requests served from cache | > 85% |
| Error rate (IPv6) | % of IPv6 requests with 4xx/5xx | < 1% |
| IPv6 traffic share | % of total traffic over IPv6 | Varies by region |

## Prometheus Blackbox Exporter for IPv6

```yaml
# /etc/prometheus/blackbox.yml

modules:
  http_ipv6_2xx:
    prober: http
    timeout: 10s
    http:
      preferred_ip_protocol: ip6
      ip_protocol_fallback: false
      valid_status_codes: [200, 301, 302]
      fail_if_not_ssl: true

  http_ipv4_2xx:
    prober: http
    timeout: 10s
    http:
      preferred_ip_protocol: ip4
      ip_protocol_fallback: false
      valid_status_codes: [200, 301, 302]
      fail_if_not_ssl: true
```

```yaml
# prometheus.yml scrape config

scrape_configs:
  - job_name: cdn_ipv6
    metrics_path: /probe
    params:
      module: [http_ipv6_2xx]
    static_configs:
      - targets:
          - https://cdn.example.com/
          - https://cdn.example.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
      - target_label: ip_version
        replacement: ipv6
```

## Grafana Dashboard Queries

```promql
# IPv6 probe success rate
probe_success{job="cdn_ipv6"}

# IPv6 vs IPv4 latency comparison
probe_http_duration_seconds{job="cdn_ipv6", phase="connect"}
probe_http_duration_seconds{job="cdn_ipv4", phase="connect"}

# IPv6 TTFB
probe_http_duration_seconds{job="cdn_ipv6", phase="processing"}

# DNS resolution time for AAAA records
probe_dns_lookup_time_seconds{job="cdn_ipv6"}

# IPv6 probe success over time
avg_over_time(probe_success{job="cdn_ipv6"}[1h])
```

## Cloudflare Analytics API

```bash
# Get IPv6 traffic metrics from Cloudflare
curl "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/analytics/dashboard?since=-10080&until=0" \
  -H "Authorization: Bearer $CF_API_TOKEN" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
# Extract IPv6-specific metrics if available
print(json.dumps(data['result']['totals'], indent=2))
"
```

## Custom IPv6 Performance Monitoring Script

```bash
#!/bin/bash
# monitor-cdn-ipv6.sh - Collect IPv6 CDN performance metrics

CDN_ENDPOINTS=(
  "https://cdn.example.com/"
  "https://cdn.example.com/static/main.css"
  "https://cdn.example.com/api/health"
)

collect_metrics() {
  local url=$1
  local result=$(curl -6 -s -o /dev/null \
    -w "%{time_namelookup} %{time_connect} %{time_starttransfer} %{time_total} %{http_code}" \
    "$url" 2>/dev/null)

  local dns=$(echo $result | awk '{print $1}')
  local connect=$(echo $result | awk '{print $2}')
  local ttfb=$(echo $result | awk '{print $3}')
  local total=$(echo $result | awk '{print $4}')
  local code=$(echo $result | awk '{print $5}')

  echo "url=\"$url\" dns=${dns}s connect=${connect}s ttfb=${ttfb}s total=${total}s status=$code"
}

echo "=== CDN IPv6 Performance Report ==="
echo "Timestamp: $(date -u)"
echo ""

for endpoint in "${CDN_ENDPOINTS[@]}"; do
  collect_metrics "$endpoint"
done

echo ""
echo "=== IPv4 vs IPv6 Comparison (main page) ==="
echo -n "IPv4: "
curl -4 -s -o /dev/null -w "TTFB=%{time_starttransfer}s total=%{time_total}s\n" \
  "${CDN_ENDPOINTS[0]}"
echo -n "IPv6: "
curl -6 -s -o /dev/null -w "TTFB=%{time_starttransfer}s total=%{time_total}s\n" \
  "${CDN_ENDPOINTS[0]}"
```

## Real User Monitoring (RUM) for IPv6

```html
<!-- Add RUM to your web pages to measure real IPv6 performance -->
<script>
  // Detect if user connected via IPv6
  fetch('https://api.ipify.org?format=json')
    .then(r => r.json())
    .then(data => {
      const isIPv6 = data.ip.includes(':');
      const navEntry = performance.getEntriesByType('navigation')[0];

      // Report metrics
      const metrics = {
        ipVersion: isIPv6 ? 'ipv6' : 'ipv4',
        dns: navEntry.domainLookupEnd - navEntry.domainLookupStart,
        connect: navEntry.connectEnd - navEntry.connectStart,
        ttfb: navEntry.responseStart - navEntry.requestStart,
        pageLoad: navEntry.loadEventEnd - navEntry.startTime
      };

      // Send to analytics
      navigator.sendBeacon('/api/metrics', JSON.stringify(metrics));
    });
</script>
```

## Alerting on IPv6 CDN Degradation

```yaml
groups:
- name: cdn-ipv6
  rules:
  - alert: CDNIPv6Down
    expr: probe_success{job="cdn_ipv6"} == 0
    for: 2m
    annotations:
      summary: "CDN IPv6 probe failed for {{ $labels.instance }}"

  - alert: CDNIPv6HighLatency
    expr: probe_http_duration_seconds{job="cdn_ipv6", phase="processing"} > 2
    for: 5m
    annotations:
      summary: "CDN IPv6 TTFB > 2 seconds for {{ $labels.instance }}"

  - alert: CDNIPv6SlowerThanIPv4
    expr: |
      probe_http_duration_seconds{job="cdn_ipv6", phase="connect"} >
      probe_http_duration_seconds{job="cdn_ipv4", phase="connect"} * 2
    for: 10m
    annotations:
      summary: "IPv6 CDN connection 2x slower than IPv4"
```

Monitoring CDN IPv6 performance separately from IPv4 helps identify cases where the IPv6 path has degraded quality, enabling targeted troubleshooting before IPv6 users notice performance issues.
