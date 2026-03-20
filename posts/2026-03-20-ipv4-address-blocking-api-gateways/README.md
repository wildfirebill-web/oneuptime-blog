# How to Implement IPv4 Address Blocking in API Gateways

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, API Gateway, Security, Kong, AWS API Gateway, Nginx, Rate Limiting

Description: Block specific IPv4 addresses and CIDR ranges at the API gateway layer using Kong, AWS API Gateway resource policies, and Nginx, to protect backend services from malicious traffic.

## Introduction

Blocking IPs at the API gateway layer stops malicious traffic before it reaches your application code. Most gateways support static IP lists, CIDR ranges, and dynamic blocking rules.

## Kong Gateway - IP Restriction Plugin

```bash
# Apply to a service

curl -X POST http://localhost:8001/services/my-service/plugins \
  --data "name=ip-restriction" \
  --data "config.deny[]=192.168.50.20" \
  --data "config.deny[]=10.10.0.0/16"

# Apply globally
curl -X POST http://localhost:8001/plugins \
  --data "name=ip-restriction" \
  --data "config.deny[]=203.0.113.45"
```

```yaml
# declarative (deck/KongIngress)
plugins:
  - name: ip-restriction
    config:
      deny:
        - 203.0.113.0/24
        - 198.51.100.5
      allow:
        - 10.0.0.0/8
```

## AWS API Gateway - Resource Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:us-east-1:123456789012:abc123/*/*/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": ["203.0.113.0/24", "198.51.100.5/32"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:us-east-1:123456789012:abc123/*/*/*"
    }
  ]
}
```

## Nginx - geo Module

```nginx
http {
    geo $blocked_ip {
        default         0;
        203.0.113.0/24  1;
        198.51.100.5    1;
        10.10.0.0/16    1;
    }

    server {
        listen 443 ssl;
        location /api/ {
            if ($blocked_ip) {
                return 403 "Forbidden";
            }
            proxy_pass http://backend;
        }
    }
}
```

## Dynamic Blocking with Redis (Node.js + Express)

```javascript
const express = require('express');
const redis = require('redis');

const app = express();
const client = redis.createClient();
await client.connect();

async function ipBlockMiddleware(req, res, next) {
    const ip = req.ip.replace('::ffff:', '');
    const blocked = await client.get(`blocked:${ip}`);
    if (blocked) return res.status(403).json({ error: 'Access denied' });
    next();
}

// Block an IP with optional TTL (seconds)
async function blockIP(ip, ttl = 3600) {
    if (ttl > 0) await client.setEx(`blocked:${ip}`, ttl, '1');
    else await client.set(`blocked:${ip}`, '1');
}

app.use(ipBlockMiddleware);
app.get('/api/data', (req, res) => res.json({ ok: true }));
```

## Terraform - AWS WAF IP Set

```hcl
resource "aws_wafv2_ip_set" "blocked_ips" {
  name               = "blocked-ipv4-set"
  scope              = "REGIONAL"
  ip_address_version = "IPV4"

  addresses = [
    "203.0.113.0/24",
    "198.51.100.5/32",
  ]
}

resource "aws_wafv2_web_acl_association" "api_gw" {
  resource_arn = aws_api_gateway_stage.prod.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

## Conclusion

Block IPv4 addresses at the gateway layer for zero-overhead enforcement - the request never reaches your application. Use static lists for known bad actors, CIDR ranges for subnets, and dynamic Redis-backed lists for automated threat response.
