# How to Implement IP Whitelisting with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Security, IP Whitelisting, Middleware, Network

Description: Learn how to implement IP whitelisting with Dapr middleware to restrict service access to specific IP addresses or CIDR ranges at the sidecar level.

---

IP whitelisting restricts service access to requests originating from known, trusted IP addresses. Dapr provides a `middleware.http.routerchecker` component and you can also implement IP filtering through Dapr's allow/deny list middleware, protecting services without touching application code.

## Using the Allow List Middleware

Dapr provides `middleware.http.sentinel` for more advanced traffic control, but the most practical approach for IP filtering is using Dapr's middleware combined with network policies.

## IP Allow List Middleware Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: ip-allowlist
  namespace: default
spec:
  type: middleware.http.ipAllowlist
  version: v1
  metadata:
  - name: allowedRanges
    value: "10.0.0.0/8,192.168.1.0/24,172.16.0.0/12"
```

## Apply via Dapr Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: ip-filtered-config
  namespace: default
spec:
  httpPipeline:
    handlers:
    - name: ip-allowlist
      type: middleware.http.ipAllowlist
```

## Annotate Protected Services

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: internal-api
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "internal-api"
        dapr.io/app-port: "8080"
        dapr.io/config: "ip-filtered-config"
```

## Complement with Kubernetes Network Policies

IP whitelisting at the application level should be combined with network policies for defense-in-depth:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-trusted-ips
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: internal-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: "10.0.0.0/8"
    - ipBlock:
        cidr: "192.168.1.0/24"
    ports:
    - protocol: TCP
      port: 3500
```

## Application-Level IP Filtering as Fallback

For cases where middleware does not support IP filtering natively, implement it in application code:

```python
from flask import Flask, request, jsonify, abort
import ipaddress

app = Flask(__name__)

ALLOWED_CIDRS = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("192.168.1.0/24"),
    ipaddress.ip_network("172.16.0.0/12"),
]

def is_allowed(ip: str) -> bool:
    try:
        client_ip = ipaddress.ip_address(ip)
        return any(client_ip in network for network in ALLOWED_CIDRS)
    except ValueError:
        return False

@app.before_request
def check_ip():
    # Use X-Forwarded-For if behind a proxy
    client_ip = request.headers.get('X-Forwarded-For', request.remote_addr)
    if client_ip:
        client_ip = client_ip.split(',')[0].strip()
    if not is_allowed(client_ip):
        abort(403, description=f"IP {client_ip} is not in the allowed list")

@app.route('/internal/data', methods=['GET'])
def get_internal_data():
    return jsonify({"data": "sensitive internal data"})
```

## Testing IP Whitelisting

```bash
# Test from an allowed IP
curl -H "X-Forwarded-For: 10.0.1.100" \
  http://localhost:3500/v1.0/invoke/internal-api/method/internal/data

# Test from a denied IP (should return 403)
curl -H "X-Forwarded-For: 203.0.113.1" \
  http://localhost:3500/v1.0/invoke/internal-api/method/internal/data
```

## Summary

IP whitelisting with Dapr can be implemented at multiple levels: via Dapr middleware components, Kubernetes NetworkPolicies, and application-level CIDR checks. Use all three layers for comprehensive defense-in-depth. Always combine IP whitelisting with authentication and authorization rather than relying on it as the sole access control mechanism.
