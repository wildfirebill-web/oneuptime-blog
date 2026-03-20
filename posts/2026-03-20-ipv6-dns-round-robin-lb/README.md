# How to Configure IPv6 Load Balancing with DNS Round-Robin

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, Round-Robin, Load Balancing, AAAA, High Availability

Description: A guide to implementing IPv6 load balancing using DNS round-robin with AAAA records, including TTL tuning, health checking, and limitations.

DNS round-robin load balancing distributes traffic across multiple IPv6 servers by returning different AAAA records in rotation. It's the simplest form of load balancing, requiring no dedicated load balancer hardware or software — just DNS configuration.

## Basic DNS Round-Robin for IPv6

Configure multiple AAAA records for the same hostname:

```bash
# BIND zone file configuration
$TTL 60    # Short TTL for fast failover (60 seconds)

api.example.com.    IN    AAAA    2001:db8::server1
api.example.com.    IN    AAAA    2001:db8::server2
api.example.com.    IN    AAAA    2001:db8::server3
```

DNS servers return all three records, rotating the order on each query response.

## Configuration in Different DNS Systems

### BIND (named.conf + zone file)

```
; zone/example.com
$TTL 60

@    IN    SOA    ns1.example.com. admin.example.com. (
                  2026031901 ; serial
                  3600 ; refresh
                  1800 ; retry
                  604800 ; expire
                  300 ) ; minimum TTL

; IPv6 round-robin
api    60    IN    AAAA    2001:db8::server1
api    60    IN    AAAA    2001:db8::server2
api    60    IN    AAAA    2001:db8::server3
```

### Cloudflare DNS (Terraform)

```hcl
# Multiple AAAA records for round-robin
resource "cloudflare_record" "api_v6_1" {
  zone_id = var.zone_id
  name    = "api"
  type    = "AAAA"
  value   = "2001:db8::server1"
  ttl     = 60
}

resource "cloudflare_record" "api_v6_2" {
  zone_id = var.zone_id
  name    = "api"
  type    = "AAAA"
  value   = "2001:db8::server2"
  ttl     = 60
}

resource "cloudflare_record" "api_v6_3" {
  zone_id = var.zone_id
  name    = "api"
  type    = "AAAA"
  value   = "2001:db8::server3"
  ttl     = 60
}
```

### AWS Route 53 (Multiple Value Answer Routing)

```hcl
resource "aws_route53_record" "api_v6" {
  zone_id = aws_route53_zone.main.id
  name    = "api.example.com"
  type    = "AAAA"
  ttl     = 60

  records = [
    "2001:db8::server1",
    "2001:db8::server2",
    "2001:db8::server3"
  ]
}

# Or use Route 53 Weighted Routing for controlled distribution
resource "aws_route53_record" "api_v6_1" {
  zone_id = aws_route53_zone.main.id
  name    = "api.example.com"
  type    = "AAAA"
  ttl     = 60
  records = ["2001:db8::server1"]

  weighted_routing_policy {
    weight = 50    # 50% of traffic
  }

  set_identifier = "server1"
}
```

## Health Checking with DNS Round-Robin

Pure DNS round-robin doesn't remove failed servers. Use DNS health checking:

```bash
# Using Route 53 health checks
resource "aws_route53_health_check" "server1" {
  ip_address        = "2001:db8::server1"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_record" "api_v6_1" {
  # Only return this record if health check passes
  health_check_id = aws_route53_health_check.server1.id
  ...
}
```

## TTL Strategy

```
Short TTL (30-60s):
  Pros: Fast failover if a server goes down
  Cons: Higher DNS query load, more cache misses

Long TTL (300-3600s):
  Pros: Fewer DNS queries, more efficient
  Cons: Slow failover, clients cache old addresses

Recommended:
  Normal: 60 seconds (balance of speed and efficiency)
  Emergency/active incident: 30 seconds
```

## Testing DNS Round-Robin

```bash
# Query multiple times to see different order
for i in {1..5}; do
  dig +short AAAA api.example.com
  echo "---"
done

# Verify all addresses are returned
dig AAAA api.example.com

# Test each address directly
for addr in 2001:db8::server1 2001:db8::server2 2001:db8::server3; do
  echo -n "Testing $addr: "
  curl -6 --connect-to "::[$addr]" https://api.example.com/health 2>&1 | tail -1
done
```

## Limitations of DNS Round-Robin

| Limitation | Impact |
|---|---|
| No active health checking | Failed servers stay in rotation until removed manually |
| Client-side caching | TTL may not be respected, distribution uneven |
| No session persistence | Same client may go to different server each request |
| Uneven distribution | Some resolvers don't rotate properly |

For production use, DNS round-robin works best as a secondary distribution mechanism combined with a proper load balancer for health checking and session management.
