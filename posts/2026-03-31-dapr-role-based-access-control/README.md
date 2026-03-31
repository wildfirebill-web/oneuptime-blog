# How to Implement Role-Based Access Control with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, RBAC, Authorization, Security, Access Control

Description: Implement role-based access control for Dapr service invocation using the built-in access control policies and mTLS identity verification.

---

## RBAC with Dapr Access Control

Dapr's access control policies restrict which services can invoke which methods on other services. Policies are defined in the Dapr Configuration resource and enforced by the sidecar based on SPIFFE identities established via mTLS.

## Enabling mTLS (Required for RBAC)

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
  namespace: default
spec:
  mtls:
    enabled: true
    workloadCertTTL: "24h"
    allowedClockSkew: "15m"
```

## Defining Access Control Policies

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: payment-service-config
  namespace: default
spec:
  accessControl:
    defaultAction: deny
    trustDomain: "cluster.local"
    policies:
    - appId: "order-service"
      defaultAction: deny
      namespace: "default"
      operations:
      - name: "/v1/payments"
        httpVerb: ["POST"]
        action: allow
      - name: "/v1/payments/*/refund"
        httpVerb: ["POST"]
        action: allow

    - appId: "admin-service"
      defaultAction: allow
      namespace: "default"

    - appId: "reporting-service"
      defaultAction: deny
      namespace: "default"
      operations:
      - name: "/v1/payments"
        httpVerb: ["GET"]
        action: allow
```

## Attaching the Config to Your Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "payment-service"
        dapr.io/config: "payment-service-config"
```

## Application-Level Role Checking

For finer-grained RBAC within your service, check roles from JWT claims:

```python
from functools import wraps
from fastapi import Request, HTTPException

ROLE_PERMISSIONS = {
    "admin": ["read", "write", "delete"],
    "manager": ["read", "write"],
    "viewer": ["read"],
}

def require_permission(permission: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(request: Request, *args, **kwargs):
            # Dapr forwards SPIFFE identity in X-Forwarded-Client-Cert
            # App-level roles come from JWT claims
            roles = request.headers.get("X-JWT-Roles", "viewer").split(",")
            allowed = any(
                permission in ROLE_PERMISSIONS.get(role.strip(), [])
                for role in roles
            )
            if not allowed:
                raise HTTPException(
                    status_code=403,
                    detail=f"Permission '{permission}' required"
                )
            return await func(request, *args, **kwargs)
        return wrapper
    return decorator


@app.delete("/v1/payments/{payment_id}")
@require_permission("delete")
async def delete_payment(request: Request, payment_id: str):
    await payment_service.delete(payment_id)
    return {"deleted": payment_id}
```

## Testing RBAC Policies

```bash
# order-service CAN invoke POST /v1/payments - allowed
curl -H "dapr-app-id: payment-service" \
  http://localhost:3500/v1.0/invoke/payment-service/method/v1/payments \
  -X POST -d '{"amount":100}'

# reporting-service CANNOT invoke POST /v1/payments - denied by policy
# Dapr sidecar returns 403 before the request reaches the payment service
```

## Summary

Dapr's access control policies provide network-level RBAC based on verified SPIFFE identities, meaning you don't have to trust the `app-id` header alone. Default-deny policies are the safest starting point: define only the operations each service truly needs. For application-level role checking, JWT claim forwarding from Dapr middleware complements the service-level policies.
