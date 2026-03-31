# How to Configure ClickHouse Ingress on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kubernetes, Ingress, Networking, TLS

Description: Configure Kubernetes Ingress to expose ClickHouse HTTP and native TCP interfaces with TLS termination, authentication, and access control.

---

ClickHouse exposes two primary interfaces: the HTTP interface (port 8123) and the native TCP interface (port 9000). Exposing these through Kubernetes Ingress requires different approaches depending on which protocol your clients use.

## HTTP Interface via Ingress

The HTTP interface works naturally with standard Kubernetes Ingress controllers. Here is a basic configuration using nginx-ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: clickhouse-http
  namespace: clickhouse
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "500m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - clickhouse.example.com
      secretName: clickhouse-tls
  rules:
    - host: clickhouse.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: clickhouse
                port:
                  number: 8123
```

## TLS Certificate with cert-manager

Use cert-manager to automatically provision and renew TLS certificates:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: clickhouse-tls
  namespace: clickhouse
spec:
  secretName: clickhouse-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - clickhouse.example.com
```

## Adding Basic Authentication

Protect the ClickHouse HTTP endpoint with basic auth at the Ingress layer:

```bash
htpasswd -c auth clickhouse-admin
kubectl create secret generic clickhouse-basic-auth \
  --from-file=auth \
  -n clickhouse
```

Then annotate the Ingress:

```yaml
annotations:
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: clickhouse-basic-auth
  nginx.ingress.kubernetes.io/auth-realm: "ClickHouse Authentication Required"
```

Note: ClickHouse also has its own user authentication, so this adds a second layer.

## Native TCP Protocol

The native TCP protocol (port 9000) cannot be routed through a standard HTTP Ingress. Use a LoadBalancer service instead, or use NGINX TCP pass-through:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clickhouse-native
  namespace: clickhouse
spec:
  type: LoadBalancer
  ports:
    - name: native
      port: 9000
      targetPort: 9000
  selector:
    app: clickhouse
```

For nginx-ingress TCP pass-through, add to the controller's ConfigMap:

```yaml
data:
  "9000": "clickhouse/clickhouse:9000"
```

## IP Allowlisting

Restrict access to known CIDR ranges:

```yaml
annotations:
  nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.1.0/24"
```

## Rate Limiting

Protect against query floods:

```yaml
annotations:
  nginx.ingress.kubernetes.io/limit-rps: "100"
  nginx.ingress.kubernetes.io/limit-connections: "20"
```

## Summary

Exposing ClickHouse on Kubernetes via Ingress works well for the HTTP interface using standard Ingress resources with TLS termination and optional basic auth. For the native TCP protocol, use LoadBalancer services or NGINX TCP pass-through. Always combine Ingress-level access control with ClickHouse's own user authentication for defense in depth.
